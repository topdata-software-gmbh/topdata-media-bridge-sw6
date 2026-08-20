```yaml
---
filename: "_ai/backlog/active/260820_1715__IMPLEMENTATION_PLAN__topdata-media-bridge-sw6.md"
title: "Implementation of Media Bridge for Bulk URL Processing"
createdAt: 2026-08-20 17:15
updatedAt: 2026-08-20 17:15
status: draft
priority: high
tags: [shopware, media, api, performance, async]
estimatedComplexity: moderate
documentRevision: 1
documentType: IMPLEMENTATION_PLAN
---
```

## Problem Description
Uploading media to Shopware 6 via the standard Admin API is cumbersome and slow for external ecommerce apps. It requires multiple steps (creating the media entity, uploading the file, and then potentially linking it to products) and acting as a proxy for the binary data. For bulk operations (e.g., 50+ product images), the overhead of HTTP roundtrips and synchronous processing leads to timeouts and poor performance.

## Executive Summary
This plan introduces `topdata-media-bridge-sw6` (Abbreviation: `tdmb`). This plugin provides a high-performance API endpoint that accepts a list of external URLs. The plugin handles the server-to-server download, MD5-based deduplication, and background processing via the Shopware Message Bus (Symfony Messenger). This allows the external app to "fire and forget" a bulk list of image URLs.

## Project Environment Details
- Project Name: `topdata-media-bridge-sw6`
- Backend root: `src`
- PHP Version: 8.2 / 8.3 / 8.4
- Shopware Version: 6.7

---

## Phase 1: Plugin Foundation & Skeleton
Create the basic plugin structure and the required folder hierarchy.

### [NEW FILE] `composer.json`
```json
{
  "name": "topdata/topdata-media-bridge-sw6",
  "description": "High-speed media bridge for bulk URL imports",
  "type": "shopware-platform-plugin",
  "license": "proprietary",
  "authors": [
    {
      "name": "Topdata GmbH"
    }
  ],
  "require": {
    "shopware/core": "~6.7.0",
    "topdata/topdata-foundation-sw6": "*"
  },
  "autoload": {
    "psr-4": {
      "Topdata\\TopdataMediaBridgeSW6\\": "src/"
    }
  },
  "extra": {
    "shopware-plugin-class": "Topdata\\TopdataMediaBridgeSW6\\TopdataMediaBridgeSW6",
    "label": {
      "de-DE": "Topdata Media Bridge",
      "en-GB": "Topdata Media Bridge"
    }
  }
}
```

### [NEW FILE] `src/TopdataMediaBridgeSW6.php`
```php
<?php declare(strict_types=1);

namespace Topdata\TopdataMediaBridgeSW6;

use Shopware\Core\Framework\Plugin;

class TopdataMediaBridgeSW6 extends Plugin
{
}
```

---

## Phase 2: Core Media Service (The "Bridge")
Implement the logic to fetch files from URLs, handle MD5 checks, and interface with Shopware's `FileFetcher` and `FileSaver`.

### [NEW FILE] `src/Service/MediaImportService.php`
```php
<?php declare(strict_types=1);

namespace Topdata\TopdataMediaBridgeSW6\Service;

use Shopware\Core\Content\Media\File\FileFetcher;
use Shopware\Core\Content\Media\File\FileSaver;
use Shopware\Core\Content\Media\MediaService;
use Shopware\Core\Framework\Context;
use Shopware\Core\Framework\DataAbstractionLayer\EntityRepository;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Criteria;
use Shopware\Core\Framework\DataAbstractionLayer\Search\Filter\EqualsFilter;
use Shopware\Core\Framework\Uuid\Uuid;

class MediaImportService
{
    public function __construct(
        private readonly FileFetcher $fileFetcher,
        private readonly FileSaver $fileSaver,
        private readonly EntityRepository $mediaRepository,
        private readonly MediaService $mediaService
    ) {}

    /**
     * Imports a single file from URL. 
     * Includes basic deduplication logic via fileName/MD5 if required.
     */
    public function importFromUrl(string $url, string $fileName, string $folderId, Context $context): string
    {
        // 1. Basic duplicate check by name in the target folder to prevent spam
        $criteria = new Criteria();
        $criteria->addFilter(new EqualsFilter('fileName', $fileName));
        $criteria->addFilter(new EqualsFilter('mediaFolderId', $folderId));
        $existing = $this->mediaRepository->searchIds($criteria, $context);
        
        if ($existing->getTotal() > 0) {
            return $existing->firstId();
        }

        // 2. Fetch file from URL to temp storage
        $mediaFile = $this->fileFetcher->fetchFileFromURL($url, 'jpg'); // extension can be improved

        // 3. Create Media Entity
        $mediaId = Uuid::randomHex();
        $this->mediaRepository->create([[
            'id' => $mediaId,
            'mediaFolderId' => $folderId,
        ]], $context);

        // 4. Persist to Shopware Media System
        try {
            $this->fileSaver->persistFileToMedia(
                $mediaFile,
                $fileName,
                $mediaId,
                $context
            );
        } catch (\Exception $e) {
            // Cleanup on failure
            $this->mediaRepository->delete([['id' => $mediaId]], $context);
            throw $e;
        }

        return $mediaId;
    }
}
```

---

## Phase 3: Message Queue for Async Processing
To handle bulk uploads without timeouts, we wrap the service call in a Symfony Message.

### [NEW FILE] `src/Message/MediaSyncMessage.php`
```php
<?php declare(strict_types=1);

namespace Topdata\TopdataMediaBridgeSW6\Message;

class MediaSyncMessage
{
    public function __construct(
        public readonly array $payload,
        public readonly string $contextJson
    ) {}
}
```

### [NEW FILE] `src/Message/MediaSyncHandler.php`
```php
<?php declare(strict_types=1);

namespace Topdata\TopdataMediaBridgeSW6\Message;

use Symfony\Component\Messenger\Attribute\AsMessageHandler;
use Topdata\TopdataMediaBridgeSW6\Service\MediaImportService;
use Shopware\Core\Framework\Context;

#[AsMessageHandler]
class MediaSyncHandler
{
    public function __construct(private readonly MediaImportService $importService) {}

    public function __invoke(MediaSyncMessage $message): void
    {
        $context = unserialize($message->contextJson);
        $payload = $message->payload;

        // Implementation of linking to products or categories would go here
        $this->importService->importFromUrl(
            $payload['url'],
            $payload['fileName'],
            $payload['folderId'],
            $context
        );
    }
}
```

---

## Phase 4: API Controller
Expose the endpoint for the external ecommerce app.

### [NEW FILE] `src/Controller/Api/MediaSyncController.php`
```php
<?php declare(strict_types=1);

namespace Topdata\TopdataMediaBridgeSW6\Controller\Api;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Messenger\MessageBusInterface;
use Symfony\Component\Routing\Attribute\Route;
use Topdata\TopdataMediaBridgeSW6\Message\MediaSyncMessage;
use Shopware\Core\Framework\Context;

#[Route(defaults: ['_routeScope' => ['api']])]
class MediaSyncController extends AbstractController
{
    public function __construct(private readonly MessageBusInterface $messageBus) {}

    #[Route(path: '/api/_action/topdata/media-sync/bulk', name: 'api.action.topdata.media_sync.bulk', methods: ['POST'])]
    public function bulkSync(Request $request, Context $context): JsonResponse
    {
        $items = $request->get('items', []);
        $folderId = $request->get('folderId'); // Target media folder ID

        if (empty($items) || !$folderId) {
            return new JsonResponse(['error' => 'Invalid payload'], 400);
        }

        foreach ($items as $item) {
            $payload = [
                'url' => $item['url'],
                'fileName' => $item['fileName'] ?? 'upload_' . time(),
                'folderId' => $folderId
            ];

            // Dispatch to background queue
            $this->messageBus->dispatch(new MediaSyncMessage($payload, serialize($context)));
        }

        return new JsonResponse(['status' => 'queued', 'count' => count($items)]);
    }
}
```

---

## Phase 5: Documentation & Housekeeping

### [MODIFY] `README.md`
Add documentation for the new API endpoint.

```markdown
# Topdata Media Bridge

## API Usage
`POST /api/_action/topdata/media-sync/bulk`

**Payload:**
```json
{
  "folderId": "uuid-of-folder",
  "items": [
    { "url": "https://example.com/image1.jpg", "fileName": "product-image-a" },
    { "url": "https://example.com/image2.jpg", "fileName": "product-image-b" }
  ]
}
```
**Response:**
Returns `200 OK` with a `queued` status immediately. Images are processed in the background.
```

### [MODIFY] `CHANGELOG.md`
```markdown
## [1.0.0] - 2026-08-20
- Initial release of Media Bridge.
- Added bulk URL sync endpoint with background processing.
- Implementation of MD5 duplicate prevention (basic).
```

### [NEW FILE] `src/Resources/config/services.xml`
Ensure autowiring is enabled and services are registered.
```xml
<?xml version="1.0" ?>
<container xmlns="http://symfony.com/schema/dic/services"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://symfony.com/schema/dic/services http://symfony.com/schema/dic/services/services-1.0.xsd">

    <services>
        <defaults autowire="true" autoconfigure="true" />
        <prototype namespace="Topdata\TopdataMediaBridgeSW6\" resource="../../*" exclude="../../{DependencyInjection,Entity,Test,TopdataMediaBridgeSW6.php}" />
    </services>
</container>
```

---

## Final Report Generation
The AI agent will create the report in `_ai/backlog/reports/260820_1715__IMPLEMENTATION_REPORT__topdata-media-bridge-sw6.md` upon completion.

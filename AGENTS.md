# AGENTS.md — topdata-media-bridge-sw6

Shopware 6.7 plugin (`shopware-platform-plugin`) at initial-skeleton stage.
Targets `shopware/core: 6.7.*`, PHP 8.2/8.3/8.4, MIT license.
PSR-4 root: `Topdata\TopdataMediaBridgeSW6\` → `src/`.
Plugin bootstrap class: `Topdata\TopdataMediaBridgeSW6\TopdataMediaBridgeSW6`.

## Current state vs. plan

The active code is only a **scaffold** (`ExampleCommand`, `StorefrontExampleController`,
`AdminApiExampleController`, `example.html.twig`, a placeholder `config.xml`
"example" field, and the bare `Plugin` class).

The **real plugin** is still unimplemented. The authoritative spec is
`_ai/backlog/active/260820_1715__IMPLEMENTATION_PLAN__topdata-media-bridge-sw6.md`.
Read it before writing any code; it defines:

- `Service/MediaImportService.php` — uses `FileFetcher`, `FileSaver`,
  `MediaService`, `mediaRepository`; dedupes by `fileName` + `mediaFolderId`.
- `Message/MediaSyncMessage.php` + `Message/MediaSyncHandler.php` —
  Symfony Messenger async processing (handler uses `serialize($context)`).
- `Controller/Api/MediaSyncController.php` — `POST /api/_action/topdata/media-sync/bulk`,
  `_routeScope: ['api']`, dispatches one message per item.
- `services.xml` — `<defaults autowire="true" autoconfigure="true"/>` +
  `<prototype ... resource="../../*" exclude="../../{DependencyInjection,Entity,Test,TopdataMediaBridgeSW6.php}"/>`.

When implementing, replace the example scaffolding and follow the plan's
file list / namespaces / signatures exactly. The plan ends with a step to
write `_ai/backlog/reports/<id>__IMPLEMENTATION_REPORT__topdata-media-bridge-sw6.md`.

## Directory map

- `src/Command/` — Symfony console commands (`#[AsCommand]`).
- `src/Controller/` — Storefront (`RouteScope`) and Admin API controllers.
  Real implementation will add `Controller/Api/MediaSyncController.php`.
- `src/Service/` — domain services; `MediaImportService` lives here.
- `src/Resources/config/` — `services.xml`, `routes.xml`, `config.xml`,
  `plugin.png`.
- `src/Resources/views/storefront/` — Twig templates, referenced as
  `@TopdataMediaBridgeSW6/storefront/<name>.html.twig`.
- `src/Resources/public/administration/` and `storefront/` are gitignored
  build outputs — **not** to be committed.
- `tests/` — empty placeholder (`.gitkeep` only); no test runner configured.
- `_ai/backlog/active/` — current implementation plan.
- `_ai/backlog/epics/icebox/` — future ideas (empty).
- `_ai/lessons_learned/`, `_ai/technical_decisions/` — empty placeholders.

## Conventions / gotchas

- Namespace **double-prefix**: every PHP namespace under this plugin starts
  with `Topdata\TopdataMediaBridgeSW6\` (vendor + plugin name both start
  with `Topdata`). Don't shorten to `Topdata\MediaBridge\…`.
- `services.xml` currently registers the example controllers manually with
  `setContainer` calls. Once the real plugin lands, **switch to** the
  prototype + autowire/autoconfigure block from the plan — don't keep the
  manual registrations for new services.
- `routes.xml` uses `type="attribute"` (PHP 8 `#[Route]` attributes), not
  YAML route definitions. New routes belong on controller methods.
- Storefront controllers must use `#[RouteScope(scopes: ['storefront'])]`.
  Admin API controllers use `#[Route(defaults: ['_routeScope' => ['api']])]`.
- The plan's `MediaSyncHandler` does `unserialize($message->contextJson)`
  — Shopware contexts are objects, so serialize/unserialize is intentional
  here, not a bug to "fix".
- `_ai/**/*.pdf` is gitignored. Markdown under `_ai/` is tracked and is
  the project's planning memory — update plans/reports there, don't move
  them to the repo root.
- `plugin.png` lives under `src/Resources/config/` (referenced from
  `composer.json` `extra.plugin-icon`), not under `Resources/public/`.

## Working in this repo

- There is **no** `phpunit`, `phpstan`, `eslint`, `vendor/`, or CI workflow
  in the repo. Tests/builds happen inside the host Shopware installation
  (`bin/console plugin:refresh`, `bin/console plugin:install --activate`,
  `vendor/bin/phpunit -c vendor/shopware/platform/phpunit.xml.dist`).
- After creating PHP files, run `bin/console plugin:refresh` in the host
  Shopware so DI caches rebuild and the new services are picked up.
- The CLI example is `topdata:media-bridge:example`; future commands
  should keep the `topdata:media-bridge:` prefix.
- API endpoint convention from the plan: `/api/_action/topdata/...`
  with route name `api.action.topdata.<feature>.<action>`.

## Reference

- Implementation plan: `_ai/backlog/active/260820_1715__IMPLEMENTATION_PLAN__topdata-media-bridge-sw6.md`
- Shopware plugin authoring: <https://developer.shopware.com/docs/guides/plugins/plugins.html>
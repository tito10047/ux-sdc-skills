# UX SDC

This project uses [`tito10047/ux-sdc`](https://github.com/tito10047/ux-sdc) — Single Directory Components for Symfony UX. The `ux-sdc` agent skill is installed; load it before touching component directories, asset injection, or SDC configuration.

## Conventions

- One directory per component, holding `<Name>.php`, `<Name>.html.twig`, `<Name>.css` and optionally `<Name>_controller.js`
- Three top-level buckets: `UI/` (generic), `Layout/` (page furniture), `Page/` (one directory per route)
- **A component lives inside the component that owns it.** A section used only by `Homepage` goes in `Page/Homepage/Hero/`, not in a shared bucket; promote it to `Page/`, `UI/` or `Layout/` only when a second page needs it
- A page's Symfony controller is co-located: `Page/Homepage/HomepageAction.php` + `HomepageAction.html.twig`, generated with `make:sdc-component Page:Homepage --action`
- Generate nested sections with the same command: `make:sdc-component Page:Homepage:Hero`
- Pages fetch data and pass it down; sections stay dumb
- Auto-discovery matches the **exact short class name** — `Alert.css` is found, `alert.css` and `styles.css` are not
- `component_namespace` and `ux_components_dir` are one PSR-4 pair, and `composer.json` must declare the same mapping
- Component tags include every namespace level: `App\Component\Page\Homepage\Hero\Hero` → `<twig:Page:Homepage:Hero:Hero />`
- `#[AsSdcComponent]` extends `#[AsTwigComponent]` — props, slots and hooks behave exactly as in Symfony UX
- **Exactly one component attribute per class.** `#[AsSdcComponent]` needs no `#[AsTwigComponent]`, and `#[AsLiveComponent]` needs no `#[AsSdcComponent]`; stacking two fails the container with "another component already has this name". Use repeatable `#[Asset]` for extra assets on a Live component

## Key rules

- `{{ render_component_assets() }}` must be in `<head>` of the main layout, or no asset is ever injected
- `config/packages/stimulus.yaml` must declare `%kernel.project_dir%/assets/controllers`; the bundle's prepended `controller_paths` otherwise replaces that default and your own controllers stop loading
- `use Stimulus;` requires `implements ComponentNamespaceInterface`, or `{{ controller }}` throws `LogicException`
- Build Stimulus attributes from `{{ controller }}`: `data-action="click->{{ controller }}#toggle"`, `data-{{ controller }}-target="content"`
- Stimulus **outlet** properties collapse `--` to `-` and need bracket access: `this['UI-Calendar-MonthOutlet']`, `this['hasUI-Calendar-MonthOutlet']`
- `<Name>.js` is injected as a `<script>`; `<Name>_controller.js` is a lazily loaded Stimulus controller — never both for the same behaviour
- LiveComponents need `<body {{ stimulus_controller(sdc_loader_controller) }}>` or their CSS never arrives after an AJAX re-render
- Exclude `*/src_component/*.html.twig` in `framework.asset_mapper.excluded_patterns`, or templates are compiled into `public/assets/`
- `--action` routes need `controllers: { resource: routing.controllers }` in `config/routes.yaml` (Symfony 7.4+); otherwise add a routing resource for `src_component/` or the page 404s
- The generated action template extends `layout.html.twig` with a `content` block, and hardcodes `#[Route('/en', ...)]` — fix both
- Adding a CSS file in prod requires `cache:clear` — metadata is resolved at container build time
- `#[Asset]` without a `path` is a no-op; it does not enable auto-discovery
- Use high `priority` for resets and design tokens; everything defaults to `0`

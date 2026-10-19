# UX SDC

This project uses [`tito10047/ux-sdc`](https://github.com/tito10047/ux-sdc) — Single Directory Components for Symfony UX. The `ux-sdc` agent skill is installed; load it before touching component directories, asset injection, or SDC configuration.

## Conventions

- One directory per component, holding `<Name>.php`, `<Name>.html.twig`, `<Name>.css` and optionally `<Name>_controller.js`
- Auto-discovery matches the **exact short class name** — `Alert.css` is found, `alert.css` and `styles.css` are not
- `component_namespace` and `ux_components_dir` are one PSR-4 pair, and `composer.json` must declare the same mapping
- Component tags include every namespace level: `App\Component\UI\Alert\Alert` → `<twig:UI:Alert:Alert />`
- `#[AsSdcComponent]` extends `#[AsTwigComponent]` — props, slots and hooks behave exactly as in Symfony UX

## Key rules

- `{{ render_component_assets() }}` must be in `<head>` of the main layout, or no asset is ever injected
- `config/packages/stimulus.yaml` must declare `%kernel.project_dir%/assets/controllers`; the bundle's prepended `controller_paths` otherwise replaces that default and your own controllers stop loading
- `use Stimulus;` requires `implements ComponentNamespaceInterface`, or `{{ controller }}` throws `LogicException`
- Build Stimulus attributes from `{{ controller }}`: `data-action="click->{{ controller }}#toggle"`, `data-{{ controller }}-target="content"`
- Stimulus **outlet** properties collapse `--` to `-` and need bracket access: `this['UI-Calendar-MonthOutlet']`, `this['hasUI-Calendar-MonthOutlet']`
- `<Name>.js` is injected as a `<script>`; `<Name>_controller.js` is a lazily loaded Stimulus controller — never both for the same behaviour
- LiveComponents need `<body {{ stimulus_controller(sdc_loader_controller) }}>` or their CSS never arrives after an AJAX re-render
- Exclude `*/src_component/*.html.twig` in `framework.asset_mapper.excluded_patterns`, or templates are compiled into `public/assets/`
- Adding a CSS file in prod requires `cache:clear` — metadata is resolved at container build time
- `#[Asset]` without a `path` is a no-op; it does not enable auto-discovery
- Use high `priority` for resets and design tokens; everything defaults to `0`

---
name: ux-sdc
description: 'tito10047/ux-sdc -- Single Directory Components for Symfony UX: one directory per component holding its PHP class, Twig template, CSS and JS, with automatic registration, template mapping and per-render asset injection into <head>. Use when restructuring components so each lives in its own directory, when a component CSS/JS file is not injected, when configuring ux_components_dir / component_namespace, when deriving Stimulus controller names, or when SDC assets must load after a LiveComponent AJAX re-render. Code triggers: AsSdcComponent, Tito10047\UX\Sdc, ux_sdc, render_component_assets(), sdc_loader_controller, make:sdc-component, ComponentNamespaceInterface, X-SDC-Assets-CSS, src_component. Also trigger when the user asks "how do I keep component PHP, Twig and CSS in one folder", "why is my component CSS not loaded", "how do I scaffold an SDC component". Do NOT trigger for plain TwigComponent/LiveComponent questions unrelated to co-locating files or per-component assets, or for Drupal SDC.'
license: MIT
metadata:
  author: Jozef Môstka
  url: https://github.com/tito10047
  version: "0.1.0"
---

# UX SDC

`tito10047/ux-sdc` is a Symfony bundle that implements the **Single Directory Component** pattern on top of Symfony UX. Each component is a directory:

```text
src_component/
└── UI/
    └── Alert/
        ├── Alert.php            # auto-registered component class
        ├── Alert.html.twig      # auto-mapped template
        ├── Alert.css            # auto-injected stylesheet
        └── Alert_controller.js  # auto-registered Stimulus controller
```

Nothing in that directory is wired by hand. The bundle discovers the class, maps the template by file location, and injects `Alert.css` into `<head>` **only on pages where `Alert` was actually rendered**.

## When to Use UX SDC

Use it when component files are scattered across `src/Twig/Components/`, `templates/components/` and `assets/`, and that spread is the problem — you want one folder per component and per-component CSS that ships only when the component renders.

It is an *organisational and asset-delivery* layer, not a rendering layer. Rendering, props, slots, actions and reactivity all stay plain Symfony UX: `#[AsSdcComponent]` extends `#[AsTwigComponent]`, and `#[AsLiveComponent]` components work inside SDC directories unchanged. If the question is about how components render rather than where their files live, it is a TwigComponent / LiveComponent question.

## Installation

```bash
composer require tito10047/ux-sdc
```

Register the bundle (Flex does this automatically):

```php
// config/bundles.php
return [
    // ...
    Tito10047\UX\Sdc\SdcBundle::class => ['all' => true],
];
```

### 1. Map the namespace to the directory

The `component_namespace` ↔ `ux_components_dir` pair must be a **1:1 PSR-4 mapping**, and Composer must know about it too:

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/",
            "App\\Component\\": "src_component/"
        }
    }
}
```

```yaml
# config/packages/ux_sdc.yaml
ux_sdc:
    ux_components_dir: '%kernel.project_dir%/src_component'
    component_namespace: 'App\Component'
```

Run `composer dump-autoload` after editing `composer.json`.

### 2. Add the asset placeholder

```twig
{# templates/base.html.twig #}
<head>
    {# ... #}
    {{ render_component_assets() }}
</head>
```

Without this call nothing is ever injected — see [Gotchas](references/gotchas.md).

### 3. Re-declare your own Stimulus controller path

**Required, and easy to miss.** The bundle prepends `stimulus.controller_paths` with `ux_components_dir`. Because that node's default (`%kernel.project_dir%/assets/controllers`) only applies when *nobody* sets the key, supplying it drops the default — and every controller in `assets/controllers/` silently stops being registered. Create the file so both paths are present:

```yaml
# config/packages/stimulus.yaml
stimulus:
    controller_paths:
        - '%kernel.project_dir%/assets/controllers'
```

### 4. (LiveComponent only) Add the loader controller

```twig
<body {{ stimulus_controller(sdc_loader_controller) }}>
```

### 5. Keep templates out of the public asset map

`ux_components_dir` is added to AssetMapper's paths so that `Alert.css` can be served. That also pulls `Alert.html.twig` into the asset map, and `asset-map:compile` would write it into `public/assets/`. Exclude templates:

```yaml
# config/packages/asset_mapper.yaml
framework:
    asset_mapper:
        excluded_patterns:
            - '*/src_component/*.html.twig'
```

(`.php` files are skipped by AssetMapper itself, so only Twig needs excluding.)

## Quick Reference

```
#[AsSdcComponent]                      Register; infers name, template, CSS, JS
#[AsSdcComponent(css: 'x.css')]        Explicit AssetMapper path, forced type css
#[Asset(path: 'y.css', priority: 10)]  Repeatable; attach any asset explicitly
use Stimulus;                          Adds getController(); needs the interface below
implements ComponentNamespaceInterface Lets the bundle inject the namespace
{{ controller }}                       Stimulus controller name, in Twig
{{ render_component_assets() }}        Placeholder in <head> where assets land
{{ sdc_loader_controller }}            Global: the bundle's loader controller name
make:sdc-component UI:Alert            Scaffold a component directory
```

## Creating a Component

```bash
php bin/console make:sdc-component UI:Alert
```

Creates `<ux_components_dir>/UI/Alert/` with `Alert.php`, `Alert.html.twig`, `Alert.css`, and optionally `Alert_controller.js` (`--stimulus`) and a routed `AlertAction.php` + `AlertAction.html.twig` (`--action`). `:` and `/` both work as separators. In non-interactive mode neither the Stimulus controller nor the Action is generated unless the flag is passed.

By hand:

```php
// src_component/UI/Alert/Alert.php
namespace App\Component\UI\Alert;

use Tito10047\UX\Sdc\Attribute\AsSdcComponent;

#[AsSdcComponent]
class Alert
{
    public string $type = 'info';
    public string $message = '';
}
```

```twig
{# src_component/UI/Alert/Alert.html.twig #}
<div {{ attributes.defaults({ class: 'alert alert-' ~ type }) }}>
    {{ message }}
</div>
```

```css
/* src_component/UI/Alert/Alert.css */
@layer components {
    .alert { padding: .75rem 1rem; border-radius: .25rem; }
}
```

## Naming

The component name is the FQCN with the `component_namespace` prefix stripped and `\` replaced by `:`:

| Class | Tag | Stimulus controller |
|---|---|---|
| `App\Component\UI\Alert\Alert` | `<twig:UI:Alert:Alert />` | `UI--Alert--Alert` |
| `App\Component\Layout\TopBar\TopBar` | `<twig:Layout:TopBar:TopBar />` | `Layout--TopBar--TopBar` |

The repeated leaf (`Alert:Alert`) is the price of one-directory-per-component. Shorten the tag with an explicit name — template and asset discovery are driven by file location, not by the name, so this is safe:

```php
#[AsSdcComponent('Alert')]   // <twig:Alert />
```

## Asset Discovery

Two independent mechanisms feed the same registry.

**Auto-discovery** (on by default) looks next to the class file for exactly `<ShortClassName>.css`, `<ShortClassName>.js` and `<ShortClassName>.html.twig`. The match is exact and case-sensitive: `Alert.css` is found, `alert.css` and `styles.css` are not.

**Explicit declaration** via attributes, for assets that are not co-located — a shared stylesheet, a vendored library, a design-token file:

```php
#[AsSdcComponent(css: 'styles/alert.css', js: 'js/alert.js')]
#[Asset(path: 'vendor/tooltip.css', priority: 10)]
#[Asset(path: 'print.css', type: 'css', attributes: ['media' => 'print'])]
class Alert {}
```

Paths are **AssetMapper logical paths**, resolved through `AssetMapperInterface::getAsset()`. Higher `priority` renders earlier; the default is `0`.

`Alert_controller.js` is deliberately *not* an injected asset. It is picked up as a Stimulus controller because `ux_components_dir` is registered in `stimulus.controller_paths`. That split is the point of the bundle's "no phantom controllers" claim: CSS arrives via the registry, Stimulus behaviour via the Stimulus loader, and neither needs an empty controller file just to `import` a stylesheet.

## Asset Injection

1. A component renders → the active render listener pushes its asset paths into `AssetRegistry` (deduplicated by path + type + attributes).
2. On `kernel.response`, `AssetResponseListener` replaces the placeholder with `<link rel="stylesheet">` / `<script src>` tags, sorted by descending priority.
3. It also adds `Link: <path>; rel=preload; as=style|script` headers so the browser starts fetching before it parses the HTML.

If no component on the page declared assets, the placeholder is replaced with an empty string — it never leaks into the output.

## Stimulus Integration

To get the `controller` variable in Twig, a component needs **both** the trait and the interface:

```php
use Tito10047\UX\Sdc\Twig\ComponentNamespaceInterface;
use Tito10047\UX\Sdc\Twig\Stimulus;

#[AsSdcComponent]
class Alert implements ComponentNamespaceInterface
{
    use Stimulus;
}
```

The trait supplies both interface methods; the interface is what the render listener type-checks before injecting the namespace. Trait without interface throws `LogicException: Component namespace is not set.`

```twig
<div {{ attributes.defaults(stimulus_controller(controller)) }}>
```

Everything else in the template is built from that same `controller` variable:

```twig
<div {{ attributes.defaults(stimulus_controller(controller)) }}>
    <button data-action="click->{{ controller }}#toggle">Toggle</button>

    <div data-{{ controller }}-target="content">...</div>
</div>
```

```twig
{# passing values #}
<div {{ attributes.defaults({
    'data-controller': controller,
    ('data-' ~ controller ~ '-url-value'): path('app.search'),
}) }}>
```

**Outlets need bracket notation.** Stimulus collapses `--` to `-` when it derives property names, and a name like `UI-Calendar-Month` is not a valid JS identifier — so the generated `hasXxxOutlet` / `xxxOutlet` properties are unreachable with dot access. Declare the real controller name, read it with brackets and one dash:

```js
static outlets = ['UI--Calendar--Month'];

if (this['hasUI-Calendar-MonthOutlet']) {
    this['UI-Calendar-MonthOutlet'].refresh();
}
```

Only outlets are affected; targets, values and actions are fine. See [Gotchas](references/gotchas.md) for the rule and for how to avoid it entirely.

`ComponentNamespaceInterface` also works on controllers (`Action` classes), which is how a page-level `HomepageAction` can reach the same controller name as its component.

## Live Components

A LiveComponent in an SDC directory needs no SDC-specific attribute — auto-discovery already sees its files:

```php
namespace App\Component\Search\Search;

use Symfony\UX\LiveComponent\Attribute\AsLiveComponent;
use Symfony\UX\LiveComponent\DefaultActionTrait;
use Tito10047\UX\Sdc\Twig\ComponentNamespaceInterface;
use Tito10047\UX\Sdc\Twig\Stimulus;

#[AsLiveComponent]
class Search implements ComponentNamespaceInterface
{
    use DefaultActionTrait;
    use Stimulus;
}
```

On the **initial render** assets go into `<head>` as usual. On an **AJAX re-render** there is no `<head>` to patch, so `AssetResponseListener` detects the Live request (`Accept: application/vnd.live-component+html`) and answers with headers instead:

```
X-SDC-Assets-CSS: /assets/Search-a1b2c3.css
X-SDC-Assets-JS:  /assets/Search-d4e5f6.js
```

The bundle's `sdc_loader_controller` hooks `live:connect` → `render:started`, reads those headers off the response and appends any missing `<link>` / `<script>` to `<head>`. This is the only piece that must be wired manually:

```twig
<body {{ stimulus_controller(sdc_loader_controller) }}>
```

Without it, a component that first appears through a Live re-render (a modal, a lazily revealed panel) renders unstyled.

## Dev vs Prod

| | dev | prod / test |
|---|---|---|
| Listener | `DevComponentRenderListener` | `ComponentRenderListener` |
| Discovery | `ComponentMetadataResolver` on each render, cached per class per request | pre-built `%kernel.cache_dir%/sdc_metadata.php` |
| New CSS file | picked up immediately | requires `cache:clear` |
| Filesystem I/O | yes | none after the first `require` |

`SdcExtension` registers exactly one of the two listeners based on `kernel.environment`, and `AssetComponentCompilerPass` *deletes* the metadata cache in dev. So `SdcMetadataRegistry` is empty in dev by design — never read it in code that must work in both environments.

## Configuration

```yaml
ux_sdc:
    # Root directory for component discovery
    ux_components_dir: '%kernel.project_dir%/src_component'

    # PHP namespace mapped 1:1 onto ux_components_dir. Null disables
    # auto-registration, the twig_component defaults, and {{ controller }}.
    component_namespace: null

    # Find <ShortClassName>.css / .js / .html.twig next to the class
    auto_discovery: true

    # Register ux_components_dir as a Stimulus controller path
    stimulus:
        enabled: true

    # HTML marker that render_component_assets() emits and the response
    # listener replaces
    placeholder: '<!-- __UX_TWIG_COMPONENT_ASSETS__ -->'
```

Given `component_namespace`, the bundle prepends config for you: Twig paths for `ux_components_dir` (unnamed and as `@component`), the directory in `framework.asset_mapper.paths`, `twig_component.defaults` with `template_directory: ''`, and the directory in `stimulus.controller_paths`. Do not declare the same namespace again in your own `twig_component.yaml`.

## Key Principles

**One directory, one component, exact filenames.** Discovery is a filename match against the short class name. Rename the class and you rename four files; that is the trade for zero configuration.

**Assets are per-render, not per-page-definition.** The registry is filled by components that actually rendered, so a conditional branch that never ran ships no CSS. Equally: a component rendered only inside a Live re-render needs the loader controller, or its CSS never arrives.

**Production does no discovery.** Everything is resolved in `AssetComponentCompilerPass` at container build time. If prod behaves differently from dev, the metadata cache is stale — clear it before debugging anything else.

**`#[AsSdcComponent]` is `#[AsTwigComponent]`.** Props, slots, `PreMount`/`PostMount`, `ExposeInTemplate`, anonymous components — all unchanged. Reach for the TwigComponent and LiveComponent skills for those.

## References

- **Full API** (attributes, services, events, Twig functions, maker options, config): [references/api.md](references/api.md)
- **Patterns** (project structure, page components with co-located actions, Live SDC, shared assets, migration): [references/patterns.md](references/patterns.md)
- **Gotchas** (missing placeholder, filename mismatches, stale prod cache, asset map leakage, Live AJAX): [references/gotchas.md](references/gotchas.md)

## See Also

- **twig-component** — props, slots, attributes, anonymous components. `#[AsSdcComponent]` inherits all of it.
- **live-component** — `LiveProp`, `LiveAction`, `data-model`, forms. SDC only changes where the files live and how assets reach the browser.
- **stimulus** — controller anatomy, targets, values. SDC decides the controller *name*; the controller itself is ordinary Stimulus.

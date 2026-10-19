# UX SDC API Reference

Namespace: `Tito10047\UX\Sdc`. Package: `tito10047/ux-sdc`.

---

## AsSdcComponent Attribute

`Tito10047\UX\Sdc\Attribute\AsSdcComponent` extends `Symfony\UX\TwigComponent\Attribute\AsTwigComponent`, so the service is tagged `twig.component` through the normal UX autoconfiguration and every TwigComponent feature applies.

```php
use Tito10047\UX\Sdc\Attribute\AsSdcComponent;

#[AsSdcComponent(
    name: null,                   // defaults to FQCN minus component_namespace, "\" -> ":"
    template: null,               // defaults to <ShortClassName>.html.twig next to the class
    exposePublicProps: true,      // inherited from AsTwigComponent
    attributesVar: 'attributes',  // inherited from AsTwigComponent
    css: null,                    // AssetMapper logical path, registered with type "css"
    js: null,                     // AssetMapper logical path, registered with type "js"
)]
class Alert {}
```

`TARGET_CLASS`, not repeatable. `css` / `js` are added *in addition to* anything auto-discovery finds.

### Exactly one component attribute per class

A component carries **one** of `#[AsSdcComponent]`, `#[AsTwigComponent]` or `#[AsLiveComponent]` — never two.

`#[AsSdcComponent]` alone is enough to register the component. You do **not** add `#[AsTwigComponent]` next to it: Symfony's `AttributeAutoconfigurationPass::findConfigurators()` walks up the attribute's own class hierarchy, so a subclass of `AsTwigComponent` reuses the parent's autoconfigurator and the service is tagged `twig.component`:

```php
if (class_exists($attributeName) && $parent = get_parent_class($attributeName)) {
    return $configurators[$attributeName] = $this->findConfigurators($configurators, $parent);
}
```

`#[AsLiveComponent]` alone is likewise enough — it also extends `AsTwigComponent`, so it is tagged `twig.component` too, and SDC template mapping and asset auto-discovery apply to it without any SDC-specific attribute.

Verified by dumping the `twig.component`-tagged services of a booted kernel:

```
SdcComponent            attrs=[AsSdcComponent]         twig.component=YES
LiveComponentWithAsset  attrs=[AsLiveComponent,Asset]  twig.component=YES
```

Stacking two of them **breaks the container**, because each one registers the class as a component:

```php
#[AsSdcComponent]   // ❌
#[AsLiveComponent]
class Stacked {}
```

```
LogicException: Failed creating the "...\Stacked" component with the automatic
name "Stacked": another component already has this name.
```

So: pick the attribute by what the component needs to *be*.

| Need | Attribute |
|---|---|
| Static component, explicit `css:` / `js:` paths | `#[AsSdcComponent(css: ..., js: ...)]` |
| Static component, co-located files only | `#[AsSdcComponent]` or plain `#[AsTwigComponent]` |
| Reactive component | `#[AsLiveComponent]` — plus `#[Asset]` for any non-co-located asset |

`#[Asset]` is the one attribute that composes freely: it is repeatable and carries no component registration, so it can sit alongside any of the three.

---

## Asset Attribute

`Tito10047\UX\Sdc\Attribute\Asset` — `TARGET_CLASS`, `IS_REPEATABLE`.

```php
use Tito10047\UX\Sdc\Attribute\Asset;

#[Asset(
    path: null,        // AssetMapper logical path; REQUIRED in practice (see below)
    type: null,        // 'css' | 'js'; inferred from the extension when null
    priority: 0,       // higher renders earlier
    attributes: [],    // extra HTML attributes on the <link> / <script> tag
)]
```

```php
#[Asset(path: 'vendor/flatpickr.css', priority: 20)]
#[Asset(path: 'vendor/flatpickr.js', priority: 20)]
#[Asset(path: 'print.css', type: 'css', attributes: ['media' => 'print'])]
#[Asset(path: 'analytics.js', attributes: ['defer' => 'defer', 'async' => 'async'])]
class DatePicker {}
```

**`#[Asset]` without a `path` is a no-op.** `ComponentMetadataResolver::collectExplicitAssets()` skips any `Asset` whose `path` is null. Auto-discovery does not need the attribute and is not triggered by it — a bare `#[Asset]` on a component simply does nothing.

Attribute values are escaped with `htmlspecialchars(..., ENT_QUOTES)` and rendered as `name="value"`.

---

## Stimulus Trait & ComponentNamespaceInterface

```php
namespace Tito10047\UX\Sdc\Twig;

interface ComponentNamespaceInterface
{
    public function setComponentNamespace(string $namespace): void;
    public function getController(): string;
}
```

`Tito10047\UX\Sdc\Twig\Stimulus` implements both methods and exposes the result to Twig:

```php
#[ExposeInTemplate('controller')]
public function getController(): string
```

Derivation: take `static::class`, remove the `component_namespace` prefix, replace `\` with `--`.

| `component_namespace` | Class | `{{ controller }}` |
|---|---|---|
| `App\Component\` | `App\Component\UI\Alert\Alert` | `UI--Alert--Alert` |
| `App\Component\` | `App\Component\Search` | `Search` |

Both pieces are required: the trait gives the methods, the interface is what `ComponentRenderListener` / `DevComponentRenderListener` check with `instanceof` before calling `setComponentNamespace()`. Trait alone → `LogicException`.

Also honoured on controllers: the render listeners subscribe to `kernel.controller` and inject the namespace into any controller implementing the interface (requires `symfony/ux-live-component` to be installed — the tag is only added when `AsLiveComponent` exists).

---

## Twig

### render_component_assets()

```twig
<head>
    {{ render_component_assets() }}
</head>
```

Emits the configured placeholder string (`is_safe: html`). It renders *nothing useful on its own* — `AssetResponseListener` substitutes the real tags on `kernel.response`. Consequences: the call must land in the final HTML of the **main** request, and anything that post-processes or caches the body before `kernel.response` will see the raw marker.

### sdc_loader_controller

A Twig global holding `SdcBundle::STIMULUS_CONTROLLER` = `tito10047--ux-sdc--sdc-loader`.

```twig
<body {{ stimulus_controller(sdc_loader_controller) }}>
```

---

## Services

| Service | Role |
|---|---|
| `Tito10047\UX\Sdc\Service\AssetRegistry` | Per-request collection of assets to inject |
| `Tito10047\UX\Sdc\Service\ComponentMetadataResolver` | Resolves CSS/JS/Twig for a class (build time + dev runtime) |
| `Tito10047\UX\Sdc\Runtime\SdcMetadataRegistry` | Prod: lazy reader of the generated cache file |
| `Tito10047\UX\Sdc\EventListener\ComponentRenderListener` | Prod/test render listener |
| `Tito10047\UX\Sdc\EventListener\DevComponentRenderListener` | Dev render listener |
| `Tito10047\UX\Sdc\EventListener\AssetResponseListener` | Injects tags / sets Live headers |
| `Tito10047\UX\Sdc\Twig\AssetExtension` | `render_component_assets()`, `sdc_loader_controller` |

### AssetRegistry

```php
public function addAsset(string $path, string $type, int $priority = 0, array $attributes = []): void;

/** @return array<int, array{path: string, type: string, priority: int, attributes: array}> */
public function getSortedAssets(): array;   // descending priority

public function clear(): void;
```

Deduplication key is `path|type` plus a serialisation of `attributes` when non-empty. Re-adding the same asset with a higher priority raises the stored priority; nothing else is merged. The same path with *different* attributes is two entries and renders twice.

### SdcMetadataRegistry

```php
public function getMetadata(string $componentFccn): array|string|null;
```

Keys in the generated cache are FQCNs: `App\Component\UI\Alert\Alert` → asset array, and `App\Component\UI\Alert\Alert_template` → the relative template path (a string). Backed by `%kernel.cache_dir%/sdc_metadata.php`; returns `null` for unknown keys and `[]` for everything when the file is absent (which is always the case in dev).

---

## Events & Lifecycle

| Phase | Hook | What happens |
|---|---|---|
| Container build (non-dev) | `AssetComponentCompilerPass` | Scans `twig.component`-tagged services, resolves metadata, dumps `sdc_metadata.php` |
| Container build (dev) | `AssetComponentCompilerPass` | Deletes `sdc_metadata.php` and returns |
| `kernel.controller` | both render listeners | Inject namespace into `ComponentNamespaceInterface` controllers |
| `PostMountEvent` | both render listeners | Inject namespace into the component; prod also registers assets |
| `PreRenderEvent` | both render listeners | Override the template; register assets |
| `kernel.response` | `AssetResponseListener` | Replace placeholder + add `Link` preload headers, or set `X-SDC-Assets-*` |

`DevComponentRenderListener` skips anonymous components (`$metadata->isAnonymous()`).

---

## Live Component Response Headers

`AssetResponseListener` treats a request as a Live request when either

- the request has `Accept: application/vnd.live-component+html`, or
- the response has `Content-Type: application/vnd.live-component+html`.

In that case it skips HTML rewriting entirely and sets comma-separated, de-duplicated public paths:

```
X-SDC-Assets-CSS: /assets/Search-a1b2c3.css,/assets/Alert-9f8e7d.css
X-SDC-Assets-JS:  /assets/Search-d4e5f6.js
```

The loader controller (`assets/controllers/sdc_loader_controller.js`, exposed to AssetMapper as `@tito10047/ux-sdc/sdc_loader_controller.js`) listens for `live:connect` on `document`, registers a `render:started` hook on the component, reads the headers off the response and appends any missing tag to `<head>`. Existence is tested with `href === path || href.endsWith(path)`, so digested paths dedupe correctly. Injected `<script>` tags get `defer`.

---

## Maker

```bash
php bin/console make:sdc-component [name] [--stimulus] [--action]
```

`name` accepts `:` or `/` as a separator (`UI:Alert`, `UI/Alert`); each segment is turned into a class name. Generated under `<ux_components_dir>/<SubPath>/<Name>/`:

| File | Always | Notes |
|---|---|---|
| `<Name>.php` | yes | `#[AsSdcComponent]`; with `--stimulus` also `implements ComponentNamespaceInterface` + `use Stimulus` |
| `<Name>.html.twig` | yes | `attributes.defaults({class: snake_case})`, plus `stimulus_controller(controller)` with `--stimulus` |
| `<Name>.css` | yes | `@layer components { .snake_case {} }` |
| `<Name>_controller.js` | `--stimulus` | `/* stimulusFetch: 'lazy' */` |
| `<Name>Action.php` | `--action` | `AbstractController` with a `#[Route]` |
| `<Name>Action.html.twig` | `--action` | `{% extends 'layout.html.twig' %}` rendering the component |

Interactive mode asks about Stimulus (default yes) and Action (default no). Non-interactive mode generates neither unless the flag is given.

Requires `symfony/maker-bundle`.

### Nesting

Every segment before the last becomes a directory and a namespace level, to any depth:

| Command | Directory | Class | Tag |
|---|---|---|---|
| `make:sdc-component Alert` | `<dir>/Alert/` | `App\Component\Alert\Alert` | `<twig:Alert:Alert />` |
| `make:sdc-component UI:Button` | `<dir>/UI/Button/` | `App\Component\UI\Button\Button` | `<twig:UI:Button:Button />` |
| `make:sdc-component Page:Homepage:Hero` | `<dir>/Page/Homepage/Hero/` | `App\Component\Page\Homepage\Hero\Hero` | `<twig:Page:Homepage:Hero:Hero />` |

This is what makes page-local component nesting practical — generate a page's private sections directly into the page's own directory.

### --action

Adds two files that turn the component into a routed page:

```php
// <dir>/Page/Homepage/HomepageAction.php
namespace App\Component\Page\Homepage;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\Routing\Attribute\Route;

class HomepageAction extends AbstractController
{
    #[Route('/en', name: 'app.homepage')]
    public function index(): \Symfony\Component\HttpFoundation\Response
    {
        return $this->render('Page/Homepage/HomepageAction.html.twig');
    }
}
```

```twig
{# <dir>/Page/Homepage/HomepageAction.html.twig #}
{% extends 'layout.html.twig' %}

{% block content %}
    <twig:Page:Homepage:Homepage />
{% endblock %}
```

Details that matter:

- The route is hardcoded `#[Route('/en', name: 'app.<lowercased component name>')]`. Always edit it.
- The wrapper extends **`layout.html.twig`** and fills a **`content`** block — not Flex's `base.html.twig` / `body`. Provide `templates/layout.html.twig` or change the two lines.
- The `render()` argument is `<SubPath>/<Name>/<Name>Action.html.twig`, resolvable because `ux_components_dir` is registered as an unnamed Twig path.
- The rendered tag is the component's full name including every path segment, so the wrapper and the component always agree.
- The class name is `<Name>Action`, so it is not itself a component — it carries no `AsTwigComponent` attribute and never appears in the SDC metadata.
- Route registration depends on autoconfiguration, not on a routing resource pointing at the directory. See the routing gotcha.

---

## Configuration Reference

```yaml
ux_sdc:
    ux_components_dir: '%kernel.project_dir%/src_component'   # scalar
    component_namespace: null                                  # scalar, nullable
    auto_discovery: true                                       # boolean
    stimulus:
        enabled: true                                          # canBeDisabled()
    placeholder: '<!-- __UX_TWIG_COMPONENT_ASSETS__ -->'       # scalar
```

### Parameters set by the extension

| Parameter | Value |
|---|---|
| `ux_sdc.ux_components_dir` | as configured |
| `ux_sdc.component_namespace` | normalised with a single trailing `\`, or `null` |
| `ux_sdc.auto_discovery` | as configured |
| `app.ui_components.dir` | convenience alias of `ux_components_dir` |

### Configuration prepended into other bundles

With `component_namespace` set, `SdcExtension::prepend()` contributes:

```yaml
twig:
    paths:
        '<ux_components_dir>': ~            # unnamed path
        '<ux_components_dir>/': component   # @component namespace

framework:
    asset_mapper:
        paths:
            - '<ux_components_dir>'
            - '<bundle>/assets/controllers': '@tito10047/ux-sdc'

twig_component:
    defaults:
        'App\Component\':
            template_directory: ''

stimulus:                     # only when stimulus.enabled and the extension exists
    controller_paths:
        - '<ux_components_dir>'
```

Two consequences of prepending rather than appending:

- **`twig_component.defaults`** — do not redeclare the same namespace yourself; your own value would fight the prepended one.
- **`stimulus.controller_paths`** — because the bundle sets this key, StimulusBundle's default `%kernel.project_dir%/assets/controllers` no longer applies, and your own controllers stop being registered. You must declare that path in `config/packages/stimulus.yaml`; prototyped array nodes append across config sources, so both end up in the list.

---

## Testing

Integration tests boot a kernel with the bundle and read the registry directly:

```php
$metadataRegistry = $container->get(SdcMetadataRegistry::class);
$assets = $metadataRegistry->getMetadata(Alert::class);

$this->assertSame('Alert.css', $assets[0]['path']);
$this->assertSame('css', $assets[0]['type']);
```

`SdcMetadataRegistry` is registered public specifically to make this possible. Use the `test` (not `dev`) environment — in dev the cache file is deleted and the registry is empty.

For Live components, assert on headers rather than HTML:

```php
$response = $testComponent->response();
$this->assertStringContainsString('.css', $response->headers->get('X-SDC-Assets-CSS'));
```

---

## Requirements

| | |
|---|---|
| PHP | >= 8.1 |
| Symfony | 6.4 / 7.x / 8.x |
| `symfony/ux-twig-component` | ^2.13 \| ^3.0 |
| Required | `symfony/asset`, `symfony/asset-mapper`, `symfony/framework-bundle`, `symfony/web-link` |
| Optional | `symfony/ux-live-component`, `symfony/stimulus-bundle`, `symfony/maker-bundle` |

The bundle deliberately supports both Symfony UX 2 and 3; keep that range open when touching its `composer.json`.

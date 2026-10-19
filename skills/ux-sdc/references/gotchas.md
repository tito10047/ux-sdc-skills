# UX SDC Gotchas & Debugging

Common pitfalls and their fixes.

---

## Stimulus Outlet Properties Are Not Valid Identifiers

The headline gotcha. SDC controller names contain `--` for every namespace level (`UI--Calendar--Month`), and Stimulus mangles that when it generates outlet property names.

Stimulus derives them with `namespaceCamelize(name)`:

```js
camelize(value.replace(/--/g, '-').replace(/__/g, '_'))
// camelize only uppercases after - or _ when followed by [a-z0-9]
```

`UI--Calendar--Month` → `UI-Calendar-Month`, and because `C` and `M` are already uppercase, `camelize` leaves the dashes in place. Stimulus then defines `UI-Calendar-MonthOutlet` and `hasUI-Calendar-MonthOutlet` — names that cannot be written with dot access.

```js
// ❌ Syntax error / always undefined
static outlets = ['UI--Calendar--Month'];

if (this.hasUICalendarMonthOutlet) {      // this property does not exist
    this.uiCalendarMonthOutlet.refresh();
}
```

```js
// ✅ Declare the real controller name, read with brackets and ONE dash
static outlets = ['UI--Calendar--Month'];

if (this['hasUI-Calendar-MonthOutlet']) {
    this['UI-Calendar-MonthOutlet'].refresh();
}

// Plural / element forms follow the same rule
this['UI-Calendar-MonthOutletElements'].forEach((el) => el.hidden = true);
```

The rule: **the `static outlets` entry and the HTML attribute keep `--`; the property name uses a single `-`.**

```twig
{# the attribute side is unaffected #}
<div {{ attributes.defaults({
    'data-controller': controller,
    ('data-' ~ controller ~ '-UI--Calendar--Month-outlet'): '.calendar-month',
}) }}>
```

Only **outlets** are affected. Targets, values, classes and actions all work normally with `{{ controller }}`:

```twig
<button data-action="click->{{ controller }}#toggle">Toggle</button>
<div data-{{ controller }}-target="content">...</div>
<div data-{{ controller }}-url-value="{{ path('app.search') }}">...</div>
```

To avoid the mangling entirely, give the *referenced* controller an all-lowercase identifier instead of using `{{ controller }}` for it — `ui-calendar-month` camelizes cleanly to `uiCalendarMonthOutlet`. That means hand-writing `data-controller="ui-calendar-month"` on that component and giving up the derived name, so bracket access is usually the lesser evil.

---

## Assets Not Injected

### Missing render_component_assets()

```twig
{# ❌ No placeholder in the response -> AssetResponseListener finds nothing to replace #}
<head>
    <title>{{ title }}</title>
</head>

{# ✅ #}
<head>
    <title>{{ title }}</title>
    {{ render_component_assets() }}
</head>
```

The function only emits a marker string; the response listener does the real work. Confirm with `curl -s http://localhost:8000/ | grep __UX_TWIG_COMPONENT_ASSETS__` — if the marker is still in the output, the listener never ran; if neither the marker nor any `<link>` is there, the template never called the function.

### Sub-request or non-HTML response

HTML rewriting is skipped unless `$event->isMainRequest()`. A placeholder rendered inside an ESI/`render()` sub-request is never replaced. Put it in the main layout.

### Filename does not match the class

Auto-discovery looks for exactly `<ShortClassName>.css` / `.js` / `.html.twig` next to the class file.

```text
src_component/UI/Alert/
├── Alert.php
├── alert.css      ❌ lowercase -> not found
├── styles.css     ❌ different name -> not found
└── Alert.css      ✅
```

Renaming the class means renaming all four files. For a name you cannot change, declare it explicitly:

```php
#[AsSdcComponent(css: 'UI/Alert/styles.css')]
```

### Bare #[Asset] does nothing

```php
#[Asset]                              // ❌ no path -> skipped entirely
#[Asset(path: 'vendor/x.css')]        // ✅
```

Auto-discovery is controlled by `ux_sdc.auto_discovery`, not by the presence of `#[Asset]`. Adding the attribute to "enable" discovery is a misconception; the attribute is only for assets that are *not* co-located.

### Unknown asset type

`AssetResponseListener` renders an asset only when its type is `css` or `js`, or when the type is empty and the path ends in `.css` / `.js`. A `.woff2` or `.svg` declared via `#[Asset]` is silently dropped — load fonts and images from CSS instead.

### Component never rendered

The registry is filled at render time. A component behind a false condition contributes no CSS — that is the feature, but it also means `{% if false %}` in a layout is a plausible cause of "my CSS vanished".

---

## Template Not Found / Wrong Template

Template mapping needs the `.html.twig` to resolve to a path **relative to a known Twig root**. `ComponentMetadataResolver::findShortestRelativePath()` checks `%kernel.project_dir%/src_component`, `twig.default_path`, `ux_sdc.ux_components_dir` (plus every configured `twig.paths` entry at build time) and picks the shortest match. A component directory outside all of them gets no template override and falls back to the UX TwigComponent default of `template_directory` + name.

If a component renders the wrong template, two roots probably overlap and the "shortest path" is ambiguous — flatten the Twig path configuration.

---

## Namespace / Directory Mapping

### PSR-4 must match the bundle config

`component_namespace` and `ux_components_dir` are one PSR-4 pair, and Composer needs the identical mapping:

```yaml
ux_sdc:
    ux_components_dir: '%kernel.project_dir%/src_component'
    component_namespace: 'App\Component'
```

```json
{ "autoload": { "psr-4": { "App\\Component\\": "src_component/" } } }
```

```text
src_component/UI/Alert/Alert.php   ->   namespace App\Component\UI\Alert;
```

A mismatch — say `ux_components_dir: src_component/Component` with Composer mapping `src_component/` — leaves classes either unautoloadable or registered under the wrong name. Run `composer dump-autoload` after any change.

### Null component_namespace disables three things at once

No auto-registration of component services, no prepended `twig_component.defaults`, and `{{ controller }}` throws. If components are not found at all, check this first.

### Do not redeclare the namespace

The bundle prepends `twig_component.defaults: { 'App\Component\': { template_directory: '' } }`. Declaring the same namespace in your own `twig_component.yaml` with a different `template_directory` fights it.

---

## Component Name Surprises

`App\Component\UI\Alert\Alert` is `<twig:UI:Alert:Alert />`, not `<twig:Alert />`. The duplicated leaf is inherent to one-directory-per-component. If the tag is too long:

```php
#[AsSdcComponent('Alert')]   // <twig:Alert />
```

This is safe — template and asset discovery use the class's file location, not the component name. It does **not** change `{{ controller }}`, which is always derived from the FQCN.

---

## Stimulus Controllers

### Controllers in assets/controllers/ stop working

Installing the bundle silently disables every Stimulus controller in `assets/controllers/`.

`SdcExtension::prepend()` calls `prependExtensionConfig('stimulus', ['controller_paths' => [uxComponentsDir]])`. StimulusBundle declares that node with `->defaultValue(['%kernel.project_dir%/assets/controllers'])`, and a Symfony default applies only when *no* config source sets the key. Setting it replaces the default outright:

```
WITHOUT config/packages/stimulus.yaml  ->  ['/app/src_component']
WITH    config/packages/stimulus.yaml  ->  ['/app/src_component', '/app/assets/controllers']
```

Nothing errors. `data-controller="my-widget"` just never resolves. The fix is to declare the path yourself — prototyped array nodes append across config sources, so both survive:

```yaml
# config/packages/stimulus.yaml
stimulus:
    controller_paths:
        - '%kernel.project_dir%/assets/controllers'
```

Verify with `php bin/console debug:config stimulus` — both directories must be listed. Any controller that worked before installing the bundle and stopped afterwards is this.

### Trait without interface

```php
// ❌ LogicException: Component namespace is not set.
#[AsSdcComponent]
class Alert
{
    use Stimulus;
}

// ✅
#[AsSdcComponent]
class Alert implements ComponentNamespaceInterface
{
    use Stimulus;
}
```

The trait provides the methods; the `instanceof ComponentNamespaceInterface` check in the render listener is what triggers injection. (The bundle's own README shows the trait without the interface in its first example — the maker template gets it right.)

### `Alert.js` vs `Alert_controller.js`

Two different mechanisms with one letter of difference in effect:

| File | Mechanism | Result |
|---|---|---|
| `Alert.js` | SDC auto-discovery | `<script src>` injected into `<head>` |
| `Alert_controller.js` | `stimulus.controller_paths` | Registered as a Stimulus controller, lazily loaded |

Do not create both for the same behaviour, and do not expect `Alert_controller.js` to appear in `<head>` — it is loaded by the Stimulus loader on demand. If a Stimulus controller is not loading, the cause is `stimulus.enabled: false` or a missing `symfony/stimulus-bundle`, never SDC asset discovery.

### Uppercase in data attribute names

`data-UI--Alert--Alert-target` looks wrong but works: HTML attribute names are matched case-insensitively. Still, keep generating them from `{{ controller }}` rather than typing them — hand-written casing drifts.

---

## Dev vs Prod Divergence

### Stale prod metadata

In prod/test all discovery happened at container build time. Adding `Alert.css` to an existing component and deploying without clearing the cache ships nothing:

```bash
php bin/console cache:clear
```

Any "works in dev, not in prod" asset report starts here.

### SdcMetadataRegistry is empty in dev

`AssetComponentCompilerPass` deletes `sdc_metadata.php` in dev and `SdcExtension` removes `ComponentRenderListener`. Code or tests that read the registry must run in `prod` or `test`, not `dev`.

### Dev caches per class per request

`ComponentMetadataResolver` caches by class name for the lifetime of the request, so a file created mid-request is not seen. Reload the page.

---

## Component Twig Templates Land in public/assets/

`ux_components_dir` is registered as an AssetMapper path so `Alert.css` can be served — which also puts `Alert.html.twig` in the asset map, and `asset-map:compile` writes it into `public/assets/` and lists it in `manifest.json`. Your templates become publicly readable.

```yaml
# config/packages/asset_mapper.yaml
framework:
    asset_mapper:
        excluded_patterns:
            - '*/src_component/*.html.twig'
```

Verify with `php bin/console debug:asset-map | grep twig`. `.php` files are skipped by AssetMapper itself, so only Twig needs excluding. The patterns are globs where `*` crosses `/`.

---

## Asset Ordering & Duplication

### Priority, not declaration order

`getSortedAssets()` sorts by **descending** priority; everything defaults to `0` and keeps insertion order within the same priority. For a reset or token file that must come first, give it a high priority rather than relying on which component rendered first:

```php
#[Asset(path: 'styles/tokens.css', priority: 100)]
```

### Same path rendered twice

Dedup keys on path + type + serialised attributes. These are two entries:

```php
#[Asset(path: 'a.css')]
#[Asset(path: 'a.css', attributes: ['media' => 'print'])]
```

Sometimes intended; usually a copy-paste bug.

---

## Live Component Issues

### Component appears unstyled after an AJAX re-render

Assets are sent as `X-SDC-Assets-CSS` / `X-SDC-Assets-JS` headers on Live responses instead of being injected into HTML, because there is no `<head>` in the fragment. The loader controller has to be mounted:

```twig
<body {{ stimulus_controller(sdc_loader_controller) }}>
```

Check the Network tab: if the headers are on the response but no `<link>` appears in `<head>`, the loader is not mounted or `@symfony/ux-live-component` is missing from the importmap.

### Only on the initial render

A component that *first* appears through a Live re-render (modal content, a lazily revealed panel) has no chance to get its CSS into the initial `<head>`. That is exactly the case the loader exists for — the pattern is not optional for such components.

### Headers absent entirely

`isLiveComponentRequest()` keys on `Accept: application/vnd.live-component+html` on the request or that `Content-Type` on the response. A proxy that rewrites `Accept` makes the listener treat the request as a normal page, try an HTML replacement, find no placeholder, and do nothing.

### Nothing in the registry

If no component in the Live request declared assets, no header is set at all — an absent header is not necessarily a bug.

---

## Container & Linting

The bundle sets `app.ui_components.dir` as a **parameter**. It used to also register a service of that name with `string` as its class, which broke `lint:container`. If you reference it, reference the parameter:

```yaml
services:
    App\Something:
        arguments:
            $dir: '%app.ui_components.dir%'
```

Run `php bin/console lint:container` after changing SDC configuration — it is the cheapest check that the prepended Twig/AssetMapper/Stimulus config is still coherent.

---

## Debugging Checklist

```bash
# Is the component registered, and under what name?
php bin/console debug:twig-component

# Is the CSS a mapped asset? Are templates leaking?
php bin/console debug:asset-map | grep -i alert

# Are BOTH Stimulus controller paths registered?
php bin/console debug:config stimulus

# Which listener is active?
php bin/console debug:event-dispatcher kernel.response
php bin/console debug:container --tag=kernel.event_listener | grep Sdc

# What did the compiler pass actually record? (prod/test only)
cat var/cache/prod/sdc_metadata.php

# Is the container sound after a config change?
php bin/console lint:container
```

| Symptom | First suspect |
|---|---|
| Marker visible in HTML source | Response listener skipped (sub-request, or body rewritten later) |
| No marker, no `<link>` | `render_component_assets()` not called |
| Works in dev, not prod | Stale `sdc_metadata.php` → `cache:clear` |
| `LogicException: Component namespace is not set` | Missing `implements ComponentNamespaceInterface` |
| Component not found by Twig | `component_namespace` null, or PSR-4 mismatch |
| CSS missing only after a Live update | `sdc_loader_controller` not on `<body>` |
| `undefined` outlet in a Stimulus controller | Dot access instead of bracket access on the mangled name |
| Controllers in `assets/controllers/` never fire | Missing `config/packages/stimulus.yaml` |
| Templates visible under `/assets/` | Missing `excluded_patterns` |

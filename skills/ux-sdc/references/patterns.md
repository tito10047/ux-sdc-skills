# UX SDC Patterns

Real-world structures for an SDC codebase.

---

## Recommended Project Structure

Group by role, and **nest each page's private parts inside that page's directory**. Three top-level buckets, with depth used to express ownership:

```text
src_component/
├── UI/                           Generic, reusable, no domain knowledge
│   ├── Button/
│   ├── Badge/
│   └── LocaleSwitcher/
├── Layout/                       Structural page furniture
│   ├── TopBar/
│   └── Footer/
└── Page/                         One directory per page
    ├── Homepage/
    │   ├── Homepage.php          the page component
    │   ├── Homepage.html.twig
    │   ├── Homepage.css
    │   ├── HomepageAction.php    the route, co-located
    │   ├── HomepageAction.html.twig
    │   ├── Hero/                 private to Homepage
    │   ├── Projects/
    │   └── CTA/
    └── Contact/
        ├── Contact.php
        ├── ContactAction.php
        └── Form/
```

```yaml
ux_sdc:
    ux_components_dir: '%kernel.project_dir%/src_component'
    component_namespace: 'App\Component'
```

```json
{ "autoload": { "psr-4": { "App\\": "src/", "App\\Component\\": "src_component/" } } }
```

The nesting rule is the whole point: **a component lives inside the component that owns it.** `Hero` exists only for `Homepage`, so it sits in `Page/Homepage/Hero/`. The moment a second page needs it, it moves up — to `Page/` if two pages share it, to `UI/` or `Layout/` if it has become generic. Directory depth is a readable statement about reuse, and "where does this belong" has one answer instead of a judgement call.

Names follow the directory path, since the component name is the FQCN minus `component_namespace` with `\` → `:`:

| Directory | Class | Tag |
|---|---|---|
| `UI/Button/` | `App\Component\UI\Button\Button` | `<twig:UI:Button:Button />` |
| `Layout/TopBar/` | `App\Component\Layout\TopBar\TopBar` | `<twig:Layout:TopBar:TopBar />` |
| `Page/Homepage/` | `App\Component\Page\Homepage\Homepage` | `<twig:Page:Homepage:Homepage />` |
| `Page/Homepage/Hero/` | `App\Component\Page\Homepage\Hero\Hero` | `<twig:Page:Homepage:Hero:Hero />` |

Two consequences worth accepting up front:

- **Tags get long.** `<twig:Page:Homepage:Hero:Hero />` is verbose, but it is only ever written once — inside `Homepage.html.twig`, next to the directory it names. Shorten a frequently used `UI/` component with an explicit name (`#[AsSdcComponent('Button')]`) rather than flattening the structure.
- **Stimulus controller names get long too** — `Page--Homepage--Hero--Hero`. Always generate them from `{{ controller }}`, never by hand, and see the outlet gotcha below.

The payoff is deletion: removing `Page/Homepage/` removes the route, the page, its three sections, their markup, styles and behaviour, in one `rm -r`, with nothing left behind in a global stylesheet or a routing file.

Add a flat `Component/` bucket only if feature blocks are genuinely shared by several pages and belong to neither `UI/` nor `Layout/`; with page-local nesting most projects never need it.

---

## Building a Page: the `--action` Workflow

This is the core loop of an SDC codebase. The Symfony controller lives **inside** the page component's directory; it only routes and renders a thin wrapper, while every piece of logic and markup stays in components.

### Step 1 — scaffold the page and its route

```bash
php bin/console make:sdc-component Page:Homepage --action
```

`--action` adds two files to the usual four:

```text
src_component/Page/Homepage/
├── Homepage.php                # the page component
├── Homepage.html.twig          # where the page is composed
├── Homepage.css
├── Homepage_controller.js      # with --stimulus
├── HomepageAction.php          # the route
└── HomepageAction.html.twig    # extends the layout, renders the component
```

The generated action, verbatim apart from formatting:

```php
// src_component/Page/Homepage/HomepageAction.php
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
{# src_component/Page/Homepage/HomepageAction.html.twig #}
{% extends 'layout.html.twig' %}

{% block content %}
    <twig:Page:Homepage:Homepage />
{% endblock %}
```

**Two things to fix immediately after generating.** The route path is hardcoded `/en` with name `app.<lowercased name>` — change it. And the wrapper extends `layout.html.twig` expecting a `content` block, while Symfony Flex ships `base.html.twig` with a `body` block; either create `templates/layout.html.twig` once as your project convention, or adjust the two lines per page.

The `render()` path works because the bundle registers `ux_components_dir` as an unnamed Twig path, so `Page/Homepage/HomepageAction.html.twig` resolves straight out of the component tree. The route is registered because `SdcExtension` autoconfigures every class under `ux_components_dir`, which makes `#[Route]` tag the service — see [gotchas.md](gotchas.md) for the Symfony version that requires.

### Step 2 — generate the sections the page is composed of

The maker nests to any depth, so the page's private parts are created straight into its directory:

```bash
php bin/console make:sdc-component Page:Homepage:Hero
php bin/console make:sdc-component Page:Homepage:Projects
php bin/console make:sdc-component Page:Homepage:CTA
```

```text
src_component/Page/Homepage/
├── Homepage.php
├── Homepage.html.twig
├── HomepageAction.php
├── HomepageAction.html.twig
├── Hero/
│   ├── Hero.php
│   ├── Hero.html.twig
│   └── Hero.css
├── Projects/
└── CTA/
```

### Step 3 — compose them in the page component

```twig
{# src_component/Page/Homepage/Homepage.html.twig #}
<div {{ attributes.defaults({ class: 'homepage' }) }}>
    <twig:Page:Homepage:Hero :headline="headline" />

    <twig:Page:Homepage:Projects :projects="this.featuredProjects" />

    <twig:Page:Homepage:CTA />
</div>
```

```php
// src_component/Page/Homepage/Homepage.php
namespace App\Component\Page\Homepage;

use App\Repository\ProjectRepository;
use Tito10047\UX\Sdc\Attribute\AsSdcComponent;

#[AsSdcComponent]
class Homepage
{
    public string $headline = 'Welcome';

    public function __construct(
        private readonly ProjectRepository $projects,
    ) {}

    public function getFeaturedProjects(): array
    {
        return $this->projects->findFeatured();
    }
}
```

Each section's CSS is injected independently, so the homepage ships exactly `Homepage.css` + `Hero.css` + `Projects.css` + `CTA.css` and nothing else. A section hidden behind a condition that did not run contributes no CSS at all.

### Where the data should live

Fetch in the page component and pass down, as above — the page is the single place that talks to repositories, and `Hero` / `Projects` / `CTA` stay dumb and trivially previewable. Inject a repository into a section only when it owns a genuinely independent concern (a live-updating panel, say).

Keep the `Action` class free of dependencies and untested; test the page component instead. A one-line controller that only picks a template has nothing worth asserting.

---

## Component with a Stimulus Controller

```text
src_component/UI/Dropdown/
├── Dropdown.php
├── Dropdown.html.twig
├── Dropdown.css
└── Dropdown_controller.js
```

```php
namespace App\Component\UI\Dropdown;

use Tito10047\UX\Sdc\Attribute\AsSdcComponent;
use Tito10047\UX\Sdc\Twig\ComponentNamespaceInterface;
use Tito10047\UX\Sdc\Twig\Stimulus;

#[AsSdcComponent]
class Dropdown implements ComponentNamespaceInterface
{
    use Stimulus;

    public string $label = '';
    public bool $closeOnSelect = true;
}
```

```twig
{# Dropdown.html.twig -- everything derives from {{ controller }} #}
<div {{ attributes.defaults({
    class: 'dropdown',
    'data-controller': controller,
    ('data-' ~ controller ~ '-close-on-select-value'): closeOnSelect ? 'true' : 'false',
}) }}>
    <button
        data-action="click->{{ controller }}#toggle"
        data-{{ controller }}-target="trigger"
    >{{ label }}</button>

    <div data-{{ controller }}-target="panel" hidden>
        {% block content %}{% endblock %}
    </div>
</div>
```

```js
// Dropdown_controller.js
import { Controller } from '@hotwired/stimulus';

/* stimulusFetch: 'lazy' */
export default class extends Controller {
    static targets = ['trigger', 'panel'];
    static values = { closeOnSelect: Boolean };

    toggle() {
        this.panelTarget.hidden = !this.panelTarget.hidden;
    }
}
```

`Dropdown.css` is injected into `<head>`; `Dropdown_controller.js` is loaded lazily by the Stimulus loader. Neither needs a line of configuration.

---

## Outlets Between Two SDC Components

Outlets are the one place where SDC controller names need care — see [gotchas.md](gotchas.md) for why. The working shape:

```text
src_component/UI/Calendar/
├── Month/
│   ├── Month.php
│   ├── Month.html.twig
│   └── Month_controller.js
└── Toolbar/
    ├── Toolbar.php
    ├── Toolbar.html.twig
    └── Toolbar_controller.js
```

`App\Component\UI\Calendar\Month\Month` → controller `UI--Calendar--Month--Month`.

```twig
{# Toolbar.html.twig -- the attribute keeps the double dashes #}
<div {{ attributes.defaults({
    'data-controller': controller,
    ('data-' ~ controller ~ '-UI--Calendar--Month--Month-outlet'): '[data-controller~="UI--Calendar--Month--Month"]',
}) }}>
    <button data-action="click->{{ controller }}#next">Next month</button>
</div>
```

```js
// Toolbar_controller.js
import { Controller } from '@hotwired/stimulus';

export default class extends Controller {
    static outlets = ['UI--Calendar--Month--Month'];

    next() {
        // Property names collapse "--" to "-": bracket access is mandatory.
        if (this['hasUI-Calendar-Month-MonthOutlet']) {
            this['UI-Calendar-Month-MonthOutlet'].refresh();
        }
    }
}
```

Define the names once as constants if a controller talks to several outlets:

```js
const MONTH = 'UI-Calendar-Month-Month';

export default class extends Controller {
    static outlets = ['UI--Calendar--Month--Month'];

    get month() { return this[`${MONTH}Outlet`]; }
    get hasMonth() { return this[`has${MONTH}Outlet`]; }

    next() {
        if (this.hasMonth) this.month.refresh();
    }
}
```

When a component only needs to *notify* rather than call methods, a browser event is simpler and sidesteps the naming entirely:

```js
this.dispatch('next', { prefix: 'calendar', detail: { delta: 1 } });
```

```twig
<div data-action="calendar:next@window->{{ controller }}#onNext">
```

---

## Design Tokens and Global CSS First

Per-component CSS still needs a predictable base. Attach shared files with a high priority so they sort ahead of everything:

```php
namespace App\Component\Layout\Shell\Shell;

#[AsSdcComponent]
#[Asset(path: 'styles/reset.css', priority: 1000)]
#[Asset(path: 'styles/tokens.css', priority: 900)]
class Shell {}
```

Because `Shell` wraps every page, its assets are always registered, and descending priority puts them before component CSS that defaults to `0`. Combined with `@layer components { ... }` inside each component file, cascade order stops being a per-page accident.

---

## Wrapping a Third-Party Library

```php
namespace App\Component\UI\DatePicker;

use Tito10047\UX\Sdc\Attribute\AsSdcComponent;
use Tito10047\UX\Sdc\Attribute\Asset;
use Tito10047\UX\Sdc\Twig\ComponentNamespaceInterface;
use Tito10047\UX\Sdc\Twig\Stimulus;

#[AsSdcComponent]
#[Asset(path: 'vendor/flatpickr/flatpickr.min.css', priority: 50)]
class DatePicker implements ComponentNamespaceInterface
{
    use Stimulus;

    public string $name = 'date';
    public ?string $minDate = null;
}
```

The vendor CSS ships only on pages with a date picker. The JS side belongs in `DatePicker_controller.js` as a normal importmap dependency — use `#[Asset]` for stylesheets and for scripts that genuinely must be a plain `<script>` tag, not for anything the Stimulus controller can `import`.

```twig
<input
    type="text"
    name="{{ name }}"
    {{ attributes.defaults({
        'data-controller': controller,
        ('data-' ~ controller ~ '-min-date-value'): minDate,
    }) }}
>
```

---

## Live Component as an SDC Component

A Live component nests exactly like a static one — here it is a section of the catalogue page:

```text
src_component/Page/Catalog/ProductSearch/
├── ProductSearch.php
├── ProductSearch.html.twig
├── ProductSearch.css
└── ProductSearch_controller.js   (optional)
```

```php
namespace App\Component\Page\Catalog\ProductSearch;

use App\Repository\ProductRepository;
use Symfony\UX\LiveComponent\Attribute\AsLiveComponent;
use Symfony\UX\LiveComponent\Attribute\LiveProp;
use Symfony\UX\LiveComponent\DefaultActionTrait;
use Tito10047\UX\Sdc\Twig\ComponentNamespaceInterface;
use Tito10047\UX\Sdc\Twig\Stimulus;

#[AsLiveComponent]
class ProductSearch implements ComponentNamespaceInterface
{
    use DefaultActionTrait;
    use Stimulus;

    #[LiveProp(writable: true, url: true)]
    public string $query = '';

    public function __construct(
        private readonly ProductRepository $products,
    ) {}

    public function getResults(): array
    {
        return $this->products->search($this->query);
    }
}
```

```twig
{# {{ attributes }} carries both the live controller and ours #}
<div {{ attributes.defaults({ class: 'product-search' }) }}>
    <input type="search" data-model="debounce(300)|query">

    <div data-loading="addClass(opacity-50)">
        {% for product in this.results %}
            <twig:UI:ProductCard:ProductCard :product="product" />
        {% endfor %}
    </div>
</div>
```

No `#[AsSdcComponent]` and no `#[Asset]` are needed — auto-discovery already sees `ProductSearch.css`. `{{ attributes }}` must be rendered on the root element for Live to work at all; `attributes.defaults()` preserves what Live injected.

Note the nested `<twig:UI:ProductCard:ProductCard />`: on the initial page load its CSS is injected normally, and on each AJAX re-render it arrives via `X-SDC-Assets-CSS` — which only works if `<body>` carries the loader:

```twig
<body {{ stimulus_controller(sdc_loader_controller) }}>
```

---

## Deferred Live Component

```twig
{# in Page/Dashboard/Dashboard.html.twig -- CSS arrives with the deferred response #}
<twig:Page:Dashboard:Metrics:Metrics defer>
    <twig:UI:Spinner:Spinner />
</twig:Page:Dashboard:Metrics:Metrics>
```

The placeholder is rendered inline on the main response, so `Spinner.css` is in the initial `<head>`; `Dashboard.css` arrives with the deferred fragment's headers. This is the pattern that most visibly breaks without `sdc_loader_controller`.

---

## Nested Components and Slots

Composition is plain TwigComponent — SDC adds nothing and takes nothing away:

```twig
{# src_component/UI/Card/Card.html.twig #}
<div {{ attributes.defaults({ class: 'card' }) }}>
    <header class="card__header">{% block header %}{% endblock %}</header>
    <div class="card__body">{% block content %}{% endblock %}</div>
</div>
```

```twig
<twig:UI:Card:Card>
    <twig:block name="header">
        <twig:UI:Badge:Badge type="warning" label="Draft" />
    </twig:block>

    <twig:block name="content">
        <p>{{ post.excerpt }}</p>
    </twig:block>
</twig:UI:Card:Card>
```

`Card.css`, `Badge.css` and the outer component's CSS are all registered, deduplicated, and injected once.

---

## Conditional and Media-Specific Stylesheets

```php
#[AsSdcComponent]
#[Asset(path: 'Invoice.print.css', type: 'css', attributes: ['media' => 'print'])]
class Invoice {}
```

`Invoice.css` (auto-discovered) renders without attributes; `Invoice.print.css` renders as `<link rel="stylesheet" href="..." media="print">`. The auto-discovery naming rule means only the exact `Invoice.css` is found automatically — any variant needs the explicit attribute.

---

## Migrating a Classic Component to SDC

Before:

```text
src/Twig/Components/Alert.php
templates/components/Alert.html.twig
assets/styles/components/_alert.scss   (imported by app.scss)
assets/controllers/alert_controller.js
```

After:

```text
src_component/UI/Alert/
├── Alert.php
├── Alert.html.twig
├── Alert.css
└── Alert_controller.js
```

Steps:

1. Create the directory and `git mv` the four files, renaming them to the short class name.
2. Change the namespace to `App\Component\UI\Alert`; swap `#[AsTwigComponent]` for `#[AsSdcComponent]` (optional, but it documents the intent) and drop any `template:` argument.
3. Remove the `@import` of the old stylesheet from `app.scss`/`app.css`. This is the step people forget — leave it in and the CSS ships globally *and* per component.
4. Update the tag: `<twig:Alert />` → `<twig:UI:Alert:Alert />`, or keep the short tag with `#[AsSdcComponent('Alert')]`.
5. If the old Stimulus controller was referenced as `data-controller="alert"`, add `implements ComponentNamespaceInterface` + `use Stimulus` and switch the template to `{{ controller }}` — the identifier is now `UI--Alert--Alert`.
6. `php bin/console debug:twig-component` to confirm the name, then `debug:asset-map | grep Alert` to confirm the CSS is mapped and the template is not.

Migrate leaf components (`UI/`) first: they have the fewest inbound references, and each one you move shrinks the global stylesheet.

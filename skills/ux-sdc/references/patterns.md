# UX SDC Patterns

Real-world structures for an SDC codebase.

---

## Recommended Project Structure

Group components by role, not by type. Four top-level buckets cover almost everything:

```text
src_component/
├── UI/                  Generic, reusable, no domain knowledge
│   ├── Button/
│   ├── Spinner/
│   └── Tabs/
├── Layout/              Structural page furniture
│   ├── TopBar/
│   ├── Footer/
│   └── FlashMessage/
├── Component/           Reusable feature blocks with domain knowledge
│   ├── ProductCard/
│   └── CartSummary/
└── Page/                Whole pages, one per route
    ├── Homepage/
    └── Checkout/
```

```yaml
ux_sdc:
    ux_components_dir: '%kernel.project_dir%/src_component'
    component_namespace: 'App\Component'
```

```json
{ "autoload": { "psr-4": { "App\\": "src/", "App\\Component\\": "src_component/" } } }
```

Tags follow directly: `<twig:UI:Button:Button />`, `<twig:Layout:TopBar:TopBar />`, `<twig:Page:Homepage:Homepage />`.

The payoff is that a page's CSS budget is the sum of what it actually rendered. Deleting `src_component/Component/CartSummary/` deletes the feature — class, markup, styles, behaviour — with no dangling references in a global stylesheet.

---

## Page Component with a Co-located Action

The strongest structural move SDC enables: put the Symfony controller **inside** the page component directory. The controller only routes and renders a thin wrapper; all logic and markup stay in the component.

```bash
php bin/console make:sdc-component Page:Checkout --action
```

```text
src_component/Page/Checkout/
├── Checkout.php              # the component: logic + props
├── Checkout.html.twig        # the markup
├── Checkout.css
├── CheckoutAction.php        # the route
└── CheckoutAction.html.twig  # extends layout, renders <twig:Page:Checkout:Checkout />
```

```php
// src_component/Page/Checkout/CheckoutAction.php
namespace App\Component\Page\Checkout;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class CheckoutAction extends AbstractController
{
    #[Route('/checkout', name: 'app.checkout')]
    public function index(): Response
    {
        return $this->render('Page/Checkout/CheckoutAction.html.twig');
    }
}
```

```twig
{# src_component/Page/Checkout/CheckoutAction.html.twig #}
{% extends 'base.html.twig' %}

{% block body %}
    <twig:Page:Checkout:Checkout />
{% endblock %}
```

```php
// src_component/Page/Checkout/Checkout.php
namespace App\Component\Page\Checkout;

use App\Repository\CartRepository;
use Tito10047\UX\Sdc\Attribute\AsSdcComponent;

#[AsSdcComponent]
class Checkout
{
    public function __construct(
        private readonly CartRepository $carts,
    ) {}

    public function getItems(): array
    {
        return $this->carts->currentItems();
    }
}
```

Keep the action dependency-free and untested; test the component.

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

```text
src_component/Component/ProductSearch/
├── ProductSearch.php
├── ProductSearch.html.twig
├── ProductSearch.css
└── ProductSearch_controller.js   (optional)
```

```php
namespace App\Component\Component\ProductSearch;

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
{# CSS arrives with the deferred response, via the loader controller #}
<twig:Component:Dashboard:Dashboard defer>
    <twig:UI:Spinner:Spinner />
</twig:Component:Dashboard:Dashboard>
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

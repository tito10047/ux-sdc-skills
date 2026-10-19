# UX SDC Agent Skills

An AI agent skill for [`tito10047/ux-sdc`](https://github.com/tito10047/ux-sdc) — the Symfony bundle that implements the **Single Directory Component** pattern for Symfony UX.

Give your coding agent accurate knowledge of how SDC components are discovered, how their assets reach the browser, and which traps the bundle sets along the way — instead of letting it guess from the package name.

## What it covers

| Area | Contents |
|---|---|
| Setup | `ux_components_dir` ↔ `component_namespace` PSR-4 pairing, the `render_component_assets()` placeholder, the required `stimulus.yaml`, excluding templates from the public asset map |
| Conventions | Directory layout, component name derivation (`App\Component\UI\Alert\Alert` → `<twig:UI:Alert:Alert />`), exact-filename auto-discovery |
| Assets | `#[AsSdcComponent(css:, js:)]`, repeatable `#[Asset]`, priority, HTML attributes, `AssetRegistry` dedup, `Link: preload` headers |
| Stimulus | The `Stimulus` trait + `ComponentNamespaceInterface`, `{{ controller }}`, targets/actions/values, and the outlet property-name mangling workaround |
| Live Components | `X-SDC-Assets-CSS` / `X-SDC-Assets-JS`, the `sdc_loader_controller`, deferred components |
| Dev vs prod | Compiler-pass metadata cache vs runtime resolution, and why "works in dev, not in prod" is almost always a stale cache |

Every claim in the skill was verified against the bundle's source at `0.4.0` and against Symfony's own code — including the ones that contradict the bundle's README.

## Install

### Claude Code

```
/plugin marketplace add tito10047/ux-sdc-skills
/plugin install ux-sdc-skills
```

### Other agents

The skill is a plain directory of Markdown following the [agentskills.io](https://agentskills.io) layout. Clone it and point your tool at `skills/ux-sdc/`:

```bash
git clone https://github.com/tito10047/ux-sdc-skills.git
```

`AGENTS.md`, `CLAUDE.md` and `llms.txt` are provided for agents that read those conventions.

## Layout

```text
skills/ux-sdc/
├── SKILL.md                  Core concepts, setup, naming, assets, Stimulus, Live
└── references/
    ├── api.md                Attributes, services, events, Twig, maker, config
    ├── patterns.md           Project structure, page actions, outlets, Live, migration
    └── gotchas.md            Failure modes, root causes, debugging checklist
```

`SKILL.md` is what the agent loads first; the references are read on demand.

## Companion skills

This skill covers only what SDC adds. For the Symfony UX packages underneath it — TwigComponent, LiveComponent, Stimulus, Turbo, UX Icons, UX Map — use [`smnandre/symfony-ux-skills`](https://github.com/smnandre/symfony-ux-skills), whose structure this repository follows.

## License

MIT. See [LICENSE](LICENSE).

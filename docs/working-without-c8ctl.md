# Working without c8ctl (Camunda 8 SaaS + Web Modeler)

The Camunda skills in `.claude/skills/` assume `c8ctl` is installed as the execution
backbone (lint, deploy, run, element-template apply, FEEL eval, runtime ops). When
`c8ctl` is **not** available — e.g. authoring locally and shipping through **Camunda 8
SaaS + Web Modeler git sync** — use the alternatives below. Authoring (writing the
BPMN/DMN/form/connector XML & JSON) works unchanged; only the validate → deploy → run
→ operate half needs substitutes.

## Capability map

| c8ctl capability | Command (unavailable) | SaaS / no-c8ctl alternative |
|---|---|---|
| BPMN structural lint | `c8ctl bpmn lint` | `npx --yes bpmnlint <file>` locally or in CI; or open the diagram in Web Modeler (Problems panel) |
| BPMN format | `c8ctl bpmn format` | Hand-write canonical bpmn-js style, or round-trip once through Web Modeler |
| DMN lint | (via dmnlint) | `npx --yes dmnlint <file>` (already c8ctl-independent) |
| FEEL evaluation | `c8ctl feel evaluate` | Web Modeler FEEL editor / Play mode; or the online FEEL playground |
| Element-template discovery | `c8ctl element-template search/info/get-properties` | Browse the [connectors marketplace](https://marketplace.camunda.com/) / template JSON in the camunda/connectors repo |
| Element-template apply | `c8ctl element-template apply` | Apply the template in Web Modeler (drag connector / "Change element template"), or hand-write the `zeebe:modelerTemplate*` + `ioMapping` + `taskHeaders` |
| Deploy resources | `c8ctl deploy` | Web Modeler **Deploy** button, or git sync → deploy from the SaaS project |
| Deploy + start | `c8ctl run` | Web Modeler **Run**, or **Play** mode for a single instance |
| Auto-redeploy on save | `c8ctl watch` | Web Modeler edits deploy on demand; no watch equivalent |
| Start / inspect / cancel instances | `c8ctl create/get/cancel pi` | **Operate** (SaaS) UI |
| Complete / assign user tasks | `c8ctl complete/assign ut` | **Tasklist** (SaaS) UI |
| Incidents | `c8ctl list/resolve inc` | **Operate** → Incidents |
| Messages | `c8ctl publish/correlate msg` | Operate / API; or Play mode message events |
| Local cluster | `c8ctl cluster start` | Not needed — use the SaaS cluster (dev stage) directly |

## Lint pipeline (push → Web Modeler)

| Step | What happens | Lint? | SaaS alternative |
|---|---|---|---|
| Push to GitHub | Files land in the repo branch | ❌ No (git doesn't lint) | Add a CI workflow running `bpmnlint`/`dmnlint` so pushes are linted before sync |
| Web Modeler git sync → pull | Imports `.bpmn`/`.dmn`/`.form` into the project | ❌ Sync moves files; no lint gate | Gate in CI (above) or lint locally before pushing |
| Open diagram in Web Modeler editor | Editor validates with the same bpmn-io / dmn-lint rules; shows problems inline + Problems panel | ✅ Yes — SaaS-native lint | Use the Problems panel as the review gate |
| Click Deploy / Run | Zeebe engine validates on deploy | ⚠️ Partial — DI-less / warning-level diagrams still deploy | Deploy to a non-prod stage first; use Play / Process test to catch what lint misses |

## CI lint workflow (optional pre-sync gate)

```yaml
name: Lint Camunda models
on: [push, pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - name: Lint DMN
        run: |
          [ -f .dmnlintrc ] || echo '{ "extends": "dmnlint:recommended" }' > .dmnlintrc
          find . -name '*.dmn' -print0 | xargs -0 -r npx --yes dmnlint
      - name: Lint BPMN
        run: |
          find . -name '*.bpmn' -print0 | xargs -0 -r npx --yes bpmnlint
```

## Caveats

- A successful deploy is **not** proof a file is clean — Zeebe deploys DI-less and
  warning-level BPMN happily. Treat the Web Modeler editor (or CI lint) as the real gate.
- Hand-written connectors omit the `zeebe:modelerTemplateIcon` blob — Web Modeler still
  recognizes the template by ID, but the icon won't render until you re-apply the
  template in the Modeler.

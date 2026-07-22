# Hello World Component: Technical Design

## Component Identity
- **Resource Type**: `wknd/components/helloworld`
- **File Location**: `ui.apps/src/main/content/jcr_root/apps/wknd/components/helloworld/`
- **Sling Model**: `com.adobe.aem.guides.wknd.core.models.HelloWorldModel` (`core/src/main/java/com/adobe/aem/guides/wknd/core/models/`)
- **Purpose**: Reference/demo component that prints an author-entered text property alongside diagnostic info (resource type and containing page path) computed by a Sling Model. *(Inferred — no explicit dialog description; purpose is clear from code structure and naming.)*
- **Resource Super Type**: none (standalone component, no inheritance)
- **Status**: Reference/example component (has unit test `HelloWorldModelTest.java`); not intended for production content authoring.

## Sling Model Configuration Properties

| Property | Type | Required | Default | Constraint | Source |
|---|---|---|---|---|---|
| `resourceType` | String | NO | `"No resourceType"` | injected from `sling:resourceType` (`PROPERTY_RESOURCE_TYPE`), `InjectionStrategy.OPTIONAL` | `@ValueMapValue(name=PROPERTY_RESOURCE_TYPE)` + `@Default` |
| `text` | String | NO | - | plain text, used directly by HTL (not via the Sling Model — see below) | Dialog only, read directly as `${properties.text}` in HTL |

**Computed / derived properties:**
- `message` — `String`, built in `@PostConstruct init()`: `"Hello World!\nResource type is: {resourceType}\nCurrent page is: {currentPagePath}\n"`. `currentPagePath` is resolved via `PageManager.getContainingPage(currentResource)`, defaulting to `""` if no containing page is found (`Optional` chain).

⚠️ Note: `text` is a dialog-authored property but is **not** part of the `HelloWorldModel` Sling Model — it's read straight off `properties` (the resource ValueMap) in the HTL. The Sling Model only supplies the diagnostic `message`. These are two independent, parallel data sources rendered side-by-side.

## Author Dialog Fields

| Field Label | Field Name | Type | Required | Default | Validation |
|---|---|---|---|---|---|
| Text | `./text` | textfield | NO | - | - |

Single-field dialog, single column layout (`fixedcolumns`), no tabs.

## HTL Template Rendering

### Template
`helloworld.html`

### HTML Output
```html
<div class="cmp-helloworld" data-cmp-is="helloworld">
    <h2 class="cmp-helloworld__title">Hello World Component</h2>
    <div class="cmp-helloworld__item"> <!-- only if properties.text is truthy -->
        <p class="cmp-helloworld__item-label">Text property:</p>
        <pre class="cmp-helloworld__item-output" data-cmp-hook-helloworld="property">${properties.text}</pre>
    </div>
    <div class="cmp-helloworld__item"> <!-- only if model.message is truthy -->
        <p class="cmp-helloworld__item-label">Model message:</p>
        <pre class="cmp-helloworld__item-output" data-cmp-hook-helloworld="model">${model.message}</pre>
    </div>
</div>
```

### Properties Used
- `properties.text` (raw ValueMap, gated by `data-sly-test`)
- `model.message` (from `HelloWorldModel`, gated by `data-sly-test`)

### Rendering Notes
- Uses `data-cmp-is="helloworld"` and `data-cmp-hook-helloworld="..."` attributes — Core Components-style JS hook convention, suggesting a client-side behavior is expected to bind to this markup (see `helloworld.ts`/`.js`), though the current TS file (`helloworld.ts`) does nothing with the DOM (see CSS/JS section).
- Both blocks are conditionally rendered independently; if both `text` and `message` are empty, only the `<h2>` title renders.

## CSS Classes & Design Tokens

### BEM Structure
```
.cmp-helloworld (block)
├── .cmp-helloworld__title (element)
├── .cmp-helloworld__item (element, repeated)
├── .cmp-helloworld__item-label (element)
└── .cmp-helloworld__item-output (element)
```

### CSS Class Reference
| Class | Type | Purpose | Design Tokens |
|---|---|---|---|
| `cmp-hello-world-sass` | — | Demo-only class demonstrating a SCSS `:before` pseudo-element (content: `>`) | none |

⚠️ **Discrepancy**: The SCSS file (`helloworld.scss`) defines `.cmp-hello-world-sass`, which does **not** match any class used in `helloworld.html` (which uses `cmp-helloworld`, `cmp-helloworld__title`, etc.). The stylesheet appears to be leftover/demo boilerplate not wired to the actual template markup — **no styling currently applies to this component's real output.**

### Design Tokens Used
None — SCSS file uses no `$variable` or CSS custom property references.

## Client-Side JS
- `helloworld.ts` / `helloworld.js`: defines a `HelloWorld` class that only sets a local unused `const tsString = "Hello World"` in its constructor, then exports a singleton instance. It does not attach to the DOM, does not use `data-cmp-hook-helloworld`, and has no observable effect. **Effectively dead/demo code.**

## Sling Model API Export
No `ComponentExporter`/`@Exporter` annotation — `HelloWorldModel` is not exposed via `.model.json`. Render-only model.

## Accessibility Requirements
- No ARIA attributes present.
- `<pre>` blocks used for diagnostic text output — acceptable for a debug/reference component, not a pattern to replicate in production-facing components.
- Not evaluated for WCAG compliance — this is an internal reference/demo component, not customer-facing.

## Current Constraints & Limitations
- Demo/reference component only — not intended to represent real content-authoring functionality.
- CSS is disconnected from markup (class name mismatch) — styling has no visible effect.
- JS hooks (`data-cmp-hook-helloworld`) are present in markup but unused by the TS/JS file — no actual client-side behavior implemented despite the hook wiring.
- `text` property bypasses the Sling Model entirely, inconsistent with the model-driven pattern used by other components (e.g., Byline, ImageList) in this project.

## Related Components
None — standalone reference component with no resourceSuperType and no dependents found in the codebase.

# Byline Component: Technical Design

## Component Identity
- **Resource Type**: `wknd/components/byline`
- **File Location**: `ui.apps/src/main/content/jcr_root/apps/wknd/components/byline/`
- **Sling Model**: `com.adobe.aem.guides.wknd.core.models.Byline` (interface) / `BylineImpl` (impl, in `core/src/main/java/com/adobe/aem/guides/wknd/core/models/`)
- **Purpose**: Displays a contributor's byline — a small circular avatar image, name, and a comma-separated list of occupations. *(Explicitly defined in `_cq_dialog` `jcr:description`.)*
- **Resource Super Type**: `core/wcm/components/image/v3/image` (extends Core Components Image, reusing its asset-picking dialog tab and Image Sling Model)
- **Status**: Production Ready (has unit tests: `BylineImplTest.java`)

## Sling Model Configuration Properties

| Property | Type | Required | Default | Constraint | Source |
|---|---|---|---|---|---|
| `name` | String | YES (enforced in dialog + `isEmpty()`) | - | non-blank | `@ValueMapValue` |
| `occupations` | `List<String>` | NO (dialog) / effectively required for non-empty state | - | sorted alphabetically ascending on read via `Collections.sort()` (despite Javadoc claiming "descending") | `@ValueMapValue` |
| *(inherited)* image (`fileReference`, `alt`, etc.) | - | YES for non-empty state | - | valid DAM asset | Delegated to Core Components `Image` model via `ModelFactory.getModelFromWrappedRequest()` in `@PostConstruct` |

**Computed / derived properties (not `@ValueMapValue`):**
- `isEmpty()` — `boolean`. Returns `true` if `name` is blank, OR `occupations` is null/empty, OR the delegated `Image` model is null / has a blank `src`. Used by the HTL template to decide whether to render the component or an editor placeholder.
- `image` — internal `Image` field, populated in `@PostConstruct init()` by adapting the wrapped request to the Core Components `Image` Sling Model (delegation pattern via `ModelFactory`).

⚠️ **Note (discrepancy)**: The `getOccupations()` Javadoc says "sorted alphabetically in a descending order" but the implementation calls `Collections.sort(occupations)`, which is **ascending**. Flagging for manual review — likely a stale comment, not a bug in behavior.

## Author Dialog Fields

Dialog extends the Core Components Image dialog (`sling:resourceSuperType="core/wcm/components/image/v3/image"`), overlaying only the `_cq_dialog` and `_cq_design_dialog`.

| Field Label | Field Name | Type | Required | Default | Validation |
|---|---|---|---|---|---|
| Name | `./name` | textfield | YES | - | - |
| Occupations | `./occupations` | multifield (textfield) | NO | - | free text per entry |
| *(Asset tab)* | inherited | Core Image asset picker | implicitly required for non-empty render | - | inherited from Core Components Image |

**Dialog customizations over the inherited Image dialog:**
- `asset` tab: explicitly shown (`sling:hideResource="false"`)
- `metadata` tab: hidden (`sling:hideResource="true"`)
- Design dialog hides several Core Image options not relevant to Byline: `decorative`, `altValueFromDAM`, `titleValueFromDAM`, `displayCaptionPopup`, `disableUuidTracking`, accordion `orientation`, `crop`

## HTL Template Rendering

### Template
`byline.html`

### HTML Output
```html
<div class="cmp-byline">
    <div class="cmp-byline__image"><!-- Core Image component resource, resourceType=core/wcm/components/image/v2/image --></div>
    <h2 class="cmp-byline__name">${byline.name}</h2>
    <p class="cmp-byline__occupations">${byline.occupations @ join=', '}</p>
</div>
<!-- OR, when hasContent is false: editor placeholder via core/wcm/components/commons/v1/templates.html -->
```

### Properties Used
- `byline.name` → `<h2 class="cmp-byline__name">`
- `byline.occupations` → joined with `', '` → `<p class="cmp-byline__occupations">`
- `byline.empty` (negated as `hasContent`) → gates rendering vs. author-mode placeholder
- Nested `data-sly-resource` include renders the delegated image at `core/wcm/components/image/v2/image` (note: v2, while the component's own resourceSuperType is v3 — used only for the dialog/model, not the child image renderer)

### Rendering Notes
- Root element is always `<div>`, not semantic `<article>`/`<figure>`.
- No conditional variant classes — single fixed visual style (no modifiers).

## CSS Classes & Design Tokens

### BEM Structure
```
.cmp-byline (block)
├── .cmp-byline__image (element)
├── .cmp-byline__name (element)
└── .cmp-byline__occupations (element)
```
(Also styles the nested Core Image's `.cmp-image__image` element directly — a cross-component CSS dependency.)

### CSS Class Reference
| Class | Type | Purpose | Design Tokens / Values |
|---|---|---|---|
| `cmp-byline` | block | Container, defines local `$imageSize: 60px` | - |
| `cmp-byline__image` | element | Floats avatar left | `$imageSize` (60px) |
| `cmp-byline__image .cmp-image__image` | cross-component override | Makes nested Core Image circular | `border-radius: 30px`, `object-fit: cover` |
| `cmp-byline__name` | element | Contributor name | `$font-size-large`, `$font-family-serif` |
| `cmp-byline__occupations` | element | Occupations list | `$gray`, `$font-size-xsmall`, `text-transform: uppercase` |

### Design Tokens / SCSS Variables Used
- `$font-size-large`, `$font-family-serif`, `$font-size-xsmall`, `$gray` (from shared SCSS variables, not inspected directly — referenced but not locally defined in this file)
- Local SCSS variable: `$imageSize: 60px`

⚠️ No responsive (`@media`) rules or pseudo-states (`:hover`, `:focus`) defined for this component.

## Sling Model API Export
No `ComponentExporter`/`@Exporter` annotation present — `Byline` model is **not** exposed as a JSON export endpoint (no `.model.json` support). It is a request-scoped view model only, used purely for HTL rendering.

## Accessibility Requirements
- **Heading structure**: Uses `<h2>` for the name — assumes correct heading hierarchy context on the page (not verified here — manual review required).
- **Image alt text**: Delegated entirely to the nested Core Components Image component's own alt-text handling (`alt`, `altValueFromDAM`) — Byline does not add its own `aria-label`.
- **Empty state**: When incomplete (`isEmpty() == true`), renders an editor-only placeholder rather than a broken/partial component in author mode — but this placeholder does not render on publish (component is simply omitted), which is correct for real content.

## Current Constraints & Limitations
- No visual variants (no modifier classes) — single fixed look.
- Occupation sort order is hardcoded ascending; not author-configurable.
- Image size is hardcoded at 60px via SCSS; not configurable via dialog.
- Component is entirely empty (renders nothing on publish) unless name, ≥1 occupation, AND a valid image are all present — no partial rendering supported.
- Tight coupling to Core Components Image internals (targets `.cmp-image__image` directly in CSS) — upgrading the Core Image component's DOM/class structure could silently break Byline's avatar styling.

## Related Components
- `core/wcm/components/image/v3/image` (resourceSuperType, dialog authoring tab)
- `core/wcm/components/image/v2/image` (used for the actual nested image render in HTL — version mismatch vs. dialog's v3, worth confirming intentional)

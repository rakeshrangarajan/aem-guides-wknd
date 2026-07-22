# Image List Component: Technical Design

## Component Identity
- **Resource Type**: `wknd/components/image-list`
- **File Location**: `ui.apps/src/main/content/jcr_root/apps/wknd/components/image-list/`
- **Sling Model**: `com.adobe.aem.guides.wknd.core.models.ImageList` (interface) / `ImageListImpl` (impl, in `core/src/main/java/com/adobe/aem/guides/wknd/core/models/`)
- **Purpose**: Renders a list of pages (delegated to Core Components List) as a grid/list of image cards, where each item's image is looked up from a WKND Image component embedded on the target page (rather than a DAM asset directly attached to the list item). *(Inferred from code — no explicit dialog description.)*
- **Resource Super Type**: `core/wcm/components/list/v3/list` (delegation pattern — wraps and transforms the Core List Sling Model's output)
- **Status**: Production Ready (has unit tests: `ImageListImplTest.java`)

## Sling Model Configuration Properties

Image List has **no `@ValueMapValue` fields of its own** — all list-configuration properties (source pages, sorting, tags, etc.) are inherited/authored via the Core Components List's own dialog and read through delegation, not reimplemented here.

| Property | Type | Required | Default | Constraint | Source |
|---|---|---|---|---|---|
| *(all List config, e.g. source, tags, sortOrder)* | - | - | - | - | Inherited entirely from `core/wcm/components/list/v3/list` — dialog fully hidden/overridden locally (see below) |

**Computed / derived properties (`ImageListImpl`):**
- `getListItems()` — `Collection<ImageList.ListItem>`. Delegates to the injected Core List model (`@Self @Via(type = ResourceSuperType.class) List coreList`), maps each `ListItem` into a custom `ImageListItemImpl`, then **filters out any item where `isEmpty()` is true** (i.e., no matching Image component found on the target page). Result is cached in `imageListItems` after first computation.
- `isEmpty()` — `true` if the (filtered) list of items is empty.
- `getId()` — component instance ID via `ComponentUtils.getId(...)`, used for the data layer.
- `getData()` — returns a `ComponentData` (data layer JSON) only if data layer tracking is enabled on the resource; otherwise `null`.

**`ImageList.ListItem` (nested model, per item):**
| Property | Type | Description |
|---|---|---|
| `getImage()` | `Resource` | The first resource of type `wknd/components/image` found on the item's target page via a `QueryBuilder` search (path-scoped, limited to 1 result, ordered by JCR path ascending). Wrapped in `SimpleImageComponentResource` to force decorative/caption flags off. |
| `getTitle()` | String | Delegated from the wrapped Core `ListItem.getTitle()` |
| `getDescription()` | String | Page property `shortDescription`, falling back to the page's standard description (`page.getDescription()`) |
| `getURL()` | String | Delegated from Core `ListItem.getURL()` |
| `getId()` | String | Generated via `ComponentUtils.generateId(parentId + "-image-list-item", URL)` |
| `getData()` | `ComponentData` | Data layer entry (id, type, title, description, link URL, parent ID) if data layer is enabled on the found image resource |
| `isEmpty()` | boolean | `true` if no matching image resource was found on the page |

## Author Dialog Fields

The local `_cq_dialog.xml` does **not** define new fields — it overrides/hides parts of the inherited Core List dialog:

| Override | Effect |
|---|---|
| `itemSettings` tab | `sling:hideResource="true"`, `sling:hideChildren="*"` — fully hidden |
| `extraClientlibs` | `[core.wcm.components.list.v2.editor]` — reuses Core List's v2 editor clientlib |
| `trackingFeature` | `wknd:image-list` |

All actual configurable fields (source, tags, sort, limit, link items, etc.) come from the inherited `core/wcm/components/list/v3/list` dialog, unmodified except for the hidden `itemSettings` tab. **Not independently documented here** — refer to Core Components List documentation for the full field set.

## HTL Template Rendering

### Templates
- `image-list.html` (root)
- `item.html` (per-item template, `data-sly-template.item`)

### HTML Output
```html
<ul class="cmp-image-list" data-cmp-data-layer="${imageList.data.json}">
    <li class="cmp-image-list__item" data-cmp-data-layer="${item.data.json}">
        <article class="cmp-image-list__item-content">
            <a class="cmp-image-list__item-image-link" href="${item.URL}" data-cmp-clickable="...">
                <div class="cmp-image-list__item-image"><!-- item.image resource, wcmmode=disabled --></div>
            </a>
            <a class="cmp-image-list__item-title-link" href="${item.URL}" data-cmp-clickable="...">
                <span class="cmp-image-list__item-title">${item.title}</span>
            </a>
            <span class="cmp-image-list__item-description">${item.description}</span> <!-- only if present -->
        </article>
    </li>
    <!-- ...repeated per item... -->
</ul>
<!-- OR editor placeholder if hasContent is false -->
```

### Properties Used
- `imageList.listItems` (drives the `data-sly-list`)
- `imageList.data.json` (data layer, root `<ul>`)
- Per item: `item.URL`, `item.image`, `item.title`, `item.description`, `item.data.json`
- `imageList.empty` (negated as `hasContent`) gates the editor placeholder

### Rendering Notes
- Root element is `<ul>`/`<li>` — semantic list markup (unlike Byline's generic `<div>`).
- The item's image `<div>` is populated via `data-sly-resource="${item.image @ wcmmode='disabled'}"` — renders the WKND Image component's own HTL, so Image's own CSS classes/markup apply inside `.cmp-image-list__item-image`.
- The image link (`<a>`) itself is conditionally rendered only `data-sly-test="${item.image}"` — if no image is found, no image link renders, but title/description links still do.

## CSS Classes & Design Tokens

### BEM Structure
```
.cmp-image-list (block)
├── .cmp-image-list__item (element)
├── .cmp-image-list__item-content (element)
├── .cmp-image-list__item-image-link (element)
├── .cmp-image-list__item-image (element)
├── .cmp-image-list__item-title-link (element)
├── .cmp-image-list__item-title (element)
└── .cmp-image-list__item-description (element)
```

### CSS Class Reference
The local SCSS entry point (`image-list.scss`) only does `@import 'styles/default'` — the actual `_default.scss` partial was **not located/inspected** (only `image-list.scss` and the SCSS folder for `byline`/`helloworld` were confirmed; the `image-list` default partial file was not found in the file listing, so styling rules are **not verified** here — manual review recommended).

⚠️ **Gap**: Unlike Byline and Hello World, the underlying `styles/_default.scss` for image-list was not available to confirm actual class-level styling, design tokens, or responsive breakpoints. Treat the CSS section of this spec as incomplete.

## Sling Model API Export
No `ComponentExporter`/`@Exporter` annotation on `ImageList` — not exposed via `.model.json`. Render-only model (though it does implement a `getData()` for the WCM Core Components **data layer**, which is a separate JSON-LD-style tracking mechanism, not a REST export).

## Accessibility Requirements
- Semantic `<ul>`/`<li>`/`<article>` structure — good baseline semantics.
- Title and image are both wrapped in `<a>` tags pointing to the same `item.URL` — potential **duplicate-link accessibility concern** (two adjacent links to the same destination); not mitigated with `aria-hidden` or a combined single-link pattern. Flagging for manual review.
- Image accessibility (alt text) is delegated to the nested WKND Image component's own rendering — `SimpleImageComponentResource` explicitly sets `alt` to the item's title and disables "decorative" and DAM-sourced alt/title flags, ensuring a real alt text is always supplied.
- No explicit `aria-label` on the `<ul>` itself describing the list's purpose.

## Current Constraints & Limitations
- Only supports images sourced from a `wknd/components/image` component embedded on the **first responsive-grid level** of each target page (via `QueryBuilder`, `limit=1`) — cannot use a DAM asset directly, and only finds the *first* matching Image component per page.
- Items without a discoverable Image component are silently filtered out of the rendered list (`isEmpty()` filter in `getListItems()`) — authors get no visible warning if a page lacks an Image component; the item simply won't appear.
- `QueryBuilder` search runs per page look-up; for large source sets this could be a performance consideration (a JCR query per list item) — not optimized/batched.
- No author-configurable image size/crop specific to Image List — relies entirely on the embedded Image component + its page-level policy.
- Local dialog does not expose or document the full inherited Core List config surface (source, sorting, tag filters) — authors must rely on Core Components documentation to understand available options.
- CSS/design-token details for this component are **unverified** (see Gap above).

## Related Components
- `core/wcm/components/list/v3/list` (resourceSuperType — delegation pattern via `@Via(type = ResourceSuperType.class)`)
- `wknd/components/image` (child image component whose resource is located and rendered per list item)
- `core/wcm/components/commons/v1/templates.html` (editor placeholder template)

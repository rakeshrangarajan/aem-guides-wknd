---
name: aem-spec-extractor
description: Extract technical specifications from existing AEM Headful components. Use this skill whenever analyzing a brownfield AEM component to create tech-design.md documentation. Triggers when user mentions "extract spec from component", "analyze AEM component", "create tech-design", "document existing component", or provides component files (ButtonModel.java, _cq_dialog.xml, button.html, button.css). The skill reads the component's Sling Model, dialog definition, HTL template, and CSS to synthesize a complete technical specification document.
---

# AEM Spec Extractor

## Purpose

Extract a comprehensive technical specification (tech-design.md) from existing brownfield AEM Headful component code. This enables spec-driven development by creating a single source of truth from scattered implementation artifacts.

## When to Use

- **Brownfield component analysis**: Documenting an undocumented existing component
- **Spec extraction phase**: Phase 1 of the spec-driven workflow
- **Component inventory**: Building specifications for all existing components
- **Requirements baseline**: Understanding what a component currently does before planning changes

**Do NOT use** for creating new component documentation from scratch (use design documents instead).

---

## Inputs

Provide access to these 4 component files:

```
ui.apps/apps/myapp/components/COMPONENT/v1/
├── ComponentModel.java          (Sling Model with @ValueMapValue fields)
├── _cq_dialog.xml              (Author dialog definition)
├── component.html              (HTL template)
└── component.css               (Component styles)
```

**Minimum**: ComponentModel.java (Sling Model) — others are optional but increase accuracy

**Helpful**: Also provide related files:
- `_cq_editConfig.xml` (edit behavior)
- Design token definitions (if available)
- README.md (any existing documentation)

---

## Extraction Process

### Step 1: Analyze Sling Model (Java)

Extract configuration properties by parsing:

```java
@Model(adaptables = Resource.class, resourceType = "myapp/components/button/v1")
public class ButtonModel implements ComponentExporter {
  
  @ValueMapValue
  private String text;                    // ← Extract: name, type, required
  
  @ValueMapValue
  private String link;
  
  @ValueMapValue
  private String target = "_self";        // ← Extract: default value
  
  @PostConstruct
  protected void init() { ... }           // ← Extract: initialization logic
  
  public String getText() { ... }         // ← Extract: getter methods
}
```

**What to extract:**
- Field name: `text`, `link`, `target`
- Field type: `String`, `Integer`, `boolean`
- Is required: Infer from `@ValueMapValue` presence + dialog
- Default value: From field initializer or dialog
- Getter/setter methods
- `@PostConstruct` logic
- Computed properties (properties not in @ValueMapValue but derived)

### Step 2: Analyze Dialog XML

Extract author configurable fields:

```xml
<items jcr:primaryType="nt:unstructured">
  <text sling:resourceType=".../textfield"
        fieldLabel="Button Text"
        name="./text"
        required="{Boolean}true"
        maxlength="100"/>
  
  <link sling:resourceType=".../pathfield"
        fieldLabel="Link"
        name="./link"
        rootPath="/content"/>
  
  <target sling:resourceType=".../select"
          name="./target"
          value="_self">
    <items>
      <self value="_self" text="Same Window"/>
      <blank value="_blank" text="New Window"/>
    </items>
  </target>
</items>
```

**What to extract:**
- Field name: `./text` → property name `text`
- Field label: Human-readable name
- Field type: textfield, select, pathfield, etc.
- Required: `required="{Boolean}true"`
- Validation: `maxlength`, `min`, `max`, etc.
- Default value: `value="..."`
- Enum options: For select/radio fields
- Field grouping: Tab structure, sections

### Step 3: Analyze HTL Template

Extract rendering behavior:

```html
<button class="button button--${properties.variant @ default:'solid'}"
        href="${properties.link}"
        target="${properties.target}"
        aria-label="${properties.ariaLabel}">
  <span class="button__label">${properties.text}</span>
</button>
```

**What to extract:**
- Which properties are used: `text`, `link`, `target`, `variant`, `ariaLabel`
- CSS classes applied: `button`, `button--${variant}` (dynamic)
- HTML elements: `<button>`, `<span>` (semantic structure)
- Conditional logic: `@ default:`, `@if`, `@use` directives
- Attributes: `href`, `target`, `aria-label`
- Does it render as `<button>` or `<a>`?

### Step 4: Analyze CSS

Extract available styles:

```css
.button { 
  padding: var(--spacing-md); 
  font-weight: bold;
}

.button__label { 
  color: var(--color-text); 
}

.button--solid { 
  background: var(--color-primary); 
}

.button--outline { 
  border: 2px solid var(--color-primary); 
}

.button--primary { 
  background: var(--color-primary); 
}

.button--danger { 
  background: var(--color-danger); 
}
```

**What to extract:**
- Base class: `.button` (block)
- Element classes: `.button__label`, `.button__icon` (elements)
- Modifier classes: `.button--solid`, `.button--outline` (variants)
- Design tokens used: `--spacing-md`, `--color-primary`, `--color-danger`
- Responsive rules: `@media` breakpoints
- Pseudo-states: `:hover`, `:focus`, `:disabled`

### Step 5: Synthesize into tech-design.md

Combine all four sources into a unified specification:

```markdown
# [Component Name]: Technical Design

## Component Identity
- Resource Type: myapp/components/button/v1
- Purpose: [From dialog/code/comments]

## Sling Model Configuration Properties
[Table of all @ValueMapValue fields]

## Author Dialog Fields  
[Table of all configurable fields with constraints]

## HTL Template Rendering
[How component renders, which properties used, HTML output]

## CSS Classes & BEM Structure
[All CSS classes, design tokens, responsive behavior]

## Sling Model API Export
[JSON schema of .model.json endpoint]

## Accessibility Requirements
[WCAG compliance, ARIA requirements]

## Design Tokens Used
[All --token references]

## Current Constraints & Limitations
[What component can't do, edge cases]
```

---

## Output Format

### tech-design.md Structure

```markdown
# [Component Name] Component: Technical Design

## Component Identity
- **Resource Type**: myapp/components/[component]/v1
- **File Location**: ui.apps/apps/myapp/components/[component]/v1/
- **Purpose**: [One sentence description]
- **Current Version**: v1
- **Status**: Production Ready

## Sling Model Configuration Properties

| Property | Type | Required | Default | Constraint | Source |
|----------|------|----------|---------|-----------|--------|
| text | String | YES | - | max 100 chars | @ValueMapValue |
| link | String | NO | - | valid path/URL | @ValueMapValue |
| target | String | NO | "_self" | enum: _self, _blank | @ValueMapValue |
| ariaLabel | String | NO | - | icon-only required | @ValueMapValue |

## Author Dialog Fields

| Field Label | Field Name | Type | Required | Default | Validation |
|-------------|-----------|------|----------|---------|-----------|
| Button Text | text | textfield | YES | - | maxlength: 100 |
| Link | link | pathfield | NO | - | rootPath: /content |
| Target | target | select | NO | _self | options: _self, _blank, _parent |
| Aria Label | ariaLabel | textfield | NO | - | - |

## HTL Template Rendering

### HTML Output
```html
<button class="button button--${properties.variant @ default:'solid'}"
        href="${properties.link}"
        target="${properties.target}"
        aria-label="${properties.ariaLabel}">
  <span class="button__label">${properties.text}</span>
</button>
```

### Properties Used
- Required: `text`, `link`
- Optional: `target`, `ariaLabel`, `variant`
- Dynamic CSS class: `button--${variant}`

## CSS Classes & Design Tokens

### BEM Structure
```
.button (block)
├── .button__label (element)
├── .button__icon (element)
├── .button--solid (modifier - filled style)
├── .button--outline (modifier - border style)
├── .button--primary (modifier - color)
├── .button--secondary (modifier - color)
└── .button--danger (modifier - color)
```

### CSS Class Reference
| Class | Type | Purpose | Design Tokens |
|-------|------|---------|--------|
| button | block | Base component | --spacing-md, --font-weight-bold |
| button__label | element | Text content | --color-text |
| button__icon | element | Icon wrapper | - |
| button--solid | modifier | Filled background | --color-primary |
| button--outline | modifier | Border style | --color-primary |
| button--primary | modifier | Primary color | --color-primary |
| button--danger | modifier | Danger color | --color-danger |

### Design Tokens Used
- `--spacing-md` (padding)
- `--color-primary` (primary color)
- `--color-danger` (danger/destructive color)
- `--color-text` (text color)
- `--font-weight-bold` (font weight)

## Sling Model API Export

### Endpoint
```
GET /content/{page}/jcr:content/button.model.json
```

### JSON Schema
```json
{
  "text": "String",
  "link": "String",
  "target": "String",
  "ariaLabel": "String",
  "exportedType": "myapp/components/button/v1"
}
```

### Example Response
```json
{
  "text": "Click me",
  "link": "/content/page",
  "target": "_blank",
  "ariaLabel": "Submit form",
  "exportedType": "myapp/components/button/v1"
}
```

## Accessibility Requirements
- **WCAG Level**: AA (2.1)
- **Keyboard Support**: Native HTML button behavior (Enter/Space)
- **ARIA**: Icon-only buttons require `ariaLabel`
- **Focus**: Visible focus outline (3px minimum)
- **Contrast**: 4.5:1 minimum for text

## Current Constraints & Limitations
- Icon support: NO (no icon field defined)
- Size variants: NO (fixed padding)
- HTML element: Always `<button>` (no `<a>` fallback)
- Customizable styling: NO (styles hardcoded)

## Related Components
- (List other components that depend on or are similar to this one)
```

---

## Instructions for User

### To Extract a Specification:

1. **Prepare component files**
   ```bash
   # Gather these files in one place:
   # - ComponentModel.java
   # - _cq_dialog.xml
   # - component.html
   # - component.css (optional but helpful)
   ```

2. **Provide to Claude**
   - Upload files to this conversation
   - Or paste content in message
   - Include component name and context

3. **Ask Claude**
   ```
   "Extract the technical specification for the Button component.
    Create tech-design.md that captures:
    - All Sling Model properties
    - Dialog field definitions
    - HTL rendering logic
    - CSS classes and design tokens
    - API export schema
    - Accessibility requirements
    
    Save as: specs/001-button-component/tech-design.md"
   ```

4. **Review output**
   - Verify all properties extracted
   - Check constraints captured
   - Confirm CSS classes listed
   - Validate API schema

5. **Approve & commit**
   ```bash
   git add specs/001-button-component/tech-design.md
   git commit -m "docs: Extract Button component specification"
   ```

---

## Accuracy Guidelines

### High Confidence Extraction
✅ Sling Model `@ValueMapValue` fields (directly in code)
✅ Dialog field definitions (directly in XML)
✅ CSS class names (directly in CSS file)
✅ Design token references (--token-name format)
✅ HTL property usage (${properties.xxx})

### Medium Confidence (Requires Verification)
⚠️ Purpose/description (infer from code + comments)
⚠️ Constraints (infer from dialog validation rules)
⚠️ Accessibility features (infer from ARIA attributes)
⚠️ Current limitations (infer from what's NOT in code)

### Manual Review Required
❓ Business intent (why component exists)
❓ Usage patterns (which templates use this)
❓ Performance considerations
❓ Migration notes (if component is deprecated)

**Always note**: "Inferred from code structure" vs "Explicitly defined"

---

## Example: Button Component Extraction

### Input Files Provided:
- ButtonModel.java (Sling Model)
- _cq_dialog.xml (Author dialog)
- button.html (HTL template)
- button.css (Styles)

### Extraction Process:
1. Parse ButtonModel.java → 4 @ValueMapValue fields
2. Parse _cq_dialog.xml → 4 form fields with validation
3. Parse button.html → CSS class mapping and HTML output
4. Parse button.css → 7 CSS classes, 5 design tokens

### Output:
tech-design.md with all extracted details, organized by section, ready for developer review and approval.

---

## Next Steps in Workflow

After tech-design.md is extracted and approved:

1. **Phase 2**: Use extracted spec to generate tasks.md
2. **Phase 3**: Execute tasks.md using Claude Code
3. **Phase 4**: Code review and merge

This skill is **Phase 1** of the self-contained spec-driven workflow.

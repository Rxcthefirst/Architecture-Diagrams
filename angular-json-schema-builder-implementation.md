# Angular JSON Schema Builder – Implementation Plan

## 1. Purpose
This document defines the Angular implementation plan for an **interactive, guided JSON Schema authoring UI**. The target user has little to no knowledge of JSON Schema. The system must walk users through schema construction using guardrails, sensible defaults, and reusable types, while generating valid JSON Schema (Draft 2020-12) behind the scenes.

The UI is **not** intended to generate forms for end users. It is a **schema modeling tool**.

---

## 2. Core UX Principles

1. **No raw JSON required**
   - Users interact with domain concepts (fields, types, rules)
   - JSON Schema is generated automatically

2. **Tree-first modeling**
   - Schema is visualized as a hierarchical structure
   - Editing happens through guided panels, not free-form JSON

3. **Progressive disclosure**
   - Beginners see only relevant options
   - Advanced concepts ($ref, allOf, oneOf) are opt-in

4. **Single source of truth**
   - Internal Schema AST is authoritative
   - JSON Schema is a generated artifact

---

## 3. High-Level Architecture

```
Angular UI
 ├─ Schema AST (in-memory state)
 │   ├─ Root schema
 │   ├─ $defs (reusable types)
 │   └─ Field nodes / Ref nodes
 │
 ├─ Tree Renderer (read-only view of AST)
 ├─ Wizard / Editor Panels (mutate AST)
 ├─ Validation Engine (AJV)
 └─ Export Pipeline (AST → JSON Schema)
```

State is managed centrally (signals or NgRx) to support undo/redo and diffing later.

---

## 4. Core Components

### 4.1 SchemaBuilderShellComponent
**Responsibility**
- Page layout
- Coordinates tree, editor panel, and type library
- Hosts command bar (save, export, validate)

**Layout (3-panel)**
- Left: Schema Tree
- Center/Right: Node Wizard / Properties Panel
- Optional Bottom/Side: Help + Examples

---

### 4.2 SchemaTreeComponent
**Responsibility**
- Render schema hierarchy
- Provide navigation and context actions

**Implementation**
- Angular Material `mat-tree`
- Node types rendered differently:
  - Object
  - Array
  - Scalar
  - `$ref` (link icon)

**Features**
- Expand/collapse nodes
- Context menu:
  - Add field
  - Add object
  - Add array
  - Promote to reusable type
  - Delete
- Drag & drop reordering (CDK)
- Cycle-safe expansion for `$ref`

---

### 4.3 NodeWizardComponent
**Responsibility**
- Guided editing of a selected node

**Wizard Steps**
1. Identity
   - Field name
   - Title
   - Description

2. Type Selection
   - string | number | integer | boolean | object | array | null
   - Inline vs reusable (for object/array)

3. Constraints (type-specific)
   - Strings: minLength, maxLength, pattern, format, enum
   - Numbers: minimum, maximum, multipleOf
   - Arrays: items type, minItems, maxItems, uniqueItems
   - Objects: required fields, additionalProperties

4. Examples
   - Example value(s)
   - Live validation feedback

All options shown are filtered based on the node type.

---

### 4.4 TypesLibraryComponent ($defs Manager)
**Responsibility**
- Manage reusable schema definitions

**Features**
- List all reusable types
- Create / edit reusable type (same wizard as inline object)
- Rename type (with ref updates)
- Show usage count (impact awareness)
- Navigate to definition

**UX Language**
- "Reusable Types" (not "$defs")

---

### 4.5 RefNode UX ($ref Handling)

**Display Rules**
- Icon: link / chain
- Label: `fieldName → TypeName`
- Expand shows referenced structure (read-only)
- Action: "Go to definition"

**Creation Paths**
- Use reusable type when creating field
- Promote inline object to reusable type

**Internal Representation**
- RefNode contains:
  - refTargetId
  - displayName
  - optional localConstraints

---

## 5. Internal Schema AST Model

### 5.1 Core Interfaces (Conceptual)

- SchemaRoot
  - version
  - rootNode
  - definitions: Map<DefId, SchemaNode>

- SchemaNode
  - id
  - name
  - kind: object | array | scalar | ref
  - required
  - constraints

- ObjectNode extends SchemaNode
  - properties: SchemaNode[]

- ArrayNode extends SchemaNode
  - items: SchemaNode

- RefNode extends SchemaNode
  - refTargetId
  - localConstraints (optional)

AST is normalized and acyclic except via RefNode references.

---

## 6. $ref Design Strategy

### 6.1 Promote Inline to Reusable

Flow:
1. User selects inline object
2. Action: "Promote to reusable type"
3. User names type
4. Node moved to `$defs`
5. Original node replaced with RefNode

### 6.2 Overrides on Ref (Advanced)

UX:
- "Additional constraints" accordion

Schema Output:
```
allOf:
  - $ref
  - local constraints
```

Hidden unless user explicitly enables advanced mode.

### 6.3 Cycles

- Detect cycles via visited set during tree render
- UI behavior:
  - Prevent infinite expansion
  - Show "…cycle detected" indicator

---

## 7. Validation Strategy

### 7.1 Client-side
- AJV validates generated JSON Schema
- AJV validates example JSON instances
- Errors mapped back to UI-friendly messages

### 7.2 Structural Guardrails
- Required fields stored at parent object level
- Disallow invalid constraint/type combinations
- Enforce unique property names

---

## 8. Export Pipeline

**AST → JSON Schema (Draft 2020-12)**

Steps:
1. Traverse AST
2. Emit `$schema` header
3. Emit `$defs` from Types Library
4. Emit root properties
5. Normalize required lists
6. Resolve RefNodes to `$ref`

Optional:
- Bundle external schemas
- Pretty-print / minify

---

## 9. Import (Optional, Phase 2)

- Parse JSON Schema
- Normalize `definitions` → `$defs`
- Build AST
- Mark unsupported constructs as read-only

---

## 10. State Management

Recommended:
- Angular Signals (initial)
- Command pattern for mutations
- Enable undo/redo later

Every edit operation is a command:
- AddNode
- RemoveNode
- UpdateConstraint
- PromoteToDef

---

## 11. MVP Feature Checklist

- Root object schema
- Add/edit/remove fields
- Type selection + constraints
- Required toggle
- Reusable Types ($defs)
- $ref usage + navigation
- Promote inline → reusable
- Export JSON Schema
- Client-side validation

---

## 12. Future Enhancements

- oneOf / anyOf / allOf UI
- External schema libraries
- Impact analysis for changes
- Version diff visualization
- Schema templates
- Accessibility + localization

---

## 13. Summary

This design treats JSON Schema as a **modeling language**, not a text format. The Angular UI guides users through structured decisions, enforces correctness, and introduces advanced concepts like `$ref` only when needed. The result is a schema authoring experience that is powerful, safe, and accessible to non-experts.

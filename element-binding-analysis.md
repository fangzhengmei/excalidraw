# Element Binding & Arrow Connection Update Analysis

## 1. Core Data Structures

### 1.1 Binding Types

**FixedPointBinding** (`types.ts:284-297`)
```typescript
type FixedPointBinding = {
  elementId: ExcalidrawBindableElement["id"];
  fixedPoint: FixedPoint;      // [x, y] ratio 0.0-1.0
  mode: BindMode;               // "inside" | "orbit" | "skip"
};
```

**BindMode Semantics**:
- **`inside`**: Arrow endpoint goes all the way inside to the fixed point
- **`orbit`**: Arrow endpoint stays outside on the element's outline
- **`skip`**: Special mode for free binding behavior

### 1.2 Element Binding State

**ExcalidrawArrowElement** (`types.ts:355-359`):
```typescript
type ExcalidrawArrowElement = {
  startBinding: FixedPointBinding | null;
  endBinding: FixedPointBinding | null;
  // ...
};
```

**BindableElement** (`types.ts:35-38, 76`):
```typescript
type BoundElement = {
  id: string;
  type: "arrow" | "text";
};

type ExcalidrawBindableElement = {
  boundElements: readonly BoundElement[] | null;
  // ...
};
```

---

## 2. State Change Flow

### 2.1 Primary Entry Point: Arrow Point Dragging

**Flow**: `maybeHandleArrowPointlikeDrag` → `handlePointDragging` → `bindOrUnbindBindingElement`

```
packages/element/src/arrows/helpers.ts:7-44
  ↓
packages/element/src/linearElementEditor.ts:handlePointDragging
  ↓
packages/element/src/binding.ts:145-221: bindOrUnbindBindingElement()
```

### 2.2 bindOrUnbindBindingElement Process

```typescript
bindOrUnbindBindingElement(
  arrow: ExcalidrawArrowElement,
  draggingPoints: PointsPositionUpdates,
  scenePointerX/Y: number,
  scene: Scene,
  appState: AppState,
  opts?: { newArrow?, altKey?, angleLocked?, initialBinding? }
)
```

**Step 1: Determine Binding Strategy** (`binding.ts:581-619`)

- **Simple mode**: `getBindingStrategyForDraggingBindingElementEndpoints_simple`
- **Complex mode**: `getBindingStrategyForDraggingBindingElementEndpoints_complex` (feature flag)

**Strategy Output**:
```typescript
type BindingStrategy = {
  mode: "inside" | "orbit" | "skip" | null | undefined;
  element?: ExcalidrawBindableElement;
  focusPoint?: GlobalPoint;  // Only if mode is defined
}
```

**Step 2: Apply Strategy to Both Ends** (`binding.ts:224-245`)

```typescript
bindOrUnbindBindingElementEdge(arrow, strategy, "start" | "end", scene)
```

- **`mode = null`**: Call `unbindBindingElement()` → break binding
- **`mode = undefined`**: Do nothing (keep existing binding)
- **`mode = inside/orbit`**: Call `bindBindingElement()` → create/update binding

**Step 3: Update Arrow Points** (`binding.ts:187-219`)

If focusPoint exists in strategy, update arrow points via:
```typescript
LinearElementEditor.movePoints(arrow, scene, updates)
```

---

## 3. Strategy Determination Logic

### 3.1 Simple Mode Strategy Decision Tree (`binding.ts:621-869`)

```
Conditions Checked:
├─ 1. Both ends dragged? → {start: null, end: null} (break all bindings)
├─ 2. Binding disabled?   → Break bindings for dragged endpoint only
├─ 3. Elbow arrow?        → Use bindingStrategyForElbowArrowEndpointDragging
└─ 4. Normal cases:
   ├─ A. Alt key pressed? → Force inside binding if hovering element
   ├─ B. Same element binding? → Inside-inside binding
   └─ C. Normal binding:
      ├─ Point inside element?  → "inside" mode
      └─ Point outside element? → "orbit" mode (project to diagonal)
```

### 3.2 Complex Mode Strategy (`binding.ts:871-997`)

Additional considerations:
- Global bind mode preference
- Nested element handling
- Transparency checks for overlapping elements
- Multi-point arrow handling

---

## 4. Core Binding Operations

### 4.1 bindBindingElement (`binding.ts:1016-1068`)

**Purpose**: Create or update a binding between arrow and element

```typescript
bindBindingElement(
  arrow: ExcalidrawArrowElement,
  hoveredElement: ExcalidrawBindableElement,
  mode: BindMode,
  startOrEnd: "start" | "end",
  scene: Scene,
  focusPoint?: GlobalPoint,
  shouldSnapToOutline = true
)
```

**Operations**:
1. **Calculate Fixed Point**:
   - Elbow arrows: `calculateFixedPointForElbowArrowBinding()`
   - Normal arrows: `calculateFixedPointForNonElbowArrowBinding()`

2. **Update Arrow State**:
   ```typescript
   scene.mutateElement(arrow, {
     [startOrEnd === "start" ? "startBinding" : "endBinding"]: binding
   });
   ```

3. **Update Bound Element Reference**:
   ```typescript
   // Add arrow to hoveredElement.boundElements if not present
   scene.mutateElement(hoveredElement, {
     boundElements: [...existing, { id: arrow.id, type: "arrow" }]
   });
   ```

### 4.2 unbindBindingElement (`binding.ts:1070-1100`)

**Purpose**: Remove a binding from an arrow endpoint

```typescript
unbindBindingElement(
  arrow: ExcalidrawArrowElement,
  startOrEnd: "start" | "end",
  scene: Scene
): elementId | null
```

**Operations**:
1. **Check Opposite Binding**: If other end bound to same element, DON'T remove from boundElements
2. **Clean Up Bound Element**: Remove arrow reference from bindableElement.boundElements
3. **Update Arrow State**: Set startBinding/endBinding to null

---

## 5. Bound Element Updates (Element Movement)

### 5.1 updateBoundElements (`binding.ts:1104-1204`)

**Trigger**: When a bindable element is translated/rotated/scaled

**Flow**:
```
Element moved
  ↓
updateBoundElements(changedElement, scene)
  ↓
For each bound arrow in changedElement.boundElements:
  ↓
  Check if arrow needs update (doesNeedUpdate)
  ↓
  Skip if arrow is being simultaneously updated
  ↓
  Calculate new arrow endpoint positions via updateBoundPoint()
  ↓
  LinearElementEditor.movePoints() to update arrow geometry
```

### 5.2 updateBoundPoint (`binding.ts:1746-1902`)

**Purpose**: Recalculate arrow endpoint when bound element moves

**4-Step Decision Logic**:

1. **Inside Binding Short-circuit**: If mode = "inside", use focusPoint directly
2. **Outline Inside Other Shape**: If outline point would be inside the other bound shape, use focusPoint to avoid inversion
3. **Arrow Too Short (Unconnected)**: If other end unbound and arrow too short, use focusPoint
4. **Arrow Too Short (Connected)**: If both ends bound but arrow too short, use resolved target
5. **General Case**: Snap to outline if possible

---

## 6. Exception Boundaries & Error Handling

### 6.1 invariant Checks (Hard Failures)

**Location**: Throughout `binding.ts`

```typescript
// Elbow arrow single point constraint
invariant(draggingPoints.size === 1, "Bound elbow arrows cannot be moved")

// Point existence
invariant(update, "There should be a position update for dragging an elbow arrow endpoint")

// New arrow creation path
invariant(false, "New arrow creation should not reach here")

// Single point elements
invariant(arrow.points.length > 1, "Do not attempt to bind linear elements with a single point")
```

**Impact**: These throw errors and stop execution - **critical boundaries**

### 6.2 Graceful Degradation (Soft Failures)

**Null Handling Patterns**:

```typescript
// binding.ts:1129-1137
const visitor = (element: ExcalidrawElement | undefined) => {
  if (!isArrowElement(element) || element.isDeleted) {
    return;  // Skip invalid/deleted elements
  }
  if (!doesNeedUpdate(element, changedElement)) {
    return;  // Skip if update not needed
  }
  // ...
};
```

**Safe Map Access**:
```typescript
// binding.ts:1140-1148
const startBindingElement = element.startBinding
  ? elementsMap.get(element.startBinding.elementId)
  : null;
```

**Precision Guards**:
```typescript
// binding.ts:1894-1901
// Avoid division by zero
Math.max(hoveredElement.width, PRECISION)
```

---

## 7. Edge Cases & Special Handling

### 7.1 Simultaneous Updates Detection

```typescript
// binding.ts:1150-1153
if (simultaneouslyUpdatedElementIds.has(element.id)) {
  return;  // Skip - arrow is already being moved
}
```

**Purpose**: Prevent infinite update loops when both element and arrow are selected

### 7.2 Duplication Handling

**fixDuplicatedBindingsAfterDuplication** (`binding.ts:1989-2059`):
- Remap all binding IDs from original to duplicated elements
- Update `boundElements`, `containerId`, `startBinding`, `endBinding`
- Recompute elbow arrow points after duplication

### 7.3 Deletion Cleanup

**fixBindingsAfterDeletion** (`binding.ts:2061-2075`):
```typescript
for (const element of deletedElements) {
  BoundElement.unbindAffected(elements, element, mutateFn);
  BindableElement.unbindAffected(elements, element, mutateFn);
}
```

**Bidirectional Cleanup**:
- `BoundElement.unbindAffected`: Remove references FROM bound elements TO deleted
- `BindableElement.unbindAffected`: Remove references TO bindable elements FROM deleted

---

## 8. Upstream & Downstream Collaborations

### 8.1 Upstream Dependencies (Inputs)

| Module | Purpose | Key Interfaces |
|--------|---------|----------------|
| `collision.ts` | Hit testing, element intersection | `getHoveredElementForBinding()`, `hitElementItself()`, `intersectElementWithLineSegment()` |
| `bounds.ts` | Center point calculation, element bounds | `elementCenterPoint()`, `aabbForElement()` |
| `heading.ts` | Direction/orientation logic | `headingForPointFromElement()`, `vectorToHeading()` |
| `linearElementEditor.ts` | Arrow point manipulation | `movePoints()`, `getPointGlobalCoordinates()`, `createPointAt()` |
| `mutateElement.ts` | State mutation | `mutateElement()` |
| `Scene.ts` | Element container | `getNonDeletedElementsMap()`, `mutateElement()` |
| `@excalidraw/math` | Geometry primitives | `pointRotateRads()`, `vectorFromPoint()`, `pointDistance()` |

### 8.2 Downstream Consumers (Outputs)

| Component | How it uses bindings |
|-----------|---------------------|
| **Renderer** | Uses fixedPoint + mode to compute actual arrow endpoint position |
| **Undo/Redo** | Binding state included in element snapshot |
| **Collaboration** | Binding properties serialized for sync between peers |
| **Export** | Binding information determines arrow geometry in exports |
| **Flowchart** | Uses binding system to maintain node connections |

### 8.3 Cross-Module Class Collaboration

**BoundElement Class** (`binding.ts:2187-2298`):
- Manages elements WITH bindings (arrows, text)
- `unbindAffected()`: Cleanup when bound element deleted
- `rebindAffected()`: Restore bindings after operations

**BindableElement Class** (`binding.ts:2303+`):
- Manages elements WITH boundElements property
- `unbindAffected()`: Cleanup arrows/text when bindable element deleted

---

## 9. Key State Invariants

### 9.1 Bidirectional Reference Consistency

**Must Hold True**:
```
If arrow.startBinding.elementId = "rect1"
Then rect1.boundElements contains { id: arrow.id, type: "arrow" }
```

**Enforced By**:
- `bindBindingElement()`: Adds arrow to element.boundElements
- `unbindBindingElement()`: Removes arrow from element.boundElements
- `BoundElement/BindableElement.unbindAffected()`: Cleanup on deletion

### 9.2 FixedPoint Range

**Must Hold True**:
```
fixedPoint[0] ∈ [0, 1]
fixedPoint[1] ∈ [0, 1]
```

**Enforced By**:
```typescript
// binding.ts:calculateFixedPointFor...Binding()
normalizeFixedPoint([xRatio, yRatio])
```

### 9.3 Elbow Arrow Constraint

**Must Hold True**: Elbow arrows can only drag ONE endpoint at a time when bound

**Enforced By**:
```typescript
invariant(draggingPoints.size === 1, "Bound elbow arrows cannot be moved")
```

---

## 10. Performance Considerations

### 10.1 Visitor Pattern Efficiency

```typescript
// binding.ts:2133-2182
// Avoids array copies and multiple iterations
boundElementsVisitor() / bindableElementsVisitor()
  ↓ Single pass through related elements
```

### 10.2 Early Termination Points

```typescript
// binding.ts:1135-1137
if (!doesNeedUpdate(element, changedElement)) {
  return;  // Skip unnecessary calculations
}
```

### 10.3 Map-Based Lookups

```typescript
// Using Map<> instead of array find()
const startBindingElement = elementsMap.get(element.startBinding?.elementId);
```

---

## Summary: Critical Flow Diagram

```
Arrow Point Drag
     │
     ▼
handlePointDragging()
     │
     ▼
bindOrUnbindBindingElement()
     │
     ├─► getBindingStrategy...()
     │    ├─► Check both ends dragged? ──► null
     │    ├─► Binding disabled? ───────► null
     │    ├─► Alt key? ───────────────► force inside
     │    ├─► Same element? ───────────► inside-inside
     │    └─► Point position ─────────► inside/orbit
     │
     ├─► bindBindingElement() / unbindBindingElement()
     │    ├─► Calculate fixedPoint
     │    ├─► Update arrow.*Binding
     │    └─► Update element.boundElements
     │
     └─► LinearElementEditor.movePoints()
          └─► Update arrow geometry


Bound Element Movement
     │
     ▼
updateBoundElements()
     │
     ├─► For each bound arrow
     │    ├─► Skip if simultaneously updated
     │    └─► updateBoundPoint()
     │         ├─► Inside mode? ──────► focusPoint
     │         ├─► Too short? ───────► focusPoint
     │         ├─► Inside other? ─────► focusPoint
     │         └─► Normal ───────────► outlinePoint
     └─► LinearElementEditor.movePoints()
```

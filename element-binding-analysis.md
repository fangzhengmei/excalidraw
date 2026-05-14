# Element Binding & Arrow Connection Update Analysis
## Version: 1.0 (Review Ready)

---

## 1. Core Data Structures & State Invariants

### 1.1 FixedPointBinding Structure
**File**: `packages/element/src/types.ts:284-297`
```typescript
type FixedPointBinding = {
  elementId: ExcalidrawBindableElement["id"];  // Reference to bound element
  fixedPoint: FixedPoint;                       // [x, y] ratio (0.0-1.0) relative to element bounds
  mode: BindMode;                                // "inside" | "orbit" | "skip"
};
```

**State Invariants**:
- ✅ `fixedPoint[0] ∈ [0, 1]` - Enforced by `normalizeFixedPoint()`
- ✅ `fixedPoint[1] ∈ [0, 1]` - Enforced by `normalizeFixedPoint()`
- ✅ Bidirectional consistency: If `arrow.startBinding.elementId = rect.id`, then `rect.boundElements` must contain the arrow

### 1.2 Element Binding State
**ExcalidrawArrowElement** (`types.ts:355-359`):
```typescript
type ExcalidrawArrowElement = {
  points: LocalPoint[];          // Arrow geometry (index 0 = start, length-1 = end)
  startBinding: FixedPointBinding | null;
  endBinding: FixedPointBinding | null;
  elbowed: boolean;
  // ...
};
```

**ExcalidrawBindableElement** (`types.ts:35-38, 76`):
```typescript
type BoundElement = { id: string; type: "arrow" | "text" };
type ExcalidrawBindableElement = {
  boundElements: readonly BoundElement[] | null;
  // ...
};
```

---

## 2. Main Link 1: Endpoint Drag Trigger Rebinding

### 2.1 Call Chain Overview
```
packages/excalidraw/components/InteractiveCanvas.tsx: PointerEvent
  ↓
packages/element/src/arrows/helpers.ts: maybeHandleArrowPointlikeDrag()
  ↓ [L442]
packages/element/src/linearElementEditor.ts: LinearElementEditor.handlePointDragging()
  ↓ [L542]
packages/element/src/linearElementEditor.ts: pointDraggingUpdates()
  ↓ [L2167]
packages/element/src/binding.ts: getBindingStrategyForDraggingBindingElementEndpoints()
  ↓ [L621 / L871]
packages/element/src/binding.ts: getBindingStrategyForDraggingBindingElementEndpoints_simple() / _complex()
  ↓
packages/element/src/linearElementEditor.ts: LinearElementEditor.movePoints() [L557]
  ↓ [L1737]
packages/element/src/linearElementEditor.ts: LinearElementEditor._updatePoints()
  ↓
packages/element/src/mutateElement.ts: scene.mutateElement()
```

---

### 2.2 Step-by-Step State Transition

#### Phase 1: Input Capture & Initial Validation
**Location**: `linearElementEditor.ts:442-538`

| Step | Field / State | Change Description |
|------|---------------|---------------------|
| 1 | `lastClickedPoint` | Must be valid index in `selectedPointsIndices` |
| 2 | `selectedPointsIndices` | Contains indices of points being dragged |
| 3 | `startIsSelected / endIsSelected` | Boolean flags indicating which endpoints are being dragged |
| 4 | `deltaX, deltaY` | Calculated movement (pointer position - pointerOffset) |

**Invariant Check**:
```typescript
invariant(element, "Element being dragged must exist in the scene");
invariant(element.points.length > 1, "Element must have at least 2 points");
```

---

#### Phase 2: Binding Strategy Determination
**Location**: `binding.ts:581-997`

**Strategy Decision Tree**:
```
Input: draggingPoints (Map<index, {point, isDragging}>)
       scenePointerX/Y
       appState
       opts (newArrow, angleLocked, altKey)

Branch 1: points.length <= 1 → invariant error
Branch 2: !startDragged && !endDragged → {start: undefined, end: undefined} (no change)
Branch 3: startDragged && endDragged → {start: null, end: null} (BREAK ALL bindings!)
Branch 4: !isBindingEnabled(appState) → {dragged end: null, other: undefined}
Branch 5: isElbowArrow → bindingStrategyForElbowArrowEndpointDragging()
Branch 6: Normal flow → calculate hit element and binding mode
```

**Key State Transitions Within Strategy**:
| Condition | Strategy.mode | Element State Change |
|-----------|---------------|---------------------|
| Both ends dragged | `null` | Binding BROKEN for both ends |
| Binding disabled | `null` | Binding BROKEN for dragged end |
| Alt key pressed | `"inside"` | Force inside binding if hovering element |
| Same element both ends | `"inside"` | Both bind to same element (inside-inside) |
| Point inside hit element | `"inside"` | Normal inside binding |
| Point outside (orbit range) | `"orbit"` | Outline binding |
| No hit element | `null` | Binding BROKEN |

---

#### Phase 3: FixedPoint Calculation
**Location**: `binding.ts:1952-1987` (non-elbow), `binding.ts:1904-1950` (elbow)

```typescript
// Non-elbow arrow binding calculation
const fixedPoint = {
  fixedPoint: [
    (rotatedGlobalPoint.x - bindableElement.x) / bindableElement.width,
    (rotatedGlobalPoint.y - bindableElement.y) / bindableElement.height,
  ]
};
```

**Normalization Step**:
- If calculated ratio < 0 → clamped to 0
- If calculated ratio > 1 → clamped to 1
- Ensures invariant: `fixedPoint ∈ [0, 1] × [0, 1]`

---

#### Phase 4: Apply Binding Update
**Location**: `linearElementEditor.ts:2274-2354`

```typescript
// Strategy mode = null → BREAK binding
if (start.mode === null) {
  updates.startBinding = null;  // Set binding to null
}
// Strategy mode defined → CREATE/UPDATE binding
else if (start.mode) {
  updates.startBinding = {
    elementId: start.element.id,           // Reference to bindable element
    mode: start.mode,                       // "inside" | "orbit"
    ...calculateFixedPoint(...),            // Calculate fixedPoint ratio
  };
}
// Strategy mode undefined → NO CHANGE (keep existing binding)
```

**Key Observation**: The `undefined` mode is CRITICAL - it means "don't touch this binding", which is different from `null` which means "break this binding".

---

#### Phase 5: Point Normalization & Mutation
**Location**: `linearElementEditor.ts:1554-1798`

**Critical Invariant**: Arrow start point MUST be at `[0, 0]` locally
```typescript
// HACK: Instead of moving the start point directly,
// we move ALL other points in the OPPOSITE direction
// and update element.x/y by the offset to maintain [0,0] invariant

const [offsetX, offsetY] = updatedOriginPoint;  // Point at index 0

const nextPoints = points.map((p, idx) => {
  // Move ALL points by -offset to keep index 0 at [0,0]
  return pointFrom(p[0] - offsetX, p[1] - offsetY);
});

// Update element position to compensate
scene.mutateElement(element, {
  points: nextPoints,
  x: element.x + rotatedOffset[0],  // Rotated offset by element angle
  y: element.y + rotatedOffset[1],
  startBinding: updates.startBinding,  // Apply binding change
  endBinding: updates.endBinding,
});
```

---

#### Phase 6: Bidirectional Reference Sync
**Location**: `binding.ts:1016-1068` (`bindBindingElement`)

```typescript
// Step 1: Update arrow → element reference (done above)
scene.mutateElement(arrow, {
  [startOrEnd === "start" ? "startBinding" : "endBinding"]: binding
});

// Step 2: Update element → arrow reference (bidirectional)
const boundElementsMap = arrayToMap(hoveredElement.boundElements || []);
if (!boundElementsMap.has(arrow.id)) {
  scene.mutateElement(hoveredElement, {
    boundElements: (hoveredElement.boundElements || []).concat({
      id: arrow.id,
      type: "arrow",
    }),
  });
}
```

**Why Bidirectional?**
- Enables fast lookup: `element.boundElements` tells you all arrows connected to this element
- Needed for `updateBoundElements()` when element moves
- Critical for deletion cleanup

---

## 3. Main Link 2: Bound Element Movement Trigger Arrow Update

### 3.1 Call Chain Overview
```
Element Movement (drag/resize/rotate)
  ↓
packages/element/src/binding.ts: updateBindings() [L1268-1305]
  ↓ [L1104]
packages/element/src/binding.ts: updateBoundElements()
  ↓ [L1129]
Visitor Pattern: For each arrow in changedElement.boundElements
  ↓ [L1746]
packages/element/src/binding.ts: updateBoundPoint()
  ↓ [L1554]
packages/element/src/linearElementEditor.ts: LinearElementEditor.movePoints()
  ↓
Mutation & Render
```

---

### 3.2 Step-by-Step State Transition

#### Phase 1: Entry Point Filtering
**Location**: `binding.ts:1104-1127`

| Check | Action | Purpose |
|-------|--------|---------|
| `!isBindableElement(changedElement)` | Early return | Skip non-bindable elements (lines, text, etc.) |
| `boundElements.length === 0` | N/A | Nothing to update, visitor does nothing |
| `changedElement.isDeleted` | N/A | Handled by separate deletion flow |

---

#### Phase 2: Visitor Pattern Execution
**Location**: `binding.ts:1129-1204`

```typescript
const visitor = (arrow: ExcalidrawElement | undefined) => {
  // Guard clauses
  if (!isArrowElement(arrow) || arrow.isDeleted) return;
  if (!doesNeedUpdate(arrow, changedElement)) return;
  if (simultaneouslyUpdatedElementIds.has(arrow.id)) return;  // Skip if arrow also selected

  // Calculate new endpoint positions for BOTH ends
  const updates = bindableElementsVisitor(
    elementsMap, arrow, (bindableElement, bindingProp) => {
      // Only update if this binding relates to the changed element
      if (changedElement.id === arrow[bindingProp]?.elementId) {
        const newPoint = updateBoundPoint(
          arrow, bindingProp, arrow[bindingProp], bindableElement, elementsMap
        );
        return [pointIndex, { point: newPoint }];
      }
      return null;
    }
  ).filter(x => x !== null);

  // Apply updates to arrow
  LinearElementEditor.movePoints(arrow, scene, new Map(updates), {
    moveMidPointsWithElement: bothEndsBoundToSameElement,
  });
};
```

**`doesNeedUpdate` Logic** (`binding.ts:1307-1315`):
```typescript
return (
  boundElement.startBinding?.elementId === changedElement.id ||
  boundElement.endBinding?.elementId === changedElement.id
);
```

---

#### Phase 3: updateBoundPoint Decision Logic
**Location**: `binding.ts:1746-1902`

This is the CORE of endpoint recalculation. **5-step decision tree**:

```
Input: binding mode, opposite binding state, other endpoint position

Step 1: INSIDE MODE SHORTCUT
  if binding.mode === "inside" → use focusPoint directly
  → focusPoint = element position + ratio transformation

Step 2: OUTLINE INSIDE OTHER ELEMENT?
  if outlinePoint would be inside opposite bound element
  → use focusPoint instead to avoid arrow inversion

Step 3: ARROW TOO SHORT (UNCONNECTED)
  if opposite end unbound AND arrow too short
  → use focusPoint instead of outline

Step 4: ARROW TOO SHORT (CONNECTED)
  if both ends bound AND arrow too short
  → use resolved target point

Step 5: GENERAL CASE (NORMAL OPERATION)
  → snap to element outline at intersection with arrow direction
```

**Focus Point Calculation**:
```typescript
// From ratio to global coordinates
focusPointGlobal = pointFrom(
  bindableElement.x + fixedPoint[0] * bindableElement.width,
  bindableElement.y + fixedPoint[1] * bindableElement.height
);
// Then rotate by bindableElement.angle around element center
```

**Outline Snap Calculation**:
```typescript
// Ray cast from arrow direction to element boundary
intersections = intersectElementWithLineSegment(
  bindableElement, elementsMap, lineSegment, bindingGap
);
outlinePoint = closestIntersectionToArrowEndpoint;
```

---

#### Phase 4: Midpoint Handling
**Location**: `binding.ts:1191-1195`

```typescript
moveMidPointsWithElement:
  startBindingElement?.id === endBindingElement?.id
  // TRUE if both ends bound to SAME element
```

**Behavior**:
- `true`: Arrow midpoints move with the element (maintains shape)
- `false`: Only endpoints move (arrow stretches/rotates)

---

## 4. Exception Boundaries: Trigger Conditions, Branching, Recovery

### 4.1 Hard Failures (Invariant Violations)
| Location | Trigger Condition | Branch Outcome | Recovery Strategy |
|----------|-------------------|----------------|-------------------|
| `binding.ts:646-649` | `arrow.points.length <= 1` | **Throw Error** | Should never reach here - checked at higher level |
| `binding.ts:686-691` | Dragged endpoint has no point data | **Throw Error** | Defensive check - points map should always have entry |
| `binding.ts:259` | Elbow arrow drags multi points | **Throw Error** | Elbow arrows only support single endpoint drag |
| `linearElementEditor.ts:168-179` | Start point not at `[0,0]` | **Console Error + Fix** | Auto-normalize points and mutate |

---

### 4.2 Soft Failures (Graceful Degradation)

#### Category A: Null/Undefined Handling
| Location | Trigger Condition | Branch Outcome | Recovery |
|----------|-------------------|----------------|----------|
| `binding.ts:1129-1132` | `!isArrowElement(element)` | Skip visitor | Ignore non-arrow bound elements |
| `binding.ts:1135-1137` | `!doesNeedUpdate(element)` | Skip visitor | Ignore arrows not bound to changed element |
| `binding.ts:1151-1153` | Arrow also being dragged | Skip visitor | Avoid double-update loops |
| `binding.ts:2222-2261` | Bound element not found in map | Skip that endpoint | Stale binding reference ignored |

#### Category B: Edge Case Handling
| Location | Trigger Condition | Branch Outcome | Recovery |
|----------|-------------------|----------------|----------|
| `binding.ts:736-769` | Both endpoints hovering same element | Inside-inside binding | Special case for "loop" arrows |
| `binding.ts:771-793` | Alt key pressed | Force inside binding | User override for precision |
| `linearElementEditor.ts:1571-1589` | Polygon line element | Sync start/end points | Maintain closed shape |
| `binding.ts:1838-1859` | Outline point inside opposite element | Use focusPoint | Prevent arrow inversion |
| `binding.ts:1871-1879` | Arrow too short (unconnected) | Use focusPoint | Avoid degenerate arrow |

#### Category C: Simultaneous Update Detection
**Location**: `binding.ts:1116-1121, 1150-1153`
```typescript
const simultaneouslyUpdatedElementIds = new Set(
  (simultaneouslyUpdated || []).map(e => e.id)
);

if (simultaneouslyUpdatedElementIds.has(arrow.id)) {
  return;  // SKIP - arrow is already being moved by user
}
```

**Why Needed?**
- User drags rectangle AND connected arrow together
- Without check: arrow would be updated twice → double offset
- Result: arrow drifts away from binding point

---

## 5. Upstream & Downstream Collaboration (Exact Call Relationships)

### 5.1 Upstream Dependencies

| Module | Function Call | Purpose |
|--------|---------------|---------|
| `collision.ts` | `getHoveredElementForBinding(point, elements, elementsMap, maxDistance)` | Find element under pointer for binding |
| `collision.ts` | `isPointInElement(point, element, elementsMap)` | Check if point is inside element bounds |
| `collision.ts` | `hitElementItself({point, element, threshold, ...})` | Check if point is close to/inside element outline |
| `collision.ts` | `intersectElementWithLineSegment(element, elementsMap, lineSegment, gap)` | Ray cast to find arrow outline intersection |
| `bounds.ts` | `getElementPointsCoords(element, points)` | Calculate bounding box from points |
| `bounds.ts` | `elementCenterPoint(element, elementsMap)` | Get element center in global coordinates |
| `heading.ts` | `vectorToHeading(vector)` | Convert direction vector to heading enum |
| `heading.ts` | `headingForPointFromElement(element, aabb, point)` | Determine heading from element to point |
| `mutateElement.ts` | `scene.mutateElement(element, updates, options)` | Apply state changes to element |
| `Scene.ts` | `scene.getNonDeletedElementsMap()` | Get current element state map |
| `@excalidraw/math` | `pointRotateRads(point, center, angle)` | Rotate point around center |
| `@excalidraw/math` | `pointDistance(a, b)` | Calculate distance between points |

---

### 5.2 Downstream Consumers

| Component | How It Uses Binding Data |
|-----------|--------------------------|
| **Renderer** | Uses `fixedPoint + mode` to compute actual arrow endpoint position, then draws line to outline or focus point |
| **Undo/Redo** | `startBinding/endBinding` included in element snapshot - restored as-is |
| **Collaboration** | Binding properties serialized and synced between peers |
| **Export** | Binding info used to compute final arrow geometry in SVG/PNG export |
| **Flowchart** | Uses binding system to maintain node connections when layout changes |
| **Z-Order** | `moveArrowAboveBindable()` brings connected arrows above elements when near |

---

### 5.3 Cross-Module Class Collaboration

#### BoundElement Class (`binding.ts:2187-2298`)
**Purpose**: Manage elements WITH bindings (arrows, text)

```typescript
// Main operations:
BoundElement.unbindAffected(elements, deletedElement, mutateFn);
  → Removes references FROM bound elements TO deleted
  → Updates container.boundElements array

BoundElement.rebindAffected(elements, boundElement, mutateFn);
  → Restores references after operations
  → Skips if already bound
```

#### BindableElement Class (`binding.ts:2303+`)
**Purpose**: Manage elements WITH boundElements property (rectangles, etc.)

```typescript
BindableElement.unbindAffected(elements, deletedElement, mutateFn);
  → Removes references TO bindable elements FROM deleted
  → Sets arrow.startBinding/endBinding to null
```

**Deletion Cleanup Sequence** (`binding.ts:2061-2075`):
```typescript
// IMPORTANT: Call BOTH unbind functions on deletion!
for (const element of deletedElements) {
  BoundElement.unbindAffected(elements, element, mutateFn);
  BindableElement.unbindAffected(elements, element, mutateFn);
}
// This ensures ALL references are cleaned up in BOTH directions
```

---

## 6. Performance Considerations

### 6.1 Visitor Pattern Efficiency
- **O(N)** where N = number of bound elements (typically small)
- Single pass through related elements
- Early termination at multiple guard clauses

### 6.2 Map-Based Lookups
- `elementsMap.get(binding.elementId)` → **O(1)**
- Avoids `array.find()` which would be **O(M)** per lookup

### 6.3 Early Termination Points
- Element not bindable? → return immediately
- Arrow already being updated? → skip (simultaneous update check)
- No binding change needed? → skip point calculation

---

## 7. Summary Flow Diagrams

### 7.1 Endpoint Drag → Rebind Flow
```
Pointer Drag Event
    │
    ├─ Calculate deltaX/deltaY
    │
    ├─ Determine binding strategy
    │   ├─ Both ends dragged? → BREAK ALL bindings
    │   ├─ Binding disabled? → BREAK dragged binding
    │   ├─ Elbow arrow? → Special elbow strategy
    │   └─ Normal:
    │       ├─ Hit element under pointer?
    │       ├─ Point inside element? → "inside" mode
    │       └─ Point outside (range)? → "orbit" mode
    │
    ├─ Calculate fixedPoint ratio [0-1]
    │
    ├─ Update arrow.startBinding/endBinding
    │
    ├─ Add arrow reference to element.boundElements
    │
    └─ Normalize arrow points (maintain [0,0] invariant)
        └─ Move all points by -offset, update element.x/y by +offset
```

### 7.2 Element Move → Arrow Update Flow
```
Element Position/Size Change
    │
    └─ For EACH bound arrow:
        ├─ Skip if arrow is also being dragged
        │
        ├─ For EACH bound endpoint:
        │   ├─ Inside mode? → Recalculate focusPoint from ratio
        │   ├─ Outline inside opposite element? → Use focusPoint
        │   ├─ Arrow too short? → Use focusPoint
        │   └─ Normal case? → Snap to element outline
        │
        └─ Apply point updates to arrow (maintaining [0,0] invariant)
            └─ If both ends same element: move midpoints too
```

---

**Document Version**: 1.0 | **Status**: Review Ready | **Last Updated**: 2026-05-14

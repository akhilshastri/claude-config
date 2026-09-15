---
name: flexlayout-react
description: Build multi-panel docking/tabbed layouts in React with flexlayout-react (Caplin) — JSON model structure (rows/tabsets/tabs/borders), the factory function, dispatching Actions (add/move/delete/maximize tabs), persisting layout state, theming, and custom tab/tabset rendering. Use when the user mentions FlexLayout, flexlayout-react, docking layout, or wants resizable/draggable tabsets and panels (IDE-style or trading-desk-style UI) in React.
---

# FlexLayout (flexlayout-react)

A React docking-layout manager: resizable/draggable tabsets, borders (docked side panels),
popout windows, all driven by a JSON model. Repo: https://github.com/caplin/FlexLayout — docs
live under `_autodocs/` in that repo (also indexed in Context7 as `/caplin/flexlayout`). Live
demo: https://caplin.github.io/FlexLayout/demos/v0.10/demo/index.html. Fetch a specific
`_autodocs/*.md` file from the repo, or query Context7, for anything not covered here.

## 1. Install

```bash
bun add flexlayout-react
```

## 2. Minimal example

```jsx
import { Layout, Model } from 'flexlayout-react';
import 'flexlayout-react/style/light.css';   // pick one theme, see §5

const json = {
  global: {},
  borders: [],
  layout: {
    type: 'row',
    weight: 100,
    children: [
      {
        type: 'tabset',
        weight: 50,
        children: [{ type: 'tab', name: 'One', component: 'placeholder' }],
      },
      {
        type: 'tabset',
        weight: 50,
        children: [{ type: 'tab', name: 'Two', component: 'placeholder' }],
      },
    ],
  },
};

const model = Model.fromJson(json);

function App() {
  const factory = (node) => {
    const component = node.getComponent();
    if (component === 'placeholder') return <div>{node.getName()}</div>;
    // route other component names to real panels here
  };

  return <Layout model={model} factory={factory} />;
}
```

The `factory` is how tab content is rendered — it receives the `TabNode`, and you switch on
`node.getComponent()` (a string you chose in the JSON) to pick which React component to render,
using `node.getName()` / `node.getConfig()` for tab-specific data.

## 3. JSON model shape

```json
{
  "global": {
    "tabEnableClose": true,
    "tabEnableRename": true,
    "tabSetEnableMaximize": true,
    "enableEdgeDock": true,
    "rootOrientationVertical": false,
    "borderSize": 200,
    "borderMinSize": 100,
    "borderMaxSize": 500
  },
  "borders": [
    {
      "location": "left",
      "size": 250,
      "children": [{ "type": "tab", "id": "nav-tab", "name": "Navigation", "component": "nav" }]
    },
    {
      "location": "bottom",
      "size": 200,
      "children": [{ "type": "tab", "name": "Console", "component": "console" }]
    }
  ],
  "layout": {
    "type": "row",
    "id": "root",
    "children": [
      {
        "type": "tabset",
        "id": "main-tabset",
        "weight": 100,
        "selected": 0,
        "children": [
          { "type": "tab", "id": "tab1", "name": "Dashboard", "component": "dashboard", "icon": "📊", "pinned": true },
          { "type": "tab", "id": "tab2", "name": "Details", "component": "details" }
        ]
      }
    ]
  }
}
```

- **`global`** — defaults inherited by every node (see the attribute list below); any node can
  override a global attribute by setting the same-named property directly on itself
  (`tabEnableClose` global → `enableClose` on a specific tab node).
- **`borders`** — up to 4 docked side panels (`location`: `"left" | "right" | "top" | "bottom"`),
  each holding its own tabset of tabs.
- **`layout`** — the main area: a tree of `row` (splits children along one axis) and `tabset`
  nodes, bottoming out in `tab` nodes. `weight` controls relative split size within a row.
- Give tabs/tabsets/borders a stable `id` when you'll need to target them later with `Actions`.

Common global attributes: `tabEnableClose`, `tabEnableDrag`, `tabEnableRename`,
`tabEnablePopout`, `tabEnableScrollbars`, `tabMinWidth`/`tabMaxWidth`, `tabSetEnableTabStrip`,
`tabSetEnableDrag`, `tabSetEnableDrop`, `tabSetEnableMaximize`, `tabSetEnableDivide`,
`tabSetAutoSelectTab`, `enableEdgeDock`, `enableEdgeDockIndicators`, `rootOrientationVertical`,
`borderSize`/`borderMinSize`/`borderMaxSize`, `borderAutoSelectTabWhenOpen`.

## 4. Mutating the layout — `Actions` + `model.doAction`

Never hand-edit the model tree; dispatch an `Action`. `Model.doAction` returns the new `Node`
for `addTab`, a `layoutId` for `createPopout`, `undefined` otherwise.

```javascript
import { Actions, DockLocation } from 'flexlayout-react';

// add a tab into an existing tabset (index -1 = append, select = focus it)
model.doAction(Actions.addTab(
  { type: 'tab', name: 'New', component: 'view' },
  'tabsetId', DockLocation.CENTER, -1, true
));

model.doAction(Actions.moveNode('tabId', 'otherTabsetId', DockLocation.CENTER, 0));
model.doAction(Actions.deleteTab('tabId'));
model.doAction(Actions.deleteTabset('tabsetId'));
model.doAction(Actions.renameTab('tabId', 'New Name'));
model.doAction(Actions.selectTab('tabId'));
model.doAction(Actions.setTabPinned('tabId', true));
model.doAction(Actions.maximizeToggle('tabsetId'));
model.doAction(Actions.setActiveTabset('tabsetId'));
model.doAction(Actions.updateNodeAttributes('tabId', { enableClose: false }));
model.doAction(Actions.updateModelAttributes({ tabEnableClose: false }));
model.doAction(Actions.popoutTab('tabId'));          // pop a tab into its own OS window
model.doAction(Actions.closePopout(layoutId));
```

Full `Actions` surface: `addTab`, `deleteTab`, `deleteTabset`, `renameTab`, `setTabPinned`,
`setBorderType`, `selectTab`, `moveNode`, `setActiveTabset`, `adjustWeights`,
`adjustBorderSplit`, `maximizeToggle`, `updateModelAttributes`, `updateNodeAttributes`,
`popoutTab`, `popoutTabset`, `closePopout`, `movePopoutToFront`, `moveFloat`,
`createSubLayout`.

### From a ref, without building the Action yourself

```jsx
const layoutRef = useRef(null);

<Layout ref={layoutRef} model={model} factory={factory} />

// drag-to-add from an external element (call inside a dragstart handler)
const handleDragStart = (e) => {
  layoutRef.current.addTabWithDragAndDrop(
    e.nativeEvent,
    { type: 'tab', name: 'Dragged Tab', component: 'grid' }
  );
};
```

## 5. State management (React) + persistence

Treat the `Model` as the single source of truth; keep it in React state and let
`onModelChange` fire on every layout mutation (drag, resize, close, your own `doAction` calls):

```jsx
const [model, setModel] = useState(() => Model.fromJson(json));

const handleModelChange = (newModel, action) => {
  // newModel is already the updated model (same reference model.doAction mutated)
  localStorage.setItem('layoutState', JSON.stringify(newModel.toJson()));
};

<Layout model={model} factory={factory} onModelChange={handleModelChange} />
```

Restore on load, and preserve current tab view-state (scroll position etc.) across a JSON
re-apply by passing the previous model in as the second arg:

```javascript
const saved = JSON.parse(localStorage.getItem('layoutState') ?? 'null');
const model = Model.fromJson(saved ?? defaultJson);

// later, applying a fresh layout without losing state of tabs that still exist:
const nextModel = Model.fromJson(newJson, model);
```

## 6. Theming

```javascript
import 'flexlayout-react/style/light.css';
import 'flexlayout-react/style/dark.css';
import 'flexlayout-react/style/alpha_light.css';
import 'flexlayout-react/style/alpha_dark.css';
import 'flexlayout-react/style/underline.css';
import 'flexlayout-react/style/gray.css';
import 'flexlayout-react/style/rounded.css';
import 'flexlayout-react/style/combined.css';   // all themes bundled, for runtime switching
```

Import exactly one (except `combined.css`, which lets you switch themes at runtime by toggling
a class on the layout root).

## 7. Custom tab / tabset rendering

```jsx
<Layout
  model={model}
  factory={factory}
  onRenderTab={(node, renderValues) => {
    renderValues.leading = <img src={node.getIcon()} />;           // icon/leading slot
    renderValues.buttons.push(<button key="info">ℹ</button>);       // extra tab button
    if (node.getExtraData().isDirty) renderValues.content = node.getName() + ' *';
  }}
  onRenderTabSet={(node, renderValues) => {
    renderValues.stickyButtons.push(
      <button key="add" onClick={() =>
        model.doAction(Actions.addTab({ type: 'tab', component: 'grid', name: 'New' },
          node.getId(), DockLocation.CENTER, -1, true))
      }>➕</button>
    );
  }}
/>
```

`ITabRenderValues`: `leading`, `content`, `buttons`. `ITabSetRenderValues`: `leading`,
`stickyButtons`, `buttons`, `overflowPosition`.

## 8. Where to go deeper

- Repo root / README: https://github.com/caplin/FlexLayout
- Docs source (fetch these directly if not covered above):
  `_autodocs/quick-reference.md`, `_autodocs/configuration.md`, `_autodocs/actions-api.md`,
  `_autodocs/model-classes.md`, `_autodocs/layout-component.md`,
  `_autodocs/react-integration.md`, `_autodocs/popout-windows.md`,
  `_autodocs/export-reference.md` — all under
  https://github.com/caplin/flexlayout/blob/master/_autodocs/
- Live interactive demo (great for confirming exact visual/attribute behavior before
  implementing): https://caplin.github.io/FlexLayout/demos/v0.10/demo/index.html
- TypeDoc API reference: https://caplin.github.io/FlexLayout/demos/v0.10/typedoc/index.html
- Context7 library id: `/caplin/flexlayout`

Prefer Context7 or the `_autodocs` source over recalling exact `Actions`/attribute names from
memory — this is a smaller-community library with a real but less-searchable doc set.

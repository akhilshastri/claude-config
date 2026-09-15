---
name: ag-grid
description: Build and configure AG Grid React data grids — installation, module registration, theming, and all four row models (Client-Side, Infinite, Server-Side, Viewport) with working datasource implementations. Use when the user mentions ag-grid, AgGridReact, rowModelType, or wants a data table/grid with sorting, filtering, grouping, infinite scroll, or server-driven paging in a React app.
---

# AG Grid (React)

AG Grid docs: https://www.ag-grid.com/react-data-grid/getting-started/ — this skill covers
setup and all four row models. Fetch a specific `/react-data-grid/...` page (or query Context7
library `/websites/ag-grid_react-data-grid`) for anything not covered here — the API surface is
large and changes between major versions.

## 1. Install

```bash
bun add ag-grid-react ag-grid-community      # Community only
bun add ag-grid-react ag-grid-community ag-grid-enterprise   # + Enterprise (Server-Side, Viewport, grouping, etc.)
```

`ag-grid-react` pulls in `ag-grid-community` as a dependency; add `ag-grid-enterprise`
separately for the Server-Side Row Model, Viewport Row Model, row grouping/pivoting, and other
enterprise-only features (needs a license key in production; runs in trial/watermark mode
without one).

## 2. Module registration (required — grid renders nothing without it)

AG Grid is modular: every feature (a row model, a filter type, row selection, grouping...) is a
separate module you must explicitly register, or the grid silently no-ops that feature.

**React-idiomatic pattern — wrap the app/grid in `AgGridProvider`:**

```jsx
import { AllCommunityModule } from 'ag-grid-community';
import { AgGridProvider, AgGridReact } from 'ag-grid-react';

const modules = [AllCommunityModule];   // dev convenience; prefer only the modules you use

function App() {
  return (
    <AgGridProvider modules={modules}>
      <AgGridReact rowData={rowData} columnDefs={columnDefs} />
    </AgGridProvider>
  );
}
```

**Imperative alternative — `ModuleRegistry.registerModules([...])`** (framework-agnostic, works
before any grid mounts, e.g. in an app entry point):

```javascript
import { ModuleRegistry } from 'ag-grid-community';
import { AllEnterpriseModule, LicenseManager } from 'ag-grid-enterprise';

LicenseManager.setLicenseKey('YOUR_LICENSE_KEY');
ModuleRegistry.registerModules([AllEnterpriseModule]);
```

Register only the modules a grid actually uses in production for smaller bundles — `AllCommunityModule`/`AllEnterpriseModule` disable tree-shaking. Each row model needs its own module (see §4 table) plus whatever feature modules the columns use (`TextFilterModule`, `RowSelectionModule`, `RowGroupingModule`, `SetFilterModule`, etc).

In dev, also call `enableDevValidations()` once (guarded by `NODE_ENV !== 'production'`) — it
surfaces missing-module and misconfiguration warnings in the console instead of silent failures:

```javascript
import { enableDevValidations } from 'ag-grid-community';
if (process.env.NODE_ENV !== 'production') enableDevValidations();
```

## 3. Minimal example (Client-Side Row Model, the default)

```tsx
'use client';
import React, { useState } from 'react';
import type { ColDef } from 'ag-grid-community';
import { AllCommunityModule } from 'ag-grid-community';
import { AgGridProvider, AgGridReact } from 'ag-grid-react';

interface IRow { make: string; model: string; price: number; electric: boolean }

const GridExample = () => {
  const [rowData] = useState<IRow[]>([
    { make: 'Tesla', model: 'Model Y', price: 64950, electric: true },
    { make: 'Ford', model: 'F-Series', price: 33850, electric: false },
  ]);
  const [colDefs] = useState<ColDef<IRow>[]>([
    { field: 'make' }, { field: 'model' }, { field: 'price' }, { field: 'electric' },
  ]);

  return (
    <AgGridProvider modules={[AllCommunityModule]}>
      <div style={{ width: '100%', height: '500px' }}>
        <AgGridReact rowData={rowData} columnDefs={colDefs} />
      </div>
    </AgGridProvider>
  );
};
export default GridExample;
```

The grid needs an explicit height on its container (percentage heights need a sized parent too)
— a common "grid doesn't render" cause is a `0px`-height wrapper.

### Theming

Current API is JS **theme objects** (`Theming API`), not just CSS classes:

```javascript
import { themeQuartz } from 'ag-grid-community';   // or themeBalham, themeAlpine

const myTheme = themeQuartz.withParams({
  accentColor: '#0e4491',
  backgroundColor: '#ffffff',
  headerBackgroundColor: '#faf8f5',
});

<AgGridReact theme={myTheme} ... />
```

`withParams(paramsObj, 'light' | 'dark')` can be chained twice to define both modes; toggle by
setting `document.body.dataset.agThemeMode = 'dark' | 'light'`. To keep the old CSS-class themes
(`ag-theme-quartz` etc.) instead of the Theming API, set `theme="legacy"` on the grid (or
`provideGlobalGridOptions({ theme: 'legacy' })` once, app-wide) and import the legacy CSS files.

## 4. Row models — pick one via `rowModelType`

| Row model | `rowModelType` | Module (package) | Data lives | Use when |
|---|---|---|---|---|
| Client-Side (default) | `'clientSide'` (or omit) | `ClientSideRowModelModule` (community) | All in browser memory | Whole dataset fits in memory; need in-browser sort/filter/group/pivot |
| Infinite | `'infinite'` | `InfiniteRowModelModule` (community) | Server, flat pages | Large flat dataset, incremental scroll-driven loading, no server-side grouping needed |
| Server-Side | `'serverSide'` | `ServerSideRowModelModule` (**enterprise**) | Server, lazy-loaded | Large dataset needing server-side grouping/pivoting/aggregation/filtering — the superset of Infinite |
| Viewport | `'viewport'` | `ViewportRowModelModule` (**enterprise**) | Server, only-visible-rows | Live/streaming data source that should only push updates for rows currently on screen |

### 4a. Client-Side Row Model

Default — just pass `rowData`. Update it via `applyTransaction` for efficient incremental
changes instead of replacing the whole array (important at scale):

```javascript
const myTransaction = {
  add: [{ employeeId: '4', name: 'Billy', age: 55 }],      // no existing row with this ID
  update: [{ employeeId: '2', name: 'Bob', age: 23 }],      // matched by ID, full row replaced
  remove: [{ employeeId: '5' }],                            // only ID is needed
};
gridRef.current.api.applyTransaction(myTransaction);
```

Requires `getRowId={(params) => String(params.data.id)}` on `AgGridReact` so the grid can match
rows by a stable ID rather than object identity/index. Modules: `ClientSideRowModelModule`,
`ClientSideRowModelApiModule` (for `applyTransaction`/`refreshClientSideRowModel` via the API),
`RowApiModule`.

### 4b. Infinite Row Model

Implement `IDatasource` and set it as `datasource` (or via `rowModelType="infinite"` +
`datasource` prop):

```javascript
const dataSource = {
  rowCount: undefined,   // unknown up front
  getRows: (params) => {
    // params.startRow, params.endRow, params.sortModel, params.filterModel
    fetchPage(params.startRow, params.endRow).then(({ rows, total }) => {
      const lastRow = total <= params.endRow ? total : -1;   // -1 = "more rows exist"
      params.successCallback(rows, lastRow);
    }).catch(() => params.failCallback());
  },
};

<AgGridReact
  rowModelType="infinite"
  cacheBlockSize={100}              // rows per block/request
  cacheOverflowSize={2}
  maxConcurrentDatasourceRequests={2}
  infiniteInitialRowCount={1}
  maxBlocksInCache={2}
  onGridReady={(params) => params.api.setGridOption('datasource', dataSource)}
/>
```

Module: `InfiniteRowModelModule` (community — no enterprise license needed). No server-side
grouping/aggregation; for that use Server-Side.

### 4c. Server-Side Row Model (enterprise)

Implement `IServerSideDatasource` — `getRows(params)` receives `params.request` (with
`rowGroupCols`, `groupKeys`, sort/filter models, `startRow`/`endRow`) and must call
`params.success({ rowData, rowCount })` or `params.fail()`:

```javascript
const getServerSideDatasource = (server) => ({
  getRows: (params) => {
    const response = server.getData(params.request);
    if (response.success) {
      params.success({ rowData: response.rows, rowCount: response.lastRow });
    } else {
      params.fail();
    }
  },
});

<AgGridReact
  rowModelType="serverSide"
  columnDefs={[
    { field: 'country', rowGroup: true, hide: true },   // grouping happens server-side
    { field: 'gold', aggFunc: 'sum', enableValue: true },
  ]}
  autoGroupColumnDef={{ flex: 1, minWidth: 280 }}
  getChildCount={(data) => data?.childCount}
  onGridReady={(params) => params.api.setGridOption('serverSideDatasource', getServerSideDatasource(server))}
/>
```

Modules: `ServerSideRowModelModule` (enterprise) + `RowGroupingModule` (enterprise) if grouping.
This is the superset of Infinite — supports lazy grouped rows, server-side aggregation/pivot,
and per-group data slicing.

### 4d. Viewport Row Model (enterprise)

For live-streaming sources: the grid tells the datasource exactly which row range is on screen,
and the datasource only ever pushes data for that visible window. Implement
`IViewportDatasource`:

```javascript
const createViewportDatasource = () => {
  let initParams;
  return {
    init: (params) => {
      initParams = params;
      params.setRowCount(1_000_000);        // total row count (can change later)
    },
    setViewportRange(firstRow, lastRow) {
      const rowData = {};
      for (let i = firstRow; i <= lastRow; i++) rowData[i] = fetchRow(i);
      initParams.setRowData(rowData);        // map keyed by row index
    },
    destroy: () => {},
  };
};

<AgGridReact
  rowModelType="viewport"
  viewportDatasource={createViewportDatasource()}
  rowHeight={100}
/>
```

`IViewportDatasourceParams` (passed into `init`) exposes `setRowCount(count, keepRenderedRows?)`,
`setRowData(map)`, `getRow(rowIndex)`, plus `api`/`context`. Module: `ViewportRowModelModule`
(enterprise). This is the row model to reach for when wiring the grid up to a pub/sub feed
(e.g. AMPS SOW + subscribe — see the `amps` skill — push updates only for the visible range).

## 5. Where to go deeper

- Getting started: https://www.ag-grid.com/react-data-grid/getting-started/
- Row models overview: https://www.ag-grid.com/react-data-grid/row-models
- Infinite scrolling: https://www.ag-grid.com/react-data-grid/infinite-scrolling
- Server-side datasource: https://www.ag-grid.com/react-data-grid/server-side-model-datasource
- Server-side grouping: https://www.ag-grid.com/react-data-grid/server-side-model-grouping
- Viewport: https://www.ag-grid.com/react-data-grid/viewport
- Data update transactions (Client-Side): https://www.ag-grid.com/react-data-grid/data-update-transactions
- Theming: https://www.ag-grid.com/react-data-grid/theming-colors, migration guide at
  https://www.ag-grid.com/react-data-grid/theming-migration

Prefer Context7 (`/websites/ag-grid_react-data-grid`) or fetching the specific doc page over
recalling exact prop names/module names from memory — module names and the theming API changed
significantly around v32/v33, and this SDK's surface is large.

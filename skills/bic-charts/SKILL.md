---
name: bic-charts
description: Build a web app or dashboard containing BIC-generated charts — a bubble map, a table with in-cell bars, linked cross-filtering. Use whenever the bic-chart MCP tools are available and the task is "put a chart / dashboard in this app".
---

# Building an app with BIC charts

The golden path, end to end. Following it costs a handful of tool calls; deriving it from
first principles costs about thirty, and the places people lose time are all called out
below because they were all measured.

**The mental model.** A BIC chart is *generated source you own* — a plain
`render(container, data, options)` function you commit to your repo. There is no API key at
run time, no per-render cost, and no network call. `@bicharts/chart-host` runs it; the MCP
generates it. Generation happens **once, at build time, by you** — never in your users'
browsers.

**Signing in needs no setup.** The first call that needs an account (`list_eligible_charts`,
`generate_chart`) opens the person's browser on a sign-in page, waits for them, and then finishes
the call. Tell them a browser tab is about to open. If the call comes back with "Sign in to
continue" and a link instead, give them that link, and call again once they've signed in. A key in
the server's environment is only for machines with no browser, such as CI.

## 0. Decide what to pull (only if you are starting from a data MODEL)

Skip this if you already have a table. If you are holding a semantic model, a warehouse schema or
a database, **you** pick the fields — the tools take a table, not a model, and they cannot see
what you can. Three decisions, and the third is the one that goes wrong.

**Asked which charts suit the model itself** ("good chart types for this dataset", "what could we
chart here")? Over a semantic model in a Fabric App, `suggest_charts` answers in one call: it reads
the model's tables, measures and relationships, finds its natural views (measures by place, flows
between places, measures over time, measures by category), queries each at its grain, keeps the
measures that vary there and asks the eligibility engine what fits each. Answer from its reply -
the counts in it are the model's own - and build the one the person picks from the rows it saved
(`.bic/data/<view>.csv`, the DAX beside it). Pass `views` (`columns` as `Table[Column]`, `measures`
by name, or your own `dax`) to ask about other groupings. Elsewhere, pull each candidate grouping
and ask `list_eligible_charts` about each.

**Fields and filters** follow from the question. *"All loans over the last 6 months, highlighting
those which defaulted, paying attention to credit score"* wants `LoanId`, `Defaulted`,
`CreditScore`, filtered to six months.

**A highlight is a COLUMN, not a styling note.** "Highlighting those which defaulted" means
`Defaulted` travels as its own categorical column. There is no instruction channel that can colour
rows the data does not distinguish.

**GRAIN is a choice, and the default is usually wrong for these charts.** Almost every BI query
aggregates — `GROUP BY`, `SUMMARIZECOLUMNS`, a pivot. But that sentence is asking about *the
distribution of individual loans*, which needs **one row per loan**. Aggregate it and you get a
clean bar chart of average credit score by default status: a valid chart, answering a different
question, and nothing downstream will flag it. Ask yourself which you meant:

| you want | pull | you get |
| --- | --- | --- |
| how do individual loans spread out | **one row per loan** | beeswarm, strip plot, scatter, histogram |
| how do the groups compare on a total | one row per group | bars, treemap, pie |
| how does it move over time | one row per period (per series) | lines, area, streamgraph |

**Aim with `CHART-SHAPES.md`** (beside this file). It lists every chart the engine can build and
what each one needs — minimum columns by role, category ceilings, draw caps, viewport floors,
which renderers can draw it. It is GENERATED from the chart catalogue, so it cannot drift from what
the engine will actually accept. Read the LIMITS column carefully: **the units differ** — a sankey
is capped on EDGES and a chord on NODES, so `<=80 edges` is the tighter constraint on the same
picture, not the looser one.

**Then profile before you commit.** `assess_data_shape` is free, local, and needs no credentials —
it never leaves your machine. Run it on a candidate pull, look at what came back, and adjust the
query until the shape is what you meant. Only then spend a metered `list_eligible_charts` call.
Getting the grain wrong is cheap to fix here and expensive to fix after generation. For a second
chart, read each measure's `collinearWithMeasures` and `maxAbsMeasureCorrelation`: two measures
that move together (a correlation near 1) only repeat each other in a partner chart, and one that
doesn't is the one worth showing beside it.

**When several charts genuinely fit, ASK — do not quietly pick one.** A shape that supports a
beeswarm often supports a violin, a box plot and a strip plot too, and those answer subtly
different questions. Say what you found and let the person choose:

> Your data supports a few readings. A **beeswarm** shows every loan as a point (best if you want
> to see individual outliers); a **violin** shows the shape of each group's distribution; a **box
> plot** is the most compact if you mainly want medians and spread. Which do you want?

The exception is when the request already names one, or when only one type survives — then just
build it. The rule exists because silently choosing is how a user ends up with a valid chart that
answers a question they did not ask.

## 1. Scaffold (one command block)

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install d3@7 @bicharts/chart-host
npm install -D @types/d3
```

`d3` is a peer requirement of the **chart**, not of chart-host. `@types/d3` only for
TypeScript. chart-host has no runtime dependencies.

### Blazor WebAssembly (.NET 10)

A chart runs in the browser, so a Blazor app hosts it through JavaScript: **one** small JS
module owns the charts and their cross-filter group, esbuild bundles it during `dotnet build`,
and two Razor components (section 3) hand it the elements to draw into. No chart payload ever
crosses the interop boundary, and nothing here calls a server at run time.

```powershell
dotnet new blazorwasm -o MyCharts -f net10.0
cd MyCharts
dotnet new globaljson --sdk-version 10.0.100 --roll-forward latestFeature
npm init -y
npm install d3@7 @bicharts/chart-host
npm install -D esbuild
```

- **`npm init -y` comes FIRST, in the project folder.** With no `package.json` there, `npm install`
  walks up and installs into the nearest parent folder that has one - a real consequence: it edited
  a `package.json` two levels above the project. If an install is refused or interrupted, run the
  `npm init -y` line again before retrying.
- **`global.json`** pins the build to the newest installed .NET 10 SDK. Without it a machine that
  also has a newer or preview SDK builds with that one.
- These blocks are written against `@bicharts/chart-host` 0.6.33 or later; every name they import
  is exported there.
- npm 11 may warn that esbuild's postinstall is "not yet covered by allowScripts". The build doesn't
  need it: esbuild's platform binary arrives as an ordinary dependency, and the bundling target runs.

Add this to the `.csproj`, inside `<Project>`, whole:

```xml
<PropertyGroup>
  <DefaultItemExcludes>$(DefaultItemExcludes);node_modules/**</DefaultItemExcludes>
</PropertyGroup>
<Target Name="BundleBicCharts" BeforeTargets="BeforeBuild">
  <Exec Command="npm ci" Condition="!Exists('node_modules')" />
  <RemoveDir Directories="wwwroot/js/bic" />
  <Exec Command="npx esbuild Scripts/bic.entry.js --bundle --format=esm --splitting --minify --outdir=wwwroot/js/bic" />
  <ItemGroup>
    <Content Include="wwwroot/js/bic/**" Exclude="@(Content)" />
  </ItemGroup>
</Target>
```

Each line is there because leaving it out fails quietly:

- **`node_modules/**` excluded** - otherwise MSBuild globs thousands of package files into the
  project on every build.
- **The source lives in `Scripts/`, the bundle in `wwwroot/js/bic/`** - `wwwroot` is published
  as-is, and the unbundled entry (bare `import "d3"`) cannot load in a browser.
- **`--splitting`** - map geometry (about 1.3 MB across three assets) stays in chunks fetched
  only when a map asks for one. Without it every page downloads every basemap.
- **The `Content` item inside the target** - the bundle is written DURING the build, after the
  project already globbed `wwwroot`. Without it a clean build (a fresh clone, CI, `dotnet
  publish`) ships no bundle and the page's `import` 404s, while your own incremental builds
  keep working.
- **`RemoveDir`** - chunk names are content hashes, so stale chunks would pile up.

`.gitignore`: `bin/`, `obj/`, `node_modules/`, `wwwroot/js/bic/` (build output). Commit
`package.json` and `package-lock.json` - the target's `npm ci` needs the lock.

## 2. Generate — say what the output must BE, not just what it should look like

Call `generate_chart` once per chart, with `out_dir` so the artifacts land on disk.

The single most important habit: **`prompt` is a suggestion lane.** It steers style and
emphasis, and it competes with the chart lane's own guidance about summarizing. When your
app *depends* on the shape of the output, use the contract parameters instead — they are
authoritative, and `required_columns` is verified in the generated code with an automatic
correction:

| Parameter | Use it when |
| --- | --- |
| `rows_policy: "all"` | every source row must appear — a row-per-entity table, a point per location. Without it the engine may legitimately show a Top-N with an "Other" rollup. |
| `required_columns: [...]` | a field must be visible because your app's interactions depend on it (e.g. the column you colour or filter by). |
| `forbid_derived_columns: true` | your app owns the analytics; the chart is only the display. Stops per-capita / share-of-total / rank columns being added. |

Asking for these in `prompt` prose instead is the most expensive mistake available here: it
routinely costs two or three extra generations, and each one is minutes of wall-clock.

Other parameters worth knowing: `chart_type` (a preference, honored unless the data shape
disqualifies it), `renderer` (a JavaScript or TypeScript project gets D3 without asking, because D3 is what
`@bicharts/chart-host` draws), `width`/`height`, and
`reasoning_mode: "1P"` when you want the cheapest, fastest pass.

**Generation takes minutes.** Fire the calls for independent charts **in parallel** rather
than in sequence — it is the difference between ~4 minutes and ~12.

**Where the files land.** `out_dir` and `csv_path` resolve against the MCP server's working
directory (where the agent session started), not your project, so pass absolute paths.
React: `src/charts/<name>`. Blazor: `wwwroot/charts/<name>` - the page fetches them as static
files. The language auto-detect also reads that working directory, and a .NET project gives it
nothing to read, so for Blazor name `renderer: "D3"` (chart-host runs D3 charts).

**With `out_dir`, the reply leaves the code out** (`include_code: false` is the default then; pass
it explicitly on an older chart-mcp). The code is already on disk, and a reply that carries it can
outgrow your client's tool-output limit: a 68 KB choropleth once came back as "result exceeds
maximum allowed tokens", saved to a file the agent then had to parse. Everything else you need -
`geo.kind`, the plugins, the credits, the paths - stays in the reply.

## 3. Wire it up

Import the generated code as raw text and the sample payload as JSON:

```tsx
import * as d3 from "d3";
import { BicChart, BicChartGroup } from "@bicharts/chart-host/react";
import mapCode from "./charts/map/chart.js?raw";
import tableCode from "./charts/table/chart.js?raw";
import payload from "./charts/map/data.sample.json";

// data.sample.json rows are POSITIONAL arrays; the group wants row OBJECTS.
const columns = payload.columns;
const rows = payload.rows.map(r =>
  Object.fromEntries(columns.map((c, i) => [c.name, r[i]])));
```

### The coordinated dashboard — the recipe, memorized

```tsx
<BicChartGroup columns={columns} rows={rows} point={{ city: "City", state: "StateOrProvince" }}>
  <BicChart id="map"   code={mapCode}   d3={d3} geoKind="north-america" respondsWith="highlight" />
  <BicChart id="table" code={tableCode} d3={d3} />
</BicChartGroup>
```

That is all of it. The map **highlights** (keeps every bubble, dims the rest) and the table
**filters**. Cross-filtering is mutual by default, a chart never filters itself, and
`group.clear()` clears.

Do **not** hand-roll dimming out of the `lch-*` CSS classes. They are an implementation
detail, they are re-applied on every render (so a `classList` poke is lost at the next data
change), and translating row indices across a filtered payload is the hazard below.

> **The row-index hazard.** `__rowIdx__` is a position *within the payload a chart
> received*, not an id in your table. Re-render a chart with filtered rows and its payload
> renumbers from zero, so the same integer denotes a different record. `BicChartGroup` owns
> that mapping. Comparing indices across two charts by hand does not throw — it quietly
> filters to the wrong thing.

### Sizing

Sizing is yours. Give each `<BicChart>` a sized container and pass `width`/`height` in
`options` — a chart handed `undefined` dimensions draws nothing visible.

### Blazor - the same group over JS interop

No React here, so the group is chart-host's framework-free `createChartGroup` - the object
`<BicChartGroup>` wraps, with the same rules (a chart never filters itself, mutual by default,
`respondsWith: "highlight"` keeps every mark). Three files, copied whole. The gestures are
chart-host's, with nothing to wire: a click selects, clicking the same mark again clears,
Ctrl/Cmd/Shift-click adds or removes one mark, and a click on empty canvas clears.

`Scripts/bic.entry.js` - the only JavaScript the app owns:

```js
import * as d3base from "d3";
import { assembleD3, createChartGroup, createChartHost, loadGeo } from "@bicharts/chart-host";

const d3 = assembleD3(d3base);                           // + any plugins a chart's contract names
const at = (path) => new URL(path, document.baseURI);   // honours <base href> on a sub-path host
async function text(path) {
    const r = await fetch(at(path));
    if (!r.ok) throw new Error(`${path}: HTTP ${r.status}`);
    return r.text();
}
// Draw for the canvas the chart actually sits on - the nearest opaque background behind it. A page
// that follows the reader's light/dark preference switches that background; a light-only page keeps
// it light, and the chart's text stays dark there even when the reader's system is dark.
function canvasOf(el) {
    for (let e = el; e; e = e.parentElement) {
        const m = getComputedStyle(e).backgroundColor.match(/[0-9.]+/g);
        if (m && (m.length < 4 || +m[3] > 0.5)) return m.slice(0, 3).map(Number);
    }
    return [255, 255, 255];
}
const theme = (el) => {
    const [r, g, b] = canvasOf(el);
    return 0.2126 * r + 0.7152 * g + 0.0722 * b < 128
        ? { themeFg: "#f3f4f6", themeBg: `rgb(${r}, ${g}, ${b})`, themeAccent: "#118dff" }
        : { themeFg: "#252423", themeAccent: "#118dff" };
};

// ONE group over ONE source table: a data.sample.json's positional rows, as objects.
export async function createGroup(sampleUrl, dotnet) {
    const sample = JSON.parse(await text(sampleUrl));
    const cols = sample.columns;
    const rows = sample.rows.map(r => Object.fromEntries(cols.map((c, i) => [c.name, r[i]])));
    const group = createChartGroup(cols, rows);
    const off = group.onChange(sel => dotnet.invokeMethodAsync("OnSelectionChanged", sel.sourceId, [...sel.rows]));
    return {
        async mount(el, spec) {                          // spec: { id, dir, height, respondsWith, geoKind }
            const code = await text(`${spec.dir}/chart.js`);   // TEXT: chart-host compiles it and injects d3
            if (spec.geoKind) await loadGeo(spec.geoKind);     // BEFORE the first render: render() is synchronous
            const opts = spec.respondsWith ? { respondsWith: spec.respondsWith } : {};
            const host = createChartHost(el, {
                code, d3, geoKind: spec.geoKind || undefined,
                data: group.memberPayload(spec.id, opts).payload,
                options: { width: Math.floor(el.getBoundingClientRect().width), height: spec.height, ...theme(el) },
            });
            host.render();
            const member = group.attach(spec.id, host, opts);  // its clicks publish SOURCE rows
            let w = Math.floor(el.getBoundingClientRect().width);
            const ro = new ResizeObserver(() => {         // LATER resizes only; the first size is measured above
                const now = Math.floor(el.getBoundingClientRect().width);
                if (now > 0 && now !== w) { w = now; host.setOptions({ width: now }); }
            });
            ro.observe(el);
            const mq = matchMedia("(prefers-color-scheme: dark)");
            const onScheme = () => host.setOptions(theme(el));   // the page restyled itself; follow it
            mq.addEventListener("change", onScheme);
            let alive = true;
            return {
                setOptions(patch) { if (alive) host.setOptions(patch); },   // live restyle, never a regeneration
                destroy() {
                    if (!alive) return;
                    alive = false;
                    ro.disconnect();
                    mq.removeEventListener("change", onScheme);
                    member.detach();
                    host.destroy();                       // stops animation timers too
                },
            };
        },
        clear() { group.clear(); },
        dispose() { off(); group.destroy(); },
    };
}
```

`Components/BicChartGroup.razor` - holds the JS group; its children wait for it:

```razor
@implements IAsyncDisposable
@inject IJSRuntime JS

<CascadingValue Value="this" IsFixed="true">
    @ChildContent
</CascadingValue>

@code {
    /// <summary>The data.sample.json whose rows are the one source table every chart draws from.</summary>
    [Parameter, EditorRequired] public string Source { get; set; } = "";
    [Parameter] public RenderFragment? ChildContent { get; set; }
    /// <summary>The chart that made the selection (null when cleared) and its SOURCE rows.</summary>
    [Parameter] public EventCallback<(string? SourceId, int[] Rows)> OnSelection { get; set; }

    readonly TaskCompletionSource<IJSObjectReference> ready = new(TaskCreationOptions.RunContinuationsAsynchronously);
    IJSObjectReference? module, group;
    DotNetObjectReference<BicChartGroup>? self;

    public Task<IJSObjectReference> Ready => ready.Task;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (!firstRender) return;
        try
        {
            module = await JS.InvokeAsync<IJSObjectReference>("import", "./js/bic/bic.entry.js");
            self = DotNetObjectReference.Create(this);
            group = await module.InvokeAsync<IJSObjectReference>("createGroup", Source, self);
            ready.TrySetResult(group);
        }
        catch (Exception e) { ready.TrySetException(e); throw; }
    }

    [JSInvokable]
    public Task OnSelectionChanged(string? sourceId, int[] rows) => OnSelection.InvokeAsync((sourceId, rows));

    public async Task ClearAsync()
    {
        if (group is not null) await group.InvokeVoidAsync("clear");
    }

    public async ValueTask DisposeAsync()
    {
        try
        {
            if (group is not null) { await group.InvokeVoidAsync("dispose"); await group.DisposeAsync(); }
            if (module is not null) await module.DisposeAsync();
        }
        catch (JSDisconnectedException) { }   // Blazor Server: the circuit is already gone
        self?.Dispose();
    }
}
```

`Components/BicChart.razor` - one chart:

```razor
@implements IAsyncDisposable

<div @ref="el" class="bic-chart" data-bic-chart="@Id" style="height:@(Height)px"></div>

@code {
    [CascadingParameter] public BicChartGroup Group { get; set; } = default!;
    [Parameter, EditorRequired] public string Id { get; set; } = "";
    /// <summary>The generate_chart out_dir under wwwroot, e.g. "charts/map".</summary>
    [Parameter, EditorRequired] public string Dir { get; set; } = "";
    [Parameter] public int Height { get; set; } = 420;
    /// <summary>"highlight" keeps every mark and dims the rest; the default filters.</summary>
    [Parameter] public string? RespondsWith { get; set; }
    /// <summary>A map's geo.kind from the generate_chart result (e.g. "us-state-name").</summary>
    [Parameter] public string? GeoKind { get; set; }

    ElementReference el;
    IJSObjectReference? chart;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (!firstRender) return;
        var group = await Group.Ready;
        chart = await group.InvokeAsync<IJSObjectReference>("mount", el,
            new { id = Id, dir = Dir, height = Height, respondsWith = RespondsWith, geoKind = GeoKind });
    }

    /// <summary>A live restyle: merged into the chart's options and repainted.</summary>
    public async Task SetOptionsAsync(object patch)
    {
        if (chart is not null) await chart.InvokeVoidAsync("setOptions", patch);
    }

    public async ValueTask DisposeAsync()
    {
        try
        {
            if (chart is not null) { await chart.InvokeVoidAsync("destroy"); await chart.DisposeAsync(); }
        }
        catch (JSDisconnectedException) { }
    }
}
```

Add `@using MyCharts.Components` to `_Imports.razor`, then on a page:

```razor
<BicChartGroup Source="charts/map/data.sample.json" OnSelection="OnSelection" @ref="group">
    <button data-bic-clear disabled="@(selected == 0)" @onclick="() => group!.ClearAsync()">Clear</button>
    <BicChart Id="map"  Dir="charts/map"  Height="480" RespondsWith="highlight" GeoKind="us-state-name" @ref="map" />
    <BicChart Id="bars" Dir="charts/bars" Height="360" />
</BicChartGroup>

@code {
    BicChartGroup? group;
    BicChart? map;
    int selected;
    void OnSelection((string? SourceId, int[] Rows) s) { selected = s.Rows.Length; StateHasChanged(); }
    // a restyle control: await map!.SetOptionsAsync(new { palette = new[] { "#1b9e77", "#d95f02" } });
}
```

- **Source** - every chart generated from the same CSV carries the same rows, so any chart's
  `data.sample.json` is the source table. Take the map's: it also holds the `__geo*` columns
  the tool resolved.
- **Restyle** - a chart honours only the options its code reads, so a live restyle control has to
  set one of those. The generate result's `optionsRead` lists them for that chart, and its
  `integrationContract` says what each does. The common ones: `palette` (a categorical colour
  list), `colorScaleLow`/`colorScaleHigh` (a sequential ramp's ends, what a choropleth reads),
  `maxMapPoints` (a point map's cap). An option not in `optionsRead` does nothing on that chart.
- **Hover drop lines** - a 3D scatter can draw, on hover, a line from a point to each back wall of
  its cube, so the point's three values read off the gridlines. It is off unless someone wants it:
  name it in the request ("with drop lines") and it starts on; otherwise ASK THE USER ONCE whether
  they want it, then pass `hoverDropLines: true` in the chart's options on a yes. The result's
  `dropLines` says which applies: `ask` carries the question, `on`/`off` means the user already
  answered (the client asked them) and the generated component applies it. Keep the answer when
  the chart is regenerated; never ask again.
- **GeoKind** - the result's `geo.kind` (the text reply's `Geo:` line). Without it a map draws
  its marks over no land. `loadGeo` takes that geometry from chart-host itself, so the
  `data.geo.json` the tool also writes beside a map is the same shapes as a file: the page never
  fetches it.
- **Interop only in `OnAfterRenderAsync(firstRender)`** - an `ElementReference` is empty until
  the element is rendered, and under prerendering there is no JavaScript yet. The components
  above already do this, so they run unchanged under Blazor Server or Auto.
- **Dispose** - `destroy` stops a chart's timers and detaches it from the group; skip it and an
  animated chart keeps ticking after you navigate away.
- **Measure, then mount** - the chart takes the element's width once it is laid out, so give
  `.bic-chart` its width in CSS (a block element in a sized column); the height is the
  `Height` parameter. A `display:none` container measures 0 and draws nothing.
- **Light and dark** - `theme(el)` reads the background the chart sits on, so the chart follows the
  PAGE. For a page that follows the reader, switch the page's own colours in CSS
  (`@media (prefers-color-scheme: dark) { ... }` on `body` or the chart's card) and the charts follow
  on the next `change` of the media query.

## 3a. A Microsoft Fabric App (the Rayfin `dataapp` template)

A Fabric App built from Rayfin's `dataapp` template is a React app over a semantic model. The golden
path above applies unchanged; these are the places a data app differs, each one measured on a real
build. Everything here layers on the template's own AGENTS.md and skills - follow them for everything
this doesn't cover. The template's visuals skill asks before using another chart library: a request
for RegiaBI charts is that consent.

**0. Once, right after scaffolding: `setup_fabric_app`.** It installs `@bicharts/chart-host` and `d3`, adds
`src/lib/bic-chart-options.ts` (`useBicChartOptions()`: the app's Fabric theme - palette, foreground,
background - as chart options, so charts follow light and dark) and `src/components/bic-chart-panel.tsx`
(`<BicChartPanel result={useSemanticModelQuery(...)}>`: a box that measures itself and shows the query's
loading, error and empty states), and ignores `.bic/`. It also puts every `@microsoft/rayfin-*` package on one version -
npm's latest, the release `rayfin init` installs, so run it before `rayfin init` and init has nothing left to split
(`rayfin init` installs the latest of only some packages, which leaves two copies of `rayfin-auth` on any other version).
Its reply lists every `package.json` range it changed and names the Rayfin version installed; an app already on npm's
latest is left alone. It fixes the template's own `scripts/validate-visual.spec.mjs`, which fails under ESM on a fresh
scaffold (its reply says so). It reads the theme through `src/lib/bic-css-theme.ts` - Microsoft's own `useCssTheme`
logic on `@microsoft/fabric-visuals-core` - and points the template's `src/main.tsx` at it, because importing the hook
from `@microsoft/fabric-visuals` bundles Vega into an app that doesn't chart with it (about half its download); the
reply names the one line it changed. It adds
`.agents/skills/regiabi-charts/SKILL.md` beside the template's skills, for agents that read that folder. It never
overwrites a file, and its reply ends with a worked example. Building notes on marks or saved scenarios? Once Rayfin's
data service is on (the notes recipe below, step a), run it again with `features: ["notes", "scenarios"]`: it writes
those files finished.

**1. Pull each chart's rows at the chart's GRAIN, with a query written for that chart.** One row per
country for a country map, one per route for a flow map, one per month for a trend. The template's
writing-phase command is `npx fabric-app-data query <alias> --query '<DAX>'`. Don't pull a fact
table and aggregate it in the browser: the Execute Queries API refuses a response over 15 MB, and a
column repeated on every fact row (a country's population on each sale) sums to nonsense.

**2. Profile and generate from THE WHOLE RESULT of that query - every row, never a few pasted in.**
Eligibility reads row, region and category counts. A bivariate country map needs at least 24
countries, so the same 40-country query shown as 8 sample rows hides it, and nothing fails - you are
simply offered other charts. Save the result to a file and pass `csv_path`, or pass all of it as
`data`.

**3. Rename the query's columns to plain names first, and keep the map.** Results come back as
`Geography[CountryIso3]` and `[Average Daily Rate]`. Generate on `CountryIso3` and
`AverageDailyRate`, and apply the SAME rename to the live rows at run time - the chart reads columns by
name. Pass the model's metadata as you rename: `isMeasure`, `format` and `description` (from
`INFO.VIEW.MEASURES()` / `INFO.VIEW.COLUMNS()`) go on each column in `data`, which is the context a
Power BI visual gets for free. In `INFO.VIEW.COLUMNS()` a column's name is `[Name]`, not `[Column]`.

**Steps 1-3 are one call: `query_semantic_model`.** Give it the chart's DAX (`dax`, or a `.dax` file) and a
`name`; it runs the query through the project's own `fabric-app-data` CLI, renames the headers (dropping the
table and any symbols: `[Occupancy %]` becomes `Occupancy`), saves every row to `.bic/data/<name>.csv`,
reads each column's role, format and description from the model (`INFO.VIEW.MEASURES()` / `INFO.VIEW.COLUMNS()`),
and profiles the result. Its reply ends with the arguments for the next two calls - `csv_path`, `measures`,
`dimensions`, `formats`, `descriptions` - so pass them to `list_eligible_charts` and `generate_chart` as they
are. The CLI returns at most 1,000 rows; the tool says so when it cut a result, and the fix is a query at the
chart's grain.

**4. Ask before choosing, then choose what answers the question.** `list_eligible_charts` on that
data. The catalogue goes past the template's own charts, and these are the ones people describe
without naming:

| the request sounds like | ask for |
| --- | --- |
| two measures compared across countries or states | Bivariate World (or USA) Choropleth |
| movement, routes or flows between places | Origin-Destination Flow Map |
| a forecast the reader steers with an assumption (a rate, a price change), against a goal | What-if projection - its sliders are drawn inside the chart; there is no slider to build |
| how far apart the outcomes of several assumptions land | What-if scenarios |

If the list doesn't offer what you expected, read its notes: it names a type refused on a count
when your table is small, and it says when `name_filter` hid description matches.

A chart that draws its own rate control needs the history and the goal, not the model's what-if parameter:
leave the parameter's table (a `'Price Change'[Price Change]` column, say) out of its query. Grouping by it repeats
every period once per parameter value - 12 months by 17 values is 204 rows for 12 points - and the chart's own
slider sets the rate anyway.

**5. Render live rows through the group, with the binding the generation returned.** The template
queries at run time (`useSemanticModelQuery`), so the payload is rebuilt from live rows every time -
never hand-built. **With `out_dir`, a D3 chart arrives with `chart.component.tsx` beside `chart.js`: a component that
already does all of this** - the columns, the binding (`geo`, `point`, `destination`), the geometry kind, the
d3 plugins the chart calls, and `rowsFromTable`, which maps a DAX result's `Table[Column]` and `[Measure]`
headers onto the chart's plain names. Use it as it is:

```tsx
// The component is named after the chart type ("Bivariate World Choropleth" -> BivariateWorldChoroplethChart).
import { BivariateWorldChoroplethChart, rowsFromTable } from "@/charts/occupancy-map/chart.component";

import { BicChartPanel } from "@/components/bic-chart-panel";
import { useBicChartOptions } from "@/lib/bic-chart-options";
import QUERY from "@/queries/overview/occupancy-map.dax?raw";   // the template's rule: DAX lives in .dax files

const result = useSemanticModelQuery({ connection: "<alias>", query: QUERY });  // the DAX it was generated on
const options = useBicChartOptions();
const country = useBicFilter("CountryIso3", { label: "Country", display: "CountryName" });   // step 6
<BicChartPanel result={result}>
  {(table, size) => (                                // table = the live { columns, rows } result
    <BivariateWorldChoroplethChart table={table} {...size} options={options} selects={country} />
  )}
</BicChartPanel>
```

It is rewritten with the chart on every generation, so keep app logic outside it. What it does, if you
host by hand instead - `generate_chart` returns `binding`; pass all three:

```tsx
import * as d3 from "d3";
import { BicChart, BicChartGroup } from "@bicharts/chart-host/react";
import code from "@/charts/occupancy-map/chart.js?raw";
// Saved from the generate_chart result: columns = data.columns, binding = binding, geoKind = geo.kind.
import { columns, binding, geoKind } from "@/charts/occupancy-map/meta";

// The SAME rename the chart was generated on.
const RENAME = { "Geography[CountryIso3]": "CountryIso3", "Geography[CountryName]": "CountryName",
                 "[Average Daily Rate]": "AverageDailyRate", "[Occupancy %]": "Occupancy" };

function CountryMap({ table, onPick }) {          // table = the live query result
  const rows = table.rows.map(r => Object.fromEntries(
    table.columns.map((c, i) => [RENAME[c.name] ?? c.name, r[i]])));
  return (
    <BicChartGroup columns={columns} rows={rows}
                   geo={binding.geo} point={binding.point} destination={binding.destination}>
      <BicChart id="map" code={code} d3={d3} geoKind={geoKind}
                options={{ width, height }}
                onSelect={idxs => onPick(idxs.length ? rows[idxs[0]].CountryIso3 : null)} />
    </BicChartGroup>
  );
}
```

- **`columns`** can be the generation payload's columns as they are: the group drops any
  `__rowIdx__` / `__geo*` column it is handed and builds its own from the live rows and the binding.
- **`geoKind` comes from the result's `geo.kind`**, not from `binding.geo`: a point map has a
  basemap but no join, so its `binding.geo` is null.
- **`<BicChartGroup destination>` needs `@bicharts/chart-host` 0.6.50 or later.** Before it, a flow
  map's far end is not built.

**Layout: a chart is never clipped or squashed.** Charts sit in cards, and a card is where they get cut off.
- Render every chart inside `BicChartPanel`. It sizes the chart from its cell, never from the chart.
- Give each chart its minimum: `minChartWidth` and `minChartHeight` (a world map about 560 x 300; a time series wants
  height). Below that the panel scrolls; it never squashes or clips the chart. That scroll is a safety net, not a layout:
  inside a page that already scrolls it's a second scrollbar, which reads as broken - size every chart's cell at or above
  its minimum so it never shows.
- Take the actual size from the layout - percent, flex or grid `fr` - and never a fixed height smaller than the chart's
  minimum.
- Put `min-h-0` on every flex child between a card and its chart (`min-w-0` in a row). A flex item's default
  `min-height: auto` lets it grow past its card, and the card's `overflow-hidden` slices off the rest.
- A chart with controls drawn inside it (sliders, steppers, a reset) gets its card's full width. Put a side panel that
  goes with it - saved scenarios, notes, a form - below the chart, not beside it: beside it, the chart's cell drops
  under its minimum and the panel scrolls.
- When you stack a panel below a chart, don't keep a fixed card height that can't hold both. Prefer letting the card grow
  (a `min-h-*` in place of its `h-*`); only where the card must stay fixed, give the panel below `min-h-0 flex-1
  overflow-auto` so it scrolls inside the card. The chart's own cell keeps a DEFINITE height (as the template's
  app-validation skill asks); only the card around it and the panel below may grow.
- One frame per chart: a RegiaBI chart sits in your card. The template's `VisualContainer` frames its `VegaVisual`
  charts; don't put a second frame around one of ours.
  Give the chart's cell at least the chart's `minChartHeight` (`h-72` is 288px, under a 300 minimum).

**6. Filter the page with `useBicFilter` - never write a select handler, a toggle or a Clear.** A page filter is
kept by the model's KEY (`CountryIso3`; a route's two ends), never by a row position, so it works across charts fed by
different queries and survives a re-query. Hand it to the chart that sets it with `selects`; the rest of the page
reads it:

```tsx
import { useBicFilter, BicFilterChips } from "@bicharts/chart-host/react";
import { filterQuery, treatAs } from "@bicharts/chart-host/dax";
import FORECAST_DAX from "@/queries/overview/bookings-forecast.dax?raw";

// Once, in the page component, and passed down: each useBicFilter call is its own filter.
const country = useBicFilter("CountryIso3", { label: "Country", display: "CountryName" });
const route = useBicFilter(["OriginCode", "DestinationCode"],
                          { label: "Route", display: ["OriginName", "DestinationName"] });
const forecast = useSemanticModelQuery({ connection: "<alias>",
  query: filterQuery(FORECAST_DAX, treatAs(country, "Geography[CountryIso3]")) });

<BicFilterChips />                                   {/* "Country: Japan ×" - what's filtered, and its clear */}
<BivariateWorldChoroplethChart table={countries} {...size} options={options} selects={country} />
<OriginDestinationFlowMapChart table={routes} {...size} options={options} selects={route}
  filterBy={{ filter: country, columns: ["OriginCode", "DestinationCode"] }} />
```

What the library does, so the page doesn't:
- A click sets the filter. The same click again, or a click on empty canvas, clears it. Ctrl-click adds a key.
- `country.clear()` (a button, a chip's ×) clears the filter AND the chart's marks.
- New rows keep a selection whose key is still drawn and clear one whose key is gone - reported, never silent.
- `country.value`, `country.values`, `country.keys`, `country.row` (the clicked row), `country.active` and
  `country.text` ("Japan", or "Spain → Portugal" for a route) are what the page reads. For any OTHER key - a badge's,
  a lane's end - `country.textOf(key, rows)` names it from a table that carries the display column.
- Another chart whose table carries the key: `filterBy={{ filter: country, columns: [...] }}` on its component - any
  of several columns, for a route that starts or ends there. It's applied inside the component, so the chart redraws
  only when the filter or its data changes. A table WITHOUT the key column can't be filtered that way (it warns): filter
  its query instead.
- A query: `filterQuery(dax, treatAs(country, "Table[Column]"))` puts the filter into the `.dax` file's query the way
  Microsoft's own data app skills do (CALCULATETABLE + TREATAS), escaped; with nothing selected the query is unchanged.
  It adds a filter value and never changes the query's structure, so one `.dax` file serves both. A route filter maps
  each end: `treatAs(route, "Routes[Origin]", "Routes[Destination]")`; `treatAs(route, "Geography[CountryIso3]")` takes
  the route's FIRST column (its origin). Two filters on one column intersect: pass the one that should win.

A Microsoft `VegaVisual` or `DataGrid` on the same page joins the same filter:
`onInteraction={e => fromVegaInteraction(country, e)}` (`fromVegaInteraction` from `@bicharts/chart-host/react`; a page without React takes all of these from `@bicharts/chart-host/page`). It matches
the event's field NAMES to the filter's columns, so name the VegaVisual's column `CountryIso3` too. It's one way: a
page Clear clears the filter, but the VegaVisual keeps its own highlight until it's clicked - that chart owns it. If you take clicks yourself with `onSelectRows` instead, an
empty list means the selection was CLEARED: set your state from exactly what it gives you and never toggle it again.
Anything saved against a mark - a note, a flag - is keyed the same way, never by `__rowIdx__`.

More the page gets from the same filters, each one line:
- A link that opens the page as it was: `useBicUrlState()` in the page component (after its `useBicFilter` /
  `useBicControls` calls) keeps every filter and every chart's controls in `?view=`. `useBicView()` gives `save()` /
  `restore(view)` for a saved view of your own (a user's favourite), and `token()` for a link.
- Linked hover: `useBicFilter("CountryIso3", { hover: true })` - hovering a mark lights the same key's marks on every
  chart bound to that filter (a glow, never a selection).
- A filter that narrows a query to nothing: `<BicChartPanel ... onClearFilters={country.clear}>` says so with a Clear,
  instead of an empty chart. A chart's own empty message: `emptyText` on its component.
- While building: `<BicDevtools />` shows every filter, its keys, why it last changed, and each chart's controls.

**Notes (or flags) on marks, end to end.** People leave a note on a country, a route, a product, and see
it on that mark: the notes live in Rayfin's data service, and the badge is the chart's `annotations`.
This recipe was checked against Rayfin's own docs for Rayfin 1.36.1 (a preview), and its init flags and type-check
edits again on a real build on 1.36.2, the version `setup_fabric_app` installs as this is written; on those it IS the
version-matched source:
follow it as written, and read Rayfin's docs (`node_modules/@microsoft/rayfin-guide/assets/docs/data/`)
only for what it doesn't cover. `setup_fabric_app`'s reply names the version it installed; where that
differs from those, Rayfin's docs win.

*a. Turn the data service on.* The data app template ships it off. Run `setup_fabric_app` first (it puts every `@microsoft/rayfin-*` package on one version:
npm's latest, the release `rayfin init` installs), then:

```bash
npx rayfin init --services auth,data --auth-methods fabric --dialect mssql --overwrite --project-name <id from rayfin/rayfin.yml> .
```

Each flag is needed: without `--overwrite` it prints "Initialization cancelled" and exits 0 with no
reason; without `--project-name` (pass the `id:` already in `rayfin/rayfin.yml`, so nothing is renamed)
it stops with "--project-name is required in non-interactive mode". It rewrites `rayfin/rayfin.yml`
and `.gitignore`, so `git diff` both and put back what's yours (on 1.36.1 it dropped the template's
`src/fabric.generated.ts` line from `.gitignore`). It writes `rayfin/tsconfig.json` - keep its
`outDir`, `.temp/compiled`, which is where the deploy reads the entities from - and no entities. On 1.36.2 it
writes no `rayfin/data/schema.ts` either: don't write one by hand - `setup_fabric_app` with `features` (step c) writes it
when `rayfin/rayfin.yml` shows the data service on.

*b. Make it type-check.* Three edits, each needed on the data app template:
- `rayfin/tsconfig.json`, in `compilerOptions`: `"tsBuildInfoFile": ".temp/tsconfig.tsbuildinfo"` and
  `"allowImportingTsExtensions": false` (it inherits both from the root: TS6377 and TS5096 otherwise).
- the root `tsconfig.json`: `"references": [{ "path": "./rayfin" }]`.
- type the client with your schema: `RayfinClient<AppSchema>` in `src/lib/rayfin-client.ts`
  (`import type { AppSchema } from "../../rayfin/data/schema";`) and in the constructor of
  `src/services/rayfin-auth.service.ts` (TS2345 otherwise).

Check with `npx tsc -b`; a bare `tsc --noEmit` reports TS6305 until the `rayfin/` project is built. If a
TS2345 names two copies of `rayfin-auth`, something installed only some Rayfin packages: run
`setup_fabric_app` again.

*c. The files: `setup_fabric_app` with `features: ["notes"]`.* With the data service on, it writes them finished and
never rewrites them:
- `rayfin/data/MarkNote.ts`, registered in `rayfin/data/schema.ts` - keyed by the mark's model key, never a row
  position: `chart`, `markKey`, `markKeyTo` (a route's other end), `body`, `author`, `createdAt`. Every entity it
  writes has an explicit permission decorator and a `max` on every `@text` (MSSQL gets `NVARCHAR(MAX)` without one,
  which can break the deployed GraphQL schema).
- `src/lib/bic-notes.ts`: `useMarkNotes(author)` - every note (paged with `.executePaginated()`: one query returns at
  most 100 and doesn't say there are more), an `error` when they can't load, and `add(chart, key, body)`, which saves
  with `.create(` and reloads, and REJECTS when the save fails. The author is the signed-in user:
  `useAuth().session?.user?.email`.
- `src/components/bic-notes-dialog.tsx`: `NotesDialog`, over the page through `createPortal`, never inside the
  chart's card (a card has a fixed height and `overflow-hidden`, and clips it). The reader's text is cleared only once
  the save succeeds; a failure is shown with the text kept.

*d. Badges on the marks: pass `annotations`, never measure the chart.* The host draws each badge on its key's mark
after every render and keeps it there through filters, re-queries and zooms; it never reads a row position. A route is
keyed by BOTH of its ends with `where` - no key column to add, no query to change and no chart to regenerate:

```tsx
import { noteBadges, notesFor, badgeKey } from "@bicharts/chart-host/react";

import { useAuth } from "@/hooks/auth.context";

const { notes, add } = useMarkNotes(useAuth().session?.user?.email);
<BivariateWorldChoroplethChart table={table} {...size} options={options} selects={country}
  annotations={noteBadges(notes, { chart: "occupancy-map", columns: "CountryIso3" })}
  onAnnotationClick={a => setOpen({ chart: "occupancy-map", columns: "CountryIso3", key: badgeKey(a) })} />
// a route by both ends: noteBadges(notes, { chart: "routes", columns: ["OriginCode", "DestinationCode"] })
// gives each route { where: { OriginCode: "ESP", DestinationCode: "PRT" }, label, title }

{open && <NotesDialog title={country.textOf(open.key, countryRows)} onClose={() => setOpen(null)}
  notes={notesFor(notes, open.key, { chart: open.chart, columns: open.columns })}
  onAdd={body => add(open.chart, open.key, body)} />}
// open = { chart, columns, key }: keep the chart's key columns with it - a route's notes need both ends.
```

Open a mark's notes from `onAnnotationClick` (`badgeKey(a)` is its key), and its first note from your own "Add a
note" action on the selected mark - `country.keys[0]` is its key, `country.text` its name. Don't read `data-row-idx`
or bounding boxes to place anything: that breaks on the next redraw, and this does it for you.

The files use the template's own tokens; restyle them as you like. The table and the permissions are created on
the next `npx rayfin up`, and every later `rayfin up` applies entity changes.

Don't stand up a backend to test notes; the deploy creates the tables. Finish with `npx tsc -b` and the tests, and say the notes go live on deploy.

**7. A chart's own controls are page state: `useBicControls`.** A What-if chart's sliders (a rate, a horizon) are drawn
inside the chart - build no slider beside it. Give the chart `controls` and read them on the page:

```tsx
import { useBicControls } from "@bicharts/chart-host/react";
import { ScenariosPanel } from "@/components/bic-scenarios-panel";   // setup_fabric_app features: ["scenarios"]

const scenario = useBicControls();
<WhatIfProjectionChart table={table} {...size} options={options} controls={scenario} />
<ScenariosPanel chart="bookings-forecast" controls={scenario} author={useAuth().session?.user?.email} />
```

`scenario.values` holds every slider's value from the first draw - no slider has to move before a scenario can be
saved - and `scenario.summary` is the chart's own label and readout for each ("price change per month +2.0% ·
horizon 12 months"), so a saved scenario shows the unit the chart shows. `scenario.set(saved)` puts one back; `scenario.reset()`
returns to the chart's defaults. Don't read `uiState.knobs` or the chart's source, and never save a default you made
up. `features: ["scenarios"]` writes the `SavedScenario` entity, `useSavedScenarios` and `ScenariosPanel` (save under a
name, list, apply, and which one is applied now - `applied`, the saved scenario whose values the chart shows; a failed
save keeps what was typed).

**8. What stays the template's job - this is additive, never a replacement:** sign-in, the query hooks
(`useSemanticModelQuery`; a page filter feeds it through `filterQuery`), the theme (`useCssTheme`, which
`bic-chart-options.ts` builds on), the app's database (Rayfin's data service), `VisualContainer` headers, the template's
own `VegaVisual` charts and `DataGrid` (which join a page filter through `fromVegaInteraction`), its skills, and
deployment. The chart is a function you render inside them.

**9. Check every chart, then deploy.** Check every chart locally with this server's `check_chart` tool,
at the size the panel will give it - by default it reads the chart's `minChartWidth` and `minChartHeight`
off its `BicChartPanel`, so pass no size unless you mean another one. A `WARNING` line in its reply is a
label cut off at that size: deal with it before you call the chart done. The reply says whether more room fixes it.
Never edit the generated chart code by hand: if something's wrong with a generated chart, regenerate it
(`generate_chart` with a prompt describing the fix) or report it.
Then run `check_app` once on the whole app, before every deploy: it reads how the page uses the library (selection
through `useBicFilter` + `selects`, a chart's own controls through `useBicControls`, notes through `annotations`, every
chart in a `BicChartPanel`), draws every chart together, and compares the stack with npm. Each finding names the file,
the line and the call to use instead; fix the ERRORs in the app's own files before deploying.
You don't need to deploy to check a chart. The template's `validate:visual` checks `VegaVisual` factories; a
RegiaBI chart's check is `check_chart` - don't list it there. `check_chart` and `check_app` check the charts and how
the app uses them; the template's app-validation in the portal embed still applies. Then `npx tsc -b` and the tests. To test the page's behaviour,
`@bicharts/chart-host/testing` has a reader's gestures - `clickMark`, `clickEmpty`, `selectedRows` - for the checks
worth keeping: a second click on a mark clears its filter, and a page Clear clears the chart's marks. Deploying (`npx rayfin up`), on Rayfin 1.36.1:
- Signed in? `npx rayfin login status` (a subcommand, not a `--status` flag); `npx rayfin login` if not.
- Name the workspace: `npx rayfin up -w "<workspace>"`. Left out, it deploys to My Workspace.
- The Fabric item is named by `id:` in `rayfin/rayfin.yml`, not `name:`. Two projects with the same
  `id:` in one workspace deploy into one item: give each its own `id:`, or deploy with
  `--item-name <unique-name>`. Pass `--yes` to reuse an existing item only when you know it's yours.
- Deleted the app outside Rayfin? `rayfin/.deployments.json` still names the deleted item, so the next
  `rayfin up` fails with 404 Not Found. Move that file aside and deploy again.

What it costs: only the calls that author a chart (generate_chart, and the eligibility lists beyond a free hourly allowance) draw on the account. The chart code they write is the user's to keep and ship: it needs no licence, account, credits or connection to us to run, view or deploy, for any number of viewers.

## 4. Things that will otherwise cost you time

- **Place names, not coordinates.** If your data has city/state/ZIP but no lat/lon, pass a
  `point` binding and the host resolves coordinates. Never hardcode a city→coordinate table
  from memory: it breaks silently off famous cities and reports nothing. The host reports
  what it achieved in `options.geoPoint` — `precisionCounts` (rows per tier) and
  `unplaced` — so a chart can annotate honestly instead of implying exact positions.
- **Both code forms work.** `wrap: "esm"` appends `export { render };` so the file can be
  imported; chart-host strips the clause before compiling. Either form runs anywhere.
- **D3 plugins.** Some charts call `d3.sankey`, `d3.hexbin` etc., which live in separate
  packages. `requiredD3Plugins(code)` tells you which to install **before** rendering;
  `requiredD3Plugins` on the MCP result says the same thing.
- **Read `integrationContract`** in the MCP result if you need anything beyond this file —
  it describes the exact payload and option shape for the chart you just generated, and the
  framework-free wiring (`createChartGroup` + `createChartHost` + `attach`).
- **Basemap geometry.** The React `<BicChart>` fetches the geometry for its `geoKind`
  itself (chart-host ≥ 0.4.1) and re-renders when it lands. To skip the one-frame basemap
  pop-in, `await loadGeo("north-america")` (from `@bicharts/chart-host`) before the first
  mount. Outside React, `render()` is synchronous by contract, so that preload is
  **required** — a cold cache draws marks over no land, with only a console warning.

## 5. Check it

Verify in a real browser, not jsdom — a chart can pass a DOM-shape assertion and paint
nothing. Assert marks carry `data-row-idx`, that clicking one changes the other chart, and
that `Clear` restores both. Each chart answers a selection in its own way, so assert per role:

| Chart | What it shows while something is selected |
| --- | --- |
| the one you clicked | `lch-has-selection` on its container, `lch-mark-selected` on the marks you picked; every mark stays |
| a `respondsWith: "highlight"` member | the same two classes, on the sibling's rows; every mark stays |
| a filtering member (the default) | NEITHER class: it redraws with only the selected rows, so its count of row marks drops |

Count row marks by DISTINCT single `data-row-idx` values. A legend swatch or an axis label
carries the same attribute with a comma-joined list of every row it covers (`"1,2,4,5"`), and a
one-row table's header cells carry that row's index too, so `[data-row-idx]` alone over-counts.
Read the selection from those classes rather than from paint: dimming is opacity on most charts,
but a chart may answer a highlight by recolouring its selected marks instead. If a check fails,
suspect the probe before the product: query `fillOpacity` as well as `opacity` when testing
dimming, click with `force` (a chart's zoom or tooltip overlay can sit over a mark at some
sizes), and match buttons by role rather than by text that also appears in hint copy.

### Checking a Blazor app

Install the browser driver INTO the project - its `package.json` is there, so nothing walks up - and run
the check below against the running app. Never `npm install` in a fresh scratch folder for this:
`npm init -y` refuses a folder name that starts with a dot (`.verify` fails as "Invalid name"), and
the install that follows then writes into the nearest parent folder that has a `package.json`.

```powershell
npm install -D playwright
npx playwright install chromium
dotnet run --urls http://localhost:5080      # leave it running; a second terminal for the next line
node check.mjs http://localhost:5080/
```

`check.mjs` reads the two `[data-bic-chart]` containers and the `[data-bic-clear]` button the blocks above carry,
and asserts each chart by its role in the table: the clicked chart shows its mark, the sibling answers
(classes for a highlight member, fewer rows for a filtering one), Clear restores both, Ctrl-click adds and
removes one mark, and no page error is raised. It passes unchanged on a map + bubble pair and a map +
table pair.

```js
// node check.mjs http://localhost:5080/  - drive the running app in headless Chromium and check each chart
// answers a selection the way its role says (see the table above). Exits 1 on any failure.
import { chromium } from "playwright";

const url = process.argv[2] ?? "http://localhost:5080/";
const browser = await chromium.launch();
const page = await browser.newPage({ viewport: { width: 1280, height: 1000 } });
const errors = [];
page.on("pageerror", e => errors.push(String(e)));
let failed = 0;
const check = (ok, what) => { console.log((ok ? "PASS  " : "FAIL  ") + what); if (!ok) failed++; };

await page.goto(url, { waitUntil: "networkidle" });
await page.waitForFunction(() => {
    const cs = [...document.querySelectorAll("[data-bic-chart]")];
    return cs.length >= 2 && cs.every(c => c.querySelector(".d3-mark[data-row-idx]"));
}, null, { timeout: 30000 });

// Per chart: its distinct SINGLE-row marks (a legend swatch or a header lists several rows, comma-joined),
// whether it shows a selection, and how many marks are lit.
const state = () => page.$$eval("[data-bic-chart]", cs => cs.map(c => {
    const rows = [...c.querySelectorAll(".d3-mark[data-row-idx]")].map(m => m.getAttribute("data-row-idx"))
        .filter(v => v && !v.includes(","));
    return { rows: [...new Set(rows)], selected: c.classList.contains("lch-has-selection"),
             lit: c.querySelectorAll(".lch-mark-selected").length };
}));
const click = (chart, row, modifiers = []) =>
    page.locator("[data-bic-chart]").nth(chart).locator(`.d3-mark[data-row-idx="${row}"]`).first()
        .click({ force: true, modifiers });                 // force: an overlay may sit over a mark
const settle = () => page.waitForTimeout(600);
const clear = async () => { await page.click("[data-bic-clear]"); await settle(); };

const rest = await state();
for (const [src, dst] of [[0, 1], [1, 0]]) {
    await click(src, rest[src].rows.at(-1));
    await settle();
    const now = await state();
    check(now[src].selected && now[src].lit > 0, `chart ${src + 1} shows the mark it was clicked on`);
    // A highlight member shows the classes; a filtering member redraws with fewer rows.
    check(now[dst].selected || now[dst].rows.length < rest[dst].rows.length, `chart ${dst + 1} answers it`);
    await clear();
    const back = await state();
    check(!back[src].selected && back[dst].rows.length === rest[dst].rows.length, `Clear restores both`);
}

await click(0, rest[0].rows[0]); await settle();
const one = (await state())[0].lit;
await click(0, rest[0].rows[1], ["ControlOrMeta"]); await settle();
check((await state())[0].lit > one, "Ctrl-click adds a mark");
await click(0, rest[0].rows[1], ["ControlOrMeta"]); await settle();
check((await state())[0].lit === one, "Ctrl-click on a selected mark removes just that one");
await clear();

check(errors.length === 0, "no page errors" + (errors.length ? ": " + errors.join(" | ") : ""));
await browser.close();
process.exit(failed ? 1 : 0);
```

# What shape feeds which chart

**Generated from the chart catalogue - do not hand-edit.**
Regenerate with `python tools/gen_chart_shapes.py`; a test fails the build if this
file and the catalogue disagree, so it cannot quietly go stale.

Use it to AIM before you write a query: pick the chart you want, read what it needs,
then pull a table with that shape. `assess_data_shape` (free, local) will tell you
whether the pull you built actually has it.

**Read the LIMITS column carefully - the units differ.** A sankey is capped on
EDGES and a chord on NODES, so `<=80 edges` is a tighter constraint than `<=30 nodes`
on the same picture, not a looser one.

| chart | needs | limits | renderers | good for |
| --- | --- | --- | --- | --- |
| 3D scatter plot | 3 continuous | needs 700x480 px | D3, Plotly | Three continuous measures whose joint structure is the question, on a tile large enough to turn. To read exact values off three or more measures, Pair plot (scatter matrix) and Parallel coordinates are both more precise |
| Animated bar chart | 1 categorical, 1 continuous, 5+ time points | <=40 categories, needs 360x300 px | D3 | Requires one categorical, one measure and a time column with 5 or more ordered periods; best for a leaderboard that changes over time (top products by year, teams by season, countries by decade) |
| Animated treemap | 2 categorical, 1 continuous, 3+ time points | needs 800x560 px | D3 | Requires a 2-level categorical hierarchy plus an additive measure observed at 3 or more ordered time points |
| Animated World (Bubbles) | 3+ time points, lat/lon | needs 400x320 px | D3 | Best for a measure per place observed at 3 or more ordered time points - revenue by country by year, openings by city by quarter, throughput by port by month - when the question is how each place changed rather than where it ranks |
| Animated World Choropleth | 1 categorical, 3+ time points, geo: country-iso3/country-iso2/country-name | <=250 categories, needs 400x300 px | D3 | Best for a country-level measure observed at 3 or more ordered time points — watch how sales, users, or population shift across countries year by year |
| Annotated heatmap | 2 categorical, 1 continuous | <=20 categories, needs 320x280 px | D3, Matplotlib, Plotly, Seaborn | Requires two variables: categorical × categorical with values |
| Arc diagram | 2 categorical | <=50 nodes, needs 380x240 px | D3 | Best for 5-50 nodes and a sparse-to-medium-density set of pairwise relationships |
| Area chart | 1 continuous | needs 360x280 px | D3, Matplotlib, Plotly, Seaborn | Requires ordered categories or time with a continuous variable |
| Autocorrelation plot | 1 continuous, 20+ time points | needs 400x300 px | D3, Matplotlib, Plotly, Statsmodels | Requires a datetime series with one continuous variable |
| Bar chart (horizontal) | - | needs 360x280 px | D3, Matplotlib, Plotly, Seaborn | Requires one categorical and one numeric variable |
| Bar chart (vertical) | - | needs 600x420 px | D3, Matplotlib, Plotly, Seaborn | Requires one categorical and one numeric variable |
| Basic Sankey | 2 categorical, 1 continuous | <=80 categories, <=80 edges, needs 320x240 px | D3, Plotly | Requires two categorical (nodes) and a continuous variable |
| Beeswarm chart | 1 categorical, 1 continuous | <=30 categories, needs 280x280 px | D3, Matplotlib, Plotly, Seaborn | Best for showing distribution density across small categorical sets |
| Binned 2D heatmap | 2 continuous | needs 320x280 px | D3 | Best for dense scatter data where points overplot: thousands of observations of two continuous variables, where the question is where the mass is rather than where each point is |
| Binned bubble matrix | 1 continuous | needs 320x300 px | D3, Matplotlib, Plotly, Seaborn | Two banded/categorical axes (or two continuous variables split into ranges) plus an aggregatable size measure per cell, with optional independent color. Choose this over a continuous Bubble chart when raw points would overplot into a blob, or when the axes are naturally grouped |
| Bivariate USA Choropleth (by state) | 1 categorical, 2 continuous, geo: us-state-code/us-state-name | <=56 categories, needs 400x300 px | D3 | Best for two state-level measures that might disagree: revenue vs return rate, income vs cost of living, cases vs vaccinations - at least 24 states and two measures that do not simply track each other |
| Bivariate World Choropleth | 1 categorical, 2 continuous, geo: country-iso3/country-iso2/country-name | <=250 categories, needs 400x300 px | D3 | Best for two country-level measures that might disagree: wealth vs inequality, revenue vs margin, cases vs deaths - at least 24 countries and two measures that do not simply track each other |
| Box plot | 1 continuous | needs 460x340 px | D3, Matplotlib, Plotly, Seaborn | Requires one continuous variable (plus optional category) |
| Bubble chart | 3 continuous | needs 520x380 px | D3, Matplotlib, Plotly, Seaborn | Three continuous variables (x, y, size); one mark per row on continuous axes. If you want both axes grouped into bands with one aggregated bubble per cell, pick Binned bubble matrix instead |
| Bullet chart | 1 continuous, a reference value | needs 220x120 px | D3, PLOTLY | Best for actual-vs-target KPIs with thresholds (revenue vs goal, SLA vs target) — a denser, clearer replacement for gauges |
| Bump chart | 1 categorical, 1 continuous, 3+ time points | <=15 categories, needs 320x280 px | D3, PLOTLY | Best for leaderboard/position dynamics: how a handful of regions, products, or teams change rank quarter to quarter |
| Calendar heatmap | 1 continuous, 60+ time points | needs 600x420 px | D3, Matplotlib, Plotly, Seaborn | Requires daily datetime data with associated numeric values; best for activity/contribution tracking over time |
| Candlestick chart | 4 continuous, 10+ time points | needs 600x420 px | D3, Matplotlib, Plotly | Best for financial or trading data |
| Card | 1 continuous | needs 100x60 px | D3, Plotly | One numeric value that summarizes the dataset; small viewports |
| Card with embedded | 1 continuous | needs 200x120 px | D3 | One headline value that deserves one more channel beside it: attainment with its target, a metric with its history, a profile of a few comparable scores, or a total with its split across up to eight categories; small tiles |
| Card with multiple KPI | 2 continuous | needs 180x100 px | D3, Plotly | Multiple independent numeric metrics that should be surfaced together; KPI dashboards; long-narrow / short-wide viewports |
| Card with sparkline | 1 continuous | needs 140x80 px | D3, Plotly | One numeric metric trending over an ordered dimension; small viewports |
| Chord diagram | 2 categorical, 1 continuous | <=25 categories, <=30 nodes, needs 320x320 px | D3 | Requires matrix data showing bidirectional flows between entities; best for migration, trade, or relationship intensity data |
| Circular heatmap | 1 categorical, 1 continuous | <=60 categories, needs 520x380 px | D3, Matplotlib, Plotly | Requires two dimensions - a cyclical one such as hour, weekday or month around the angle and a second category or date for the rings - and one measure for the colour |
| Clustered heatmap | 2 categorical, 1 continuous | <=50 categories, needs 320x320 px | Plotly | Requires two variables for matrix values |
| Connected scatterplot | 2 continuous, 6+ time points | needs 300x300 px | D3, PLOTLY | Best for the joint trajectory of two related metrics over time (e.g. units vs revenue, unemployment vs inflation) where the path shape is the insight |
| Custom small multiples | 1 categorical, 1 continuous | <=16 categories, needs 320x240 px | D3, Matplotlib, Plotly, Seaborn | Best when comparing patterns across categories |
| Cycle plot | 1 continuous, 24+ time points | needs 320x280 px | D3, PLOTLY | Best for multi-year monthly/quarterly/weekly data where the question is seasonality vs trend ('is Q4 always highest, and is each Q4 growing?') |
| Delta KPI | 2 continuous | needs 200x120 px | D3 | Two comparable measures, or one measure at two points in time - this period against last, actual against plan, after against before |
| Dendrogram (tree diagram) | 2 categorical, 1 continuous | <=60 leaves, needs 320x240 px | D3, Plotly | Best for 2-4 hierarchy levels with ≤ ~80 leaves |
| Density contour plot | 2 continuous | needs 600x420 px | D3, Matplotlib, Plotly, Seaborn | Requires two continuous variables |
| Donut chart | 1 categorical | <=10 categories, needs 520x380 px | D3, Matplotlib, Plotly | Requires one categorical variable with proportions |
| Dumbbell chart | 1 categorical, 2 continuous | needs 600x420 px | D3, Plotly | Best for before/after or comparison scenarios |
| ECDF plot | 1 continuous | needs 299x283 px | D3, Matplotlib, Plotly, Seaborn, Statsmodels | Requires one continuous variable |
| Error band | 1 continuous | needs 360x240 px | D3 | Best for repeated measurements over time or over a continuous axis: latency trials, sensor readings, A/B metrics - anywhere the SPREAD matters as much as the average |
| Error bar plot | 1 categorical, 1 continuous | needs 460x340 px | D3, Matplotlib, Plotly, Seaborn | Requires a category and a measure with a spread to show - repeated rows per category or a second measure holding the error |
| Event plot | 1 continuous | needs 400x300 px | D3, Matplotlib, Plotly, Seaborn | Requires a date or ordered value for each event, with an optional category that splits the events into lanes |
| Flippable multi-card | 1 continuous | <=12 categories, needs 180x120 px | D3 | Best when more headline numbers compete for one tile than fit at once: several measures side by side, or one or two measures paged through a small category, with the details on hover |
| Funnel chart | 1 continuous | <=12 categories, needs 460x340 px | D3, PLOTLY | Best for conversion/attrition pipelines: signup funnels, sales pipeline stages, checkout flow drop-off |
| Gantt chart | 1 categorical, 2 temporal | <=60 categories, needs 420x260 px | D3 | Best for project and phase schedules: tasks with a start and an end date, optionally grouped by phase, workstream or owner |
| Gauge | 1 continuous, a bounded scale | needs 140x140 px | D3 | A value against a ceiling the data states - a percentage, a rate, or a measure that has a target - overall or one dial per category; a constant value is a valid reading |
| Grouped bar chart | 2 categorical, 1 continuous | needs 700x480 px | D3, Matplotlib, Plotly, Seaborn | Requires one categorical, one subcategory, and one numeric variable |
| Heatmap | 2 categorical, 1 continuous | <=50 categories, needs 460x340 px | D3, Matplotlib, Plotly, Seaborn | Requires two categorical or ordinal fields for the rows and columns, and one measure for the cell colour |
| Hexbin plot | 2 continuous | needs 400x300 px | D3, Matplotlib, Plotly, Seaborn | Requires two continuous variables |
| Histogram | 1 continuous | needs 400x300 px | D3, Matplotlib, Plotly, Seaborn | Requires one continuous variable |
| Horizon chart | 1 categorical, 1 continuous, 12+ time points | needs 800x560 px | D3, Matplotlib, Plotly | Best for dense time series dashboards |
| Horizontal/vertical span plot | 1 categorical, 2 continuous | needs 400x300 px | D3, Matplotlib, Plotly, Seaborn | Requires a category and two numeric bounds per category - a low and a high, or a start and an end - on one shared scale |
| Icicle plot | 2 categorical, 1 continuous | <=60 leaves, needs 320x240 px | D3, Plotly | Best for 2-4 levels of hierarchy with ≤ ~50 leaves; an alternative to a sunburst when horizontal/vertical scanning is preferred |
| Joint plot | 2 continuous | needs 360x280 px | D3, Matplotlib, Plotly, Seaborn | Requires two continuous variables |
| KDE plot (1D & 2D) | 1 continuous | needs 400x300 px | D3, Matplotlib, Plotly, Seaborn, Statsmodels | Requires one continuous variable (or two for 2D KDE) |
| Lag plot (scatter of value vs lag) | 1 continuous, 20+ time points | needs 280x280 px | D3, Matplotlib, Plotly, Seaborn, Statsmodels | Requires a datetime series with one continuous variable |
| Line chart | 1 continuous | needs 400x300 px | D3, Matplotlib, Plotly, Seaborn | One or more continuous series over an ordered/temporal axis |
| Linear gauge | 1 continuous, a bounded scale | needs 180x50 px | D3 | A value against a ceiling the data states, overall or one track per category, in a short wide tile - percent complete, utilisation, progress to a target; a constant value is a valid reading |
| Lollipop chart | 1 categorical, 1 continuous | <=50 categories, needs 700x480 px | D3, PLOTLY | Best as a cleaner bar replacement for ranking many categories by a single measure |
| Marimekko / Mosaic plot | 2 categorical, 1 continuous | needs 320x280 px | D3 | Best for 3-10 outer × 3-10 inner categories with a positive measure |
| Mermaid diagram | 1 categorical | <=150 categories, needs 360x240 px | D3 | A source and a target column (a process, a hand-off chain, a dependency list), two to four nested categories (an org or product hierarchy), or a date beside an event name (milestones, releases, history) - up to a few dozen boxes |
| Motion bubble chart | 1 categorical, 2 continuous, 3+ time points | <=40 categories, needs 360x320 px | D3 | Requires one categorical whose members are the bubbles, two measures for position and a time column with 3 or more ordered periods; best for watching entities travel - plans by subscribers and churn, countries by wealth and lifespan, teams by cost and output |
| Multi-level Sankey | 3 categorical, 1 continuous | <=80 edges, needs 420x280 px | D3, Plotly | Requires three (or more) categorical and one continuous variable |
| Network diagram | 2 categorical | <=150 categories, <=80 edges, needs 320x280 px | D3, Matplotlib | Requires node and edge data (source-target pairs); best for relationship, dependency, or social network visualization |
| Normalized stacked bar chart | 2 categorical, 1 continuous | needs 600x420 px | D3 | Best with two categoricals and one additive measure when the question is what the composition is rather than how big the total is |
| North America (Bubbles) | lat/lon | needs 400x300 px | D3 | Best for point-located data with latitude and longitude columns: stores, cities, facilities, or events across North America, with a measure for bubble size and optionally a second measure or category for color |
| Origin-Destination Flow Map | lat/lon | needs 400x300 px | D3 | Best for movement between places: flight routes, shipping and trade lanes, migration, commutes or transfers, where each row names where something started and where it ended plus a measure of how much moved |
| Packed circles | 2 categorical, 1 continuous | needs 280x280 px | D3 | Best with one numeric measure plus 1-2 categorical groupings producing 20-150 leaves |
| Pair plot (scatter matrix) | 3 continuous | needs 480x400 px | D3, Matplotlib, Plotly, Seaborn | Requires at least three continuous variables |
| Parallel coordinates | 3 continuous | needs 700x480 px | D3, Plotly | Best for 4-8 numeric dimensions and up to ~1000 rows (heavier overlap with more) |
| Pareto chart | 1 categorical | <=30 categories, needs 320x280 px | D3, MATPLOTLIB, PLOTLY | Best for finding the vital few: defect/complaint/incident counts by cause, revenue concentration by product or customer (the 80/20 view) |
| Pie chart | 1 categorical | <=10 categories, needs 280x280 px | D3, Matplotlib, Plotly | Requires one categorical variable with proportions |
| Polar plot | 1 continuous | needs 299x283 px | D3, Matplotlib, Plotly | Requires angle and radius (two continuous variables) |
| Progress ring | 1 continuous, a bounded scale | needs 160x160 px | D3 | A value against a ceiling the data states, overall or one ring per category, in a small or square tile - percent complete, attainment, utilisation; a constant value is a valid reading |
| Radar (spider) chart | 1 continuous | <=8 axes, needs 700x480 px | D3, Matplotlib, Plotly | Requires multiple continuous variables across categories |
| Range-compare time series | 1 continuous, 15+ time points | needs 480x320 px | D3 | Best for a single measure tracked over many dates when the story is a chosen window: drag the two handles to total a period and compare its endpoints, like revenue across a campaign or sensor readings around an incident |
| Regression plot (line with CI) | 2 continuous | needs 520x380 px | D3, Matplotlib, Plotly, Seaborn, Statsmodels | Requires two continuous variables - a predictor and a response |
| Ridgeline plots | 1 categorical, 1 continuous | <=25 categories, needs 280x280 px | D3, Matplotlib, Plotly, Seaborn | Best with many observations per category |
| Rose (Coxcomb) chart | 1 categorical | <=24 categories, needs 300x300 px | D3 | Best for cyclical or directional categories: wind direction, hour of day, month of year, compass sectors - anywhere the categories wrap around rather than running left to right |
| Rotating carousel | 1 continuous | <=12 categories, needs 320x200 px | D3 | Best when each category value deserves its own small chart and the tile has room for only one at a time; the reader turns it, or it turns itself |
| Rug plot | 1 continuous | needs 360x280 px | D3, Matplotlib, Plotly, Seaborn | Requires one continuous variable |
| Sankey alluvial flow | 2 categorical, 1 continuous | <=10 categories, <=80 edges, needs 420x280 px | D3, Matplotlib, Plotly | Requires at least two categorical variables |
| Scatter plot | 2 continuous | needs 460x340 px | D3, Matplotlib, Plotly, Seaborn | Requires two continuous variables |
| Seasonal decomposition | 1 continuous, 24+ time points | needs 320x360 px | Matplotlib, Plotly, Statsmodels | Requires a datetime series with one continuous variable and at least two full seasonal cycles |
| Slope chart | 1 categorical, 1 continuous, 2+ time points | <=40 categories, needs 460x340 px | D3, PLOTLY | Best for two-period change across many categories: YoY by region/segment, before/after comparisons, ranking shifts between two snapshots |
| Spiral plot | 1 continuous, 12+ time points | needs 320x320 px | D3, PLOTLY | Best for long, finely-grained, strongly periodic series (years of daily data) where the cyclic pattern is the story |
| Stacked area chart | 1 categorical, 1 continuous | needs 460x340 px | D3, Matplotlib, Plotly | Requires ordered categories with multiple continuous series |
| Stacked bar chart | 2 categorical, 1 continuous | needs 700x480 px | D3, Matplotlib, Plotly, Seaborn | Requires one categorical, one subcategory, and one numeric variable |
| Stem plot | 1 continuous | needs 360x280 px | D3, Matplotlib, Plotly | Requires one measure read along an ordered sequence - a time step, a row index or an ordered category |
| Step plot | 1 continuous | needs 460x340 px | D3, Matplotlib, Plotly, Seaborn | Requires ordered categories with one continuous variable |
| Streamgraph | 1 categorical, 1 continuous, 8+ time points | needs 520x380 px | D3, Plotly | Best with 3-12 categorical series and a continuous time axis with at least 15-30 evenly-spaced points |
| Strip plot | 1 continuous | needs 700x480 px | D3, Matplotlib, Plotly, Seaborn | Requires one categorical and one continuous variable; jitter on the category axis prevents overplot at the same value |
| Sunburst chart | 2 categorical, 1 continuous | needs 400x300 px | D3, Matplotlib, Plotly | Requires hierarchical categorical data |
| Tabular with embedded | 1 categorical, 1 continuous | <=40 categories, needs 420x200 px | D3 | Best for a small set of items compared across several attributes at once (a forecast/roster/scorecard table) |
| Ternary plot | 3 continuous | needs 320x320 px | D3 | Three measures that add up to the same total on every row - product mix, budget allocation, vote share, channel split |
| Thermometer | 1 continuous, a bounded scale | needs 90x220 px | D3 | A value against a ceiling the data states, overall or one tube per category, in a tall narrow tile - a sidebar rail or a column of KPIs down a page edge; a constant value is a valid reading |
| Time series plot | 1 continuous | needs 460x340 px | D3, Plotly | Requires a datetime axis with at least one continuous variable |
| Trail | 2 continuous | needs 360x220 px | D3 | Best when a series has both a level and a weight: price with traded volume, sentiment with mention count, a metric with the sample size behind it |
| Treemap | 1 categorical, 1 continuous | needs 240x220 px | D3, Matplotlib, Plotly | Requires hierarchical categorical data with values |
| US hex-tile cartogram | 1 categorical, geo: us-state-code/us-state-name | needs 400x300 px | D3 | Best with one row per US state and a single measure, when the question is which states are high or low rather than where they are |
| USA Choropleth (by state) | 1 categorical, geo: us-state-code/us-state-name | <=56 categories, needs 400x300 px | D3 | Best for state-level US metrics: sales, users, or population by state, any measure that varies by US state (keyed to state names or USPS codes) |
| USA Choropleth (by ZIP-3) | 1 categorical, geo: us-zip5 | <=42000 categories, needs 400x300 px | D3 | Best for regional US ZIP analysis: metrics by 3-digit ZIP prefix area, any measure that varies across US postal regions (keyed to 5-digit ZIP codes, grouped to their 3-digit prefix) |
| Variance chart (budget vs actual) | 1 categorical, 2 continuous, a measure naming itself budget/plan/target | <=60 categories, needs 360x240 px | D3 | One category or period column plus two measures where one names itself the reference - Revenue and Revenue Budget, Cost and Cost Plan, Attained and Quota; finance reporting, budget reviews, plan attainment |
| Violin plot | 1 continuous | needs 280x280 px | D3, Matplotlib, Plotly, Seaborn | Requires one continuous variable (plus optional category) |
| Voronoi treemap | 2 categorical, 1 continuous | <=50 cells, needs 320x280 px | D3 | Best for 5-30 categories with one non-negative measure; an alternative to a plain treemap when the organic look is desired |
| Waffle chart | 1 categorical, 1 continuous | <=6 categories, needs 520x380 px | D3, Matplotlib, Plotly | Best for showing part-to-whole relationships with small number of categories; more accurate than pie charts |
| Waterfall chart | 1 continuous | <=20 categories, needs 360x280 px | D3, Matplotlib, Plotly | Best for financial data showing contribution to total (revenue bridges, P&L analysis, budget variance) |
| What-if predictor | 3 continuous | needs 520x300 px | D3 | Three to ten numeric measures over at least thirty rows that move together - stores, products, campaigns, regions by quarter - when the question is 'if these were X and Y, what would the others likely be?'; an ordered rating or size and one category of up to six values can be pinned too |
| What-if projection | 1 continuous, 6+ time points | needs 420x280 px | D3 | A dated measure with at least six periods when the question is what happens next under an assumption the reader chooses - revenue against a target, headcount, subscribers; bind a What-If parameter measure to drive the rate from a slicer |
| What-if scenarios | 1 continuous, 6+ time points | needs 480x320 px | D3 | A dated measure with at least six periods when the question is how far apart the outcomes of several assumptions land - revenue under a range of growth rates, headcount, subscribers; bind a What-If parameter column to draw exactly the range you defined |
| Word cloud | 1 categorical | <=100 categories, needs 280x200 px | D3, Matplotlib, Plotly | Requires text data or pre-computed word-frequency pairs; best for qualitative text exploration |
| World (Bubbles) | lat/lon | needs 400x300 px | D3 | Best for point or country-level data spanning more than one continent: offices, shipments, users or revenue by country, with a measure for bubble size and optionally a second measure or category for color |
| World Choropleth | 1 categorical, geo: country-iso3/country-iso2/country-name | <=250 categories, needs 400x300 px | D3, PLOTLY | Best for country-level metrics: sales, users, or population by country, any measure that varies by nation (keyed to country names or ISO codes) |
| World tile-grid cartogram | 1 categorical, geo: country-iso3/country-iso2/country-name | <=250 categories, needs 510x420 px | D3 | Best with one row per country and a single measure, when the question is which countries are high or low rather than where they are |

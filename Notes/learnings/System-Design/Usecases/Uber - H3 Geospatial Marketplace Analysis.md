# Uber - H3 Geospatial Marketplace Analysis

## The Core Problem

A city-wide ride marketplace needs to understand where demand exceeds available supply. Individual GPS coordinates offer detail but are expensive to analyze collectively; postal districts and manually drawn zones have inconsistent shapes and boundaries that can change independently of marketplace needs.

Uber's 2018 engineering account describes H3 as its shared grid for marketplace analysis and optimization, including supply-and-demand measurements supporting surge pricing. The key idea is to convert locations into comparable geographic buckets. H3 is the spatial representation, not a complete dispatch algorithm or pricing engine. This note separates that published use case from current H3 library capabilities.

## Architecture & Component Design

First, map locations to cells using [[Hierarchical Spatial Indexing]]. H3 offers resolutions 0 through 15, from coarse to fine. Its global grid is mostly hexagonal, with twelve pentagons at each resolution because a sphere cannot be tiled entirely with hexagons. The resolution determines the scale of analysis; the sources do not establish one universal Uber production setting.

In current H3 4.x JavaScript bindings, `latLngToCell(lat, lng, resolution)` returns the containing cell's ID. Records can then share a spatial key even when their coordinates differ. The API also exposes cell boundaries for drawing the resulting map, so analytics and visualization can refer to the same cells.

Hexagonal neighborhoods simplify local comparisons: in an ideal planar hexagonal grid, the six adjacent cell centers are equally distant, avoiding a square grid's distinction between edge and diagonal neighbors. This is a geometric motivation, not a claim that every projected H3 cell on Earth is perfectly regular.

An illustrative aggregation pipeline is:

```text
Location records → coordinate-to-cell conversion
                 → counts grouped by cell and time window
                 → neighborhood comparisons and map display
```

This is a design sketch, not Uber's disclosed service topology. Such grouped calculations can use [[Parallel Processing]]: workers calculate partial counts for input batches and merge counts with matching keys. Window definitions and the meaning of supply remain application decisions; counting every GPS update would not by itself count distinct available drivers.

The hierarchy adds another operation. `cellToParent(cell, coarserResolution)` finds a unique logical ancestor, enabling broader summaries. Separately, `gridDisk(cell, k)` enumerates cells within `k` adjacency hops for neighborhood analysis. Neither operation requires a hand-maintained neighborhood polygon.

## Trade-offs & Bottlenecks

- **Resolution versus evidence:** Small cells reveal local changes but may contain too few observations. Coarser summaries improve sample size while masking differences. Selecting resolution is part of the analysis, not merely a storage setting.
- **Hierarchy versus geography:** Fine H3 cells do not fit perfectly inside their geometric parents. Rolling up fine-cell counts can therefore differ from assigning original points directly to coarse cells. Choose one interpretation consistently; exact boundary requirements need additional geometric checks.
- **Grid proximity versus travel:** A disk measures adjacency hops, not kilometers or driving time. Inferring that a nearby cell implies a quick pickup would ignore roads, rivers, and access restrictions. This is a design implication, not a documented Uber dispatch rule.
- **Unequal areas and traffic:** H3 cell areas vary. Density comparisons should account for area; raw counts answer a different question. Likewise, a busy airport can concentrate aggregation work despite a regular grid. The hot-key risk follows from the illustrative pipeline, not a reported Uber incident.

## Key Takeaway

H3 makes geography usable as an aggregation key and supplies tools for examining neighboring areas and multiple scales. Its value comes with explicit semantics: a logical parent is not perfect geographic containment, and grid distance is not travel time. Keep those distinctions intact when using spatial summaries to inform marketplace decisions.

Sources: [Uber engineering account, 2018](https://www.uber.com/ae/en/blog/h3/) · [H3 grid construction](https://h3geo.org/docs/core-library/overview/) · [Coordinate indexing](https://h3geo.org/docs/api/indexing/) · [Hexagonal aggregation](https://h3geo.org/docs/highlights/aggregation/) · [Hierarchy API](https://h3geo.org/docs/api/hierarchy/) · [Containment semantics](https://h3geo.org/docs/highlights/indexing/) · [Grid traversal](https://h3geo.org/docs/api/traversal/) · [Cell areas](https://h3geo.org/docs/core-library/restable/). H3 4.x documentation checked September 2026; modern API names do not imply their use in the 2018 deployment.

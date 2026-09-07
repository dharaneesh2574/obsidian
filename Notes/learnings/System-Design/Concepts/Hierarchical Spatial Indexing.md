# Hierarchical Spatial Indexing

## What It Is

Hierarchical spatial indexing assigns locations to identifiable cells at multiple resolutions. A cell has a coarser ancestor and finer descendants, allowing applications to organize geographic data at different levels of detail. H3 is one implementation, using a mostly hexagonal global grid.

## Why It Exists

Raw coordinates are awkward keys for questions about areas: nearby points rarely have identical coordinates. Cells provide shared aggregation keys, while multiple resolutions let an application choose between local detail and broader summaries.

## How It Works

1. Choose a resolution and map each coordinate to its containing cell. In H3 4.x, `latLngToCell` performs this conversion.
2. Group records by cell ID and compute counts, sums, or other statistics. Independent input batches can use [[Parallel Processing]], with partial results merged afterward.
3. Use `cellToParent` to obtain a coarser logical key, or `cellToChildren` to enumerate descendants. Roll up additive statistics; combine counts and sums before recalculating averages rather than averaging averages.
4. Inspect nearby cells with grid-traversal operations. H3's `gridDisk` returns cells within a specified number of adjacency hops, not a physical-distance guarantee.

H3's logical hierarchy is exact, but finer hexagons can extend outside their parent's geographic boundary. Consequently, a point's fine-cell ancestor need not equal the cell obtained by indexing that point directly at the coarser resolution. Individual resolutions still have well-defined boundaries.

## Trade-offs

- Finer cells preserve detail but create more groups and sparser samples; coarser cells can hide local differences.
- A hierarchy is not automatically an exact polygon-containment index. For strict boundary queries, retain coordinates and use appropriate geometric checks.
- Spatial grouping does not balance computational load: popular locations can remain hot keys. This is a general processing concern, not a guarantee provided by the index.
- H3 has twelve pentagons per resolution and nonuniform cell areas. Do not assume every cell has six neighbors or identical area.

## Related

[[Uber - H3 Geospatial Marketplace Analysis]] · [[Parallel Processing]]

References: [H3 indexing and containment](https://h3geo.org/docs/highlights/indexing/) · [Coordinate-to-cell API](https://h3geo.org/docs/api/indexing/) · [Hierarchy API](https://h3geo.org/docs/api/hierarchy/) · [Grid traversal](https://h3geo.org/docs/api/traversal/) · [Grid construction](https://h3geo.org/docs/core-library/overview/) · [Cell statistics](https://h3geo.org/docs/core-library/restable/).

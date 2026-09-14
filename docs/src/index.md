# Networks.jl

Store a network together with the information needed to analyse it: actor attributes, edge values, network metadata, and explicit missing dyads. Networks.jl supplies the shared data types used throughout this ecosystem and implements the Graphs.jl graph interface.

| First analysis | Learn the model or data | Reference and detail |
|:--|:--|:--|
| [Build your first network](getting_started.md) | [Import, convert, and export](guide/conversion.md) | [Network types and operations](api/types.md) |

!!! note "Supported scope"

    Static directed, undirected, and two-mode networks are supported. Edge values are attributes, with interpretation left to the analysis package. Multiple edges and hyperedges are not implemented. A missing dyad is unobserved, rather than an observed absence; consult the [missing-data guide](guide/missing_data.md) before analysis.

## Installation

```@raw html
<p>Use Julia <strong>1.12 or newer</strong> and the <a href="/getting-started/">shared workspace installation guide</a>. These development packages are not yet registered; the guide prepares the required sibling checkouts and a Julia environment for the examples.</p>
```

## Quick Start

Load the bundled Florentine marriage network and inspect its actors, ties, and wealth attribute:

```julia
using Networks

net = load_dataset(:florentine_marriage)
wealth = vertex_attribute_vector(net, :wealth, Int)
A = as_matrix(net)
println((actors=nv(net), ties=ne(net), density=network_density(net)))
println((adjacency_size=size(A), recorded_wealth=length(wealth)))
```

```@raw html
<p>The network contains 16 families. Its adjacency matrix describes marriage ties; the wealth vector remains actor data. Continue with <a href="/SNA.jl/dev/">SNA.jl</a> for descriptive analysis or <a href="/ERGM.jl/dev/">ERGM.jl</a> for a statistical model.</p>
```

## Common Operations

| Task | Function |
|------|----------|
| Create a network | [`network(n)`](@ref network) |
| Add/remove edges | [`add_edge!`](@ref), [`rem_edge!`](@ref) |
| Vertex attributes | [`set_vertex_attribute!`](@ref), [`get_vertex_attribute`](@ref) |
| Edge attributes | [`set_edge_attribute!`](@ref), [`get_edge_attribute`](@ref) |
| Network attributes | [`set_network_attribute!`](@ref), [`get_network_attribute`](@ref) |
| To adjacency matrix | [`as_matrix`](@ref) |
| To edge list | [`as_edgelist`](@ref) |
| To DataFrame | [`as_dataframe`](@ref) |
| From matrix | [`network_from_matrix`](@ref) |
| Read Pajek file | [`read_pajek`](@ref) |
| Write Pajek file | [`write_pajek`](@ref) |

## Documentation

```@contents
Pages = [
    "getting_started.md",
    "guide/construction.md",
    "guide/attributes.md",
    "guide/missing_data.md",
    "guide/conversion.md",
    "guide/conversion_invariants.md",
    "guide/io.md",
    "guide/datasets.md",
    "api/types.md",
    "api/graph_interface.md",
    "api/attributes.md",
    "api/conversion.md",
    "api/contracts.md",
    "api/metadata.md",
    "api/inference.md",
]
Depth = 2
```

## Comparison with R's network Package

Networks.jl mirrors the R `network` package API as closely as Julia idioms allow:

| R (network package) | Julia (Networks.jl) |
|----------------------|--------------------|
| `network(n, directed=TRUE)` | `Network(n; directed=true)` or `network(n; directed=true)` |
| `network.size(net)` | `nv(net)` or `network_size(net)` |
| `network.edgecount(net)` | `ne(net)` or `network_edgecount(net)` |
| `network.density(net)` | `network_density(net)` |
| `add.edges(net, tail, head)` | `add_edges!(net, pairs)` |
| `is.adjacent(net, i, j)` | `has_edge(net, i, j)` |
| `get.neighborhood(net, v)` | `neighbors(net, v)` (`get_neighborhood(net, v, order)` includes `v`) |
| `net[i, j] <- NA` | `set_missing_dyad!(net, i, j)` — unobserved, not absent |
| `is.na(net)` | `missing_dyads(net)` |
| `network.naedgecount(net)` | `n_missing_dyads(net)` |
| `net %v% "attr"` | `get_vertex_attribute(net, :attr)` |
| `net %e% "attr"` | `get_edge_attribute(net, :attr)` |
| `net %n% "attr"` | `get_network_attribute(net, :attr)` |
| `set.vertex.attribute(net, "a", v)` | `set_vertex_attribute!(net, :a, v)` |
| `as.matrix(net)` | `as_matrix(net)` |
| `as.edgelist(net)` | `as_edgelist(net)` |
| `read.paj("file.net")` | `read_pajek("file.net")` |
| `print(net)` | `net` at the REPL / `display(net)`; `println(net)` is the one-line form |

## Module

```@docs
Networks
```

## References

1. Butts, C.T. (2008). network: A Package for Managing Relational Data in R. *Journal of Statistical Software*, 24(2), 1-36.

2. Butts, C.T. (2015). network: Classes for Relational Data. The Statnet Project. R package version 1.13.0.

3. Handcock, M.S., Hunter, D.R., Butts, C.T., Goodreau, S.M., Morris, M. (2008). statnet: Software Tools for the Representation, Visualization, Analysis and Simulation of Network Data. *Journal of Statistical Software*, 24(1), 1-11.

4. Csardi, G., Nepusz, T. (2006). The igraph software package for complex network research. *InterJournal*, Complex Systems, 1695.

## Citation

If you use Networks.jl in your work, please cite it using the entry in
[`CITATION.bib`](https://github.com/statistical-network-analysis-with-Julia/Networks.jl/blob/main/CITATION.bib):

```biblatex
@misc{SNWJNetworksJL,
  author = {{Statistical Network Analysis with Julia}},
  title = {Networks.jl: Core Network Data Structures for Julia},
  year = {2026},
  url = {https://github.com/statistical-network-analysis-with-Julia/Networks.jl},
  note = {Homepage: https://statistical-network-analysis-with-Julia.github.io/Networks.jl; GitHub: https://github.com/statistical-network-analysis-with-Julia}
}
```

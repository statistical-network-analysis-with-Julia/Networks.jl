# Networks.jl


[![Network Analysis](https://img.shields.io/badge/Network-Analysis-orange.svg)](https://github.com/statistical-network-analysis-with-Julia/Networks.jl)
[![Build Status](https://github.com/statistical-network-analysis-with-Julia/Networks.jl/actions/workflows/CI.yml/badge.svg?branch=main)](https://github.com/statistical-network-analysis-with-Julia/Networks.jl/actions/workflows/CI.yml?query=branch%3Amain)
[![Documentation](https://img.shields.io/badge/docs-stable-blue.svg)](https://statistical-network-analysis-with-Julia.github.io/Networks.jl/stable/)
[![Documentation](https://img.shields.io/badge/docs-dev-blue.svg)](https://statistical-network-analysis-with-Julia.github.io/Networks.jl/dev/)
[![Julia](https://img.shields.io/badge/Julia-1.12+-purple.svg)](https://julialang.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

<p align="center">
  <img src="docs/src/assets/logo.svg" alt="Networks.jl icon" width="160">
</p>

Core network data structures for the StatNet Julia ecosystem.

## Overview

Networks.jl provides the foundational `Network{T}` type that serves as the base data structure for all other StatNet Julia packages. It implements the Graphs.jl `AbstractGraph` interface while adding support for vertex, edge, and network-level attributes.

This package is a Julia port of the R `network` package from the StatNet collection.

## Installation

Requires Julia 1.12+.

```julia
using Pkg
Pkg.add(url="https://github.com/statistical-network-analysis-with-Julia/Networks.jl")
```

For development, clone the ecosystem repositories side by side (the
sibling-checkout layout): each package's `[sources]` path dependencies
(`path = "../Networks.jl"`) resolve the siblings, so `julia --project` inside
any package checkout instantiates against the neighbouring clones with no
ordered installs needed.

## Features

- **Network type**: `Network{T}` implementing `AbstractGraph` interface
- **Attributes**: Vertex, edge, and network-level attribute storage
- **Missing data**: a missing-dyad mask marking ties whose status is
  unobserved (statnet-style NA ties), and the ecosystem contract around it:
  every SNA measure, every adapter into another representation
  (`DynamicNetwork`, contact/event sequences, Siena matrices) and
  `network_density` refuse a masked network unless asked in writing
  (`missing=:face`); the plain `as_matrix`/`as_edgelist` carry face values
  and are lossless only via `missing_value=`/`return_mask=` (see Missing Data
  below and the invariant table); ERGM.jl's MPLE drops masked dyads from the
  pseudo-likelihood. `supports_missing(f)` is the
  trait a routine opts into, `require_observed` the guard it calls, and
  `missing_policies(f)` the vocabulary its `missing=` keyword actually
  accepts (what the capability matrix prints)
- **Flexibility**: Support for directed/undirected networks, self-loops, and
  bipartite (two-mode) networks with incidence-matrix conversion
  (parallel/hyper edges are not supported and are rejected explicitly)
- **Coercion**: Convert to/from matrices and edge lists (DataFrames via a
  package extension when DataFrames.jl is loaded)
- **I/O**: Read/write Pajek and GraphML formats
- **Datasets**: six teaching datasets via `load_dataset`, each with a
  provenance record (source URL, hash, version, licence, citation) in
  `data/README.md` and extracted by a checked-in R script: the cross-sections
  `:florentine_marriage`, `:florentine_business`, `:sampson` and statnet's
  `:faux_mesa_high` (205 students, `Grade`/`Race`/`Sex`), the RSiena `:s50`
  panel (three friendship waves + alcohol + smoking) for Siena.jl, and the
  `:wtc_police_calls` relational-event stream (481 ordinal radio calls among
  37 responders) for REM.jl/Relevent.jl
- **Shared result presentation**: the R-style coefficient table
  (`print_coeftable`, with or without a `Std.Error` column and with a
  `p_floor=` for permutation tests), its inspectable form `CoefficientTable`
  — what every package's `coeftable(fit)` returns, the StatsAPI verb being
  re-exported here as the one binding — the one z → p step (`z_pvalues`:
  `erfc`-based, floored at `floatmin`, NaN-aware), p-value formatting that
  never prints `0.0` (`format_pvalue`, `signif_code`), the `(1+k)/(N+1)`
  Monte-Carlo p-value (`mc_pvalue`), the `GOFResult`/`GOFStatistic`
  goodness-of-fit containers, and the single `gof` generic that every model
  package extends — so all fitted models in the ecosystem print the same way
- **StatsAPI surface checker**: `check_statsapi(fit; strict=true)` reports
  which of `coef`/`stderror`/`vcov`/`confint`/`loglikelihood`/`nobs`/`dof`/
  `aic`/`bic`/`coeftable` a fitted model implements (and that they agree),
  the one line every package's testset uses to pin its surface
- **Shared numerics**: the one Newton–Raphson optimizer
  (`Networks.newton_fit`, with a finiteness guard on every step and `NaN`
  standard errors — never finite nonsense — at an indefinite Hessian), the
  one logistic-likelihood kernel (`Networks.logistic_derivatives`, binary or
  weighted or binomial rows, allocation-free per evaluation), the one
  bootstrap loop (`bootstrap_cov`) and the one `se=` validator (`check_se`).
  The optimizer pair is `public`, not exported — ERGM.jl re-exports it. This
  is their home: REM.jl and Relevent.jl do not depend on ERGM.jl, and
  Networks.jl is the one package all five estimators share
- **Golden-fixture harness**: `load_golden` refuses a fixture without a
  `[provenance]` block or whose generating R script cannot be found (resolved
  by walking up from the fixture, `root=` to override; the path lands in
  `script_path`), `check_golden`/`golden_report` compare against the
  tolerance the fixture declares — how every package pins its numbers to R

## Quick Start

```julia
using Networks

# Create a directed network with 5 vertices
net = network(5; directed=true)

# Add edges
add_edge!(net, 1, 2)
add_edge!(net, 2, 3)
add_edge!(net, 3, 1)

# Set vertex attributes
set_vertex_attribute!(net, :name, 1, "Alice")
set_vertex_attribute!(net, :name, 2, "Bob")

# Set edge attributes
set_edge_attribute!(net, :weight, 1, 2, 0.5)

# Query network
println("Vertices: ", nv(net))
println("Edges: ", ne(net))
println("Directed: ", is_directed(net))
```

## Network Creation

```julia
# Empty network
net = network(10)

# From adjacency matrix
mat = [0 1 0; 1 0 1; 0 1 0]
net = network_from_matrix(mat)

# From edge list (do not name the variable `edges` — that would shadow the
# `edges(net)` iterator function)
edge_list = [(1, 2), (2, 3), (3, 1)]
net = network_from_edgelist(edge_list; n=3)
```

## Attributes

<!-- skip-check -->
```julia
# Vertex attributes
set_vertex_attribute!(net, :attr_name, v, value)
val = get_vertex_attribute(net, :attr_name)     # Returns Dict
val = get_vertex_attribute(net, :attr_name, v)  # Single vertex

# Edge attributes
set_edge_attribute!(net, :attr_name, i, j, value)
val = get_edge_attribute(net, :attr_name)       # Returns Dict
val = get_edge_attribute(net, :attr_name, i, j) # Single edge

# Network attributes
set_network_attribute!(net, :attr_name, value)
val = get_network_attribute(net, :attr_name)

# Typed fast path for model code (single conversion pass, concrete types)
vals = get_vertex_attribute(net, :attr_name, Float64)     # Dict{Int,Float64}
vals = vertex_attribute_vector(net, :attr_name, Float64)  # Vector{Float64}, vertex order
vals = get_edge_attribute(net, :attr_name, Float64)       # Dict{Tuple{Int,Int},Float64}
```

## Missing Data

A dyad can be marked as **unobserved** (its tie status is unknown — distinct
from "no tie"). Structural queries keep reporting the stored face value; the
mask is metadata that statistical packages use to exclude the dyad from
estimation (ERGM.jl drops masked dyads from the MPLE design matrix and never
toggles them during MCMC).

```julia
net = network(5; directed=true)
set_missing_dyad!(net, 3, 4)      # tie status of 3->4 is unobserved
is_missing_dyad(net, 3, 4)        # true
n_missing_dyads(net)              # 1
collect(missing_dyads(net))       # [(3, 4)]
delete_missing_dyad!(net, 3, 4)   # observed again
clear_missing_dyads!(net)         # unmask everything
```

The mask is symmetric for undirected networks, survives `copy`, and is
remapped by `rem_vertex!`, `get_induced_subgraph`, and `permute_vertices`.
Matrix conversion is lossless on request — `as_matrix(net;
missing_value=missing)` writes `missing` at masked dyads and `as_matrix(net;
return_mask=true)` returns the mask beside the matrix, both read back by
`network_from_matrix` — while the plain `as_matrix(net)` carries face values
only. `network_density(net)` refuses a masked network; `network_density(net;
missing=:face)` counts at face value, in writing, and
`missing_policies(network_density) == (:error, :face)` says so. Limitation:
the mask is not persisted by the Pajek/GraphML/CSV I/O routines.

## Datasets

```julia
flo = load_dataset(:florentine_marriage)   # Padgett, 16 families, 20 marriage ties
flob = load_dataset(:florentine_business)  # same families, 15 business ties
samp = load_dataset(:sampson)              # Sampson monastery liking, 18 monks, directed
fmh = load_dataset(:faux_mesa_high)        # statnet faux.mesa.high, 205 students, 203 ties

get_vertex_attribute(flo, :wealth, 9)      # 103 (Medici)
vertex_attribute_vector(samp, :group, String)  # "Loyal"/"Outcasts"/"Turks"
vertex_attribute_vector(fmh, :Grade, Int)      # 7-12, statnet's attribute names

# Longitudinal and event data come back as documented NamedTuples
s50 = load_dataset(:s50)                   # RSiena s50: 50 girls, 3 waves
ne.(s50.friendship)                        # [113, 116, 122] — Vector{Network{Int,true}}
size(s50.alcohol), size(s50.smoke)         # ((50, 3), (50, 3)), coded 1-5 and 1-3

wtc = load_dataset(:wtc_police_calls)      # 481 ordinal radio calls, 37 responders
size(wtc.events)                           # (481, 3): number, sender, receiver
findall(wtc.is_icr)                        # [13, 14, 32] — the coordinator roles
```

`:s50` feeds `Siena.DependentNetwork(:friendship, s50.friendship)` directly
and `:wtc_police_calls` becomes a `REM.EventSequence` (or the event vector
`Relevent.fit_obpm` takes) with the event number as ordinal time; the
Datasets guide shows both. Each dataset's source, hash, upstream version,
licence and citation are in `data/README.md`, and `data/r/extract_datasets.R`
regenerates the files with base R and `network`.

## Conversion

```julia
using Graphs

# To adjacency matrix (incidence matrix for bipartite networks;
# expand_bipartite=true gives the (n1+n2)-square adjacency instead)
mat = as_matrix(net)

# To edge list
edge_list = as_edgelist(net)

# To Graphs.jl types (copy of the internal storage)
g = SimpleDiGraph(net)
h = SimpleGraph(net)   # collapse directions
```

## I/O

```julia
# Pajek format
write_pajek(net, "network.net")
net = read_pajek("network.net")

# GraphML format
write_graphml(net, "network.graphml")
```

## Shared Result Presentation

Networks.jl carries the presentation layer used by every statistical
package built on it (ERGM.jl, TERGM.jl, Siena.jl, REM.jl, Relevent.jl,
...), so fitted models across the ecosystem render identically:

```julia
using Networks

# R-style coefficient table (Estimate / Std.Error / z value / Pr(>|z|)
# with significance codes) — model packages call this from Base.show
print_coeftable(stdout,
                ["edges", "gwesp.fixed.0.5"],
                [-3.2145, 0.8113],     # estimates
                [0.1936, 0.2854],      # standard errors
                [1.2e-62, 0.0045])     # p-values

# The same table as an inspectable value — what every package's
# coeftable(fit) returns; show(tbl) IS print_coeftable
tbl = CoefficientTable(["edges", "gwesp.fixed.0.5"],
                       [-3.2145, 0.8113], [0.1936, 0.2854])
tbl["edges"].p_value    # 6.5e-62 (from z_pvalues, floored, never 0.0)
length(tbl)             # 2

# A permutation test: no Std.Error column, resolution 1/reps
print_coeftable(stdout, ["intercept", "x"], [0.12, 0.85], nothing, [0.31, 0.0];
                z_values=[1.02, 6.7],
                header=("Estimate", "t value", "Pr(>=|t|)"), p_floor=1/1000)

# p-values are floored, never printed as exactly 0.0
format_pvalue(0.0)      # "<1e-16"
format_pvalue(3.2e-7)   # "3.2e-07"
z_pvalues([1.96, 50.0, NaN])   # [0.04999..., floatmin(Float64), NaN]

# Two-sided Monte-Carlo p-value with the (1+k)/(N+1) estimator
mc_pvalue(randn(999), 1.5)
```

`gof` is an empty generic here: each model package adds a method for its
own result type (`gof(::ERGMResult)`, `gof(::SienaResult; ...)`, ...), so
the same verb works on every fitted model and packages never collide.
`GOFStatistic`/`GOFResult` are the common observed-vs-simulated containers
with a formatted `show`.

## Graphs.jl Compatibility

Networks.jl implements the full `AbstractGraph` interface:

<!-- skip-check -->
```julia
using Graphs

nv(net)           # Number of vertices
ne(net)           # Number of edges
vertices(net)     # Vertex iterator
edges(net)        # Edge iterator
neighbors(net, v) # Neighbors of vertex v
has_edge(net, i, j)
add_edge!(net, i, j)
rem_edge!(net, i, j)
```

## Documentation

For more detailed documentation, see:

- [Stable Documentation](https://statistical-network-analysis-with-Julia.github.io/Networks.jl/stable/)
- [Development Documentation](https://statistical-network-analysis-with-Julia.github.io/Networks.jl/dev/)

## References

1. Butts, C.T. (2008). network: A Package for Managing Relational Data in R. *Journal of Statistical Software*, 24(2), 1-36.

2. Butts, C.T. (2024). network: Classes for Relational Data. R package. [https://cran.r-project.org/package=network](https://cran.r-project.org/package=network)

## Citation

If you use Networks.jl in your work, please cite it using the entry in
[`CITATION.bib`](CITATION.bib):

```biblatex
@misc{SNWJNetworksJL,
  author = {{Statistical Network Analysis with Julia}},
  title = {Networks.jl: Core Network Data Structures for Julia},
  year = {2026},
  url = {https://github.com/statistical-network-analysis-with-Julia/Networks.jl},
  note = {Homepage: https://statistical-network-analysis-with-Julia.github.io/Networks.jl; GitHub: https://github.com/statistical-network-analysis-with-Julia}
}
```

## License

MIT License - see [LICENSE](LICENSE) for details.

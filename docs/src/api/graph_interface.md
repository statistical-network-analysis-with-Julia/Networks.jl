# Graph Interface API Reference

This page documents the Graphs.jl `AbstractGraph` interface methods implemented by Networks.jl. These functions allow `Network` objects to work seamlessly with the entire Graphs.jl algorithm library.

## Basic Properties

### nv

```@docs
nv
```

### ne

```@docs
ne
```

### vertices

```@docs
vertices
```

### edges

```@docs
edges
src
dst
```

### has\_vertex

```@docs
has_vertex
```

### has\_edge

```@docs
has_edge
```

### degree, indegree, outdegree

The Graphs.jl degree family, re-exported so that `using Networks` alone
answers the first descriptive question a user asks. On an undirected network
all three agree; on a directed one `degree` is `indegree + outdegree`. A
self-loop counts once on an undirected network and twice (one in, one out) on
a directed one, as in Graphs.jl.

```@docs
degree
indegree
outdegree
```

### is\_directed

```@docs
is_directed
```

### is\_bipartite

`is_bipartite` is Graphs.jl's graph-theoretic two-colourability test, which
Networks.jl extends for `Network`. It is *not* the statnet two-mode metadata
flag — for that, see [`is_two_mode`](@ref) below.

```@docs
Graphs.is_bipartite
```

### is\_two\_mode

```@docs
is_two_mode
```

### zero

```@docs
Base.zero
```

## Vertex Modification

### add\_vertex!

```@docs
add_vertex!
```

### add\_vertices!

```@docs
add_vertices!
```

### rem\_vertex!

```@docs
rem_vertex!
```

## Edge Modification

### add\_edge!

```@docs
add_edge!
```

### add\_edges!

```@docs
add_edges!
```

### rem\_edge!

```@docs
rem_edge!
```

## Neighbors

### neighbors

```@docs
neighbors
```

### inneighbors

```@docs
inneighbors
```

### outneighbors

```@docs
outneighbors
```

## Utility Functions

### network\_size

```@docs
network_size
```

### network\_edgecount

```@docs
network_edgecount
```

### network\_density

```@docs
network_density
```

### get\_neighborhood

```@docs
get_neighborhood
```

### get\_induced\_subgraph

```@docs
get_induced_subgraph
```

### permute\_vertices

```@docs
permute_vertices
```

# Brief overview of the supercluster JavaScript library

In my current work I got the opportunity to dive into the clustering library
[supercluster](https://github.com/mapbox/supercluster) written by
[@mourner](https://github.com/mourner). In this document I will write a brief
explanation of the library and the algorithm.

## Clusters

Before we dive into the algorithm itself, let's first briefly explain what
problem the library is trying to solve.

When working with maps, it's a common use-case to display ["points of
interest"](https://wiki.openstreetmap.org/wiki/Points_of_interest) (or POI for
short). If we try to display a huge amount of POIs the map might end up
cluttered, making it hard to use the map.

Depending on what the map is trying to display, one possible solution to this
problem can be to "bunch up" or "cluster" neighbouring POIs to form a single
POI that represent all of the clustered POIs.

The supercluster library allows you to process a set of POIs to produce a set
of clusters at a range of zoom levels starting at most zoomed in with
individual POIs, all the way up to the most zoomed out level with the whole map
shown.

## KD-tree

As is often the case for algorithms, picking a good data structure is crucial
for making it efficient.  In this case, we need to be able to quickly look up
neighbouring POIs for a single POI when building clusters.

For this, supercluster uses a ["k-dimensional
tree"](https://en.wikipedia.org/wiki/K-d_tree) (or KD-tree), a
"space-partitioning data structure". Grossly over-simplified, it's similar to
binary trees but tailored for dimensional data (e.g. points in space) with
lookup time complexities of `O(log n)`.

## Algorithm overview

The algorithm for building the clusters follows a recursive process (although
the actual implementation is iterative), starting from the bottom (high zoom
level), building the next layer with the previous layer as input. The original
set of POIs are used as the initial layer set at maximum zoom level.

For each layer, a radius is calculated as `r_l = r / pow(2, z)` where `r_l` is
the radius for current layer, `r` is a constant used for controlling the size
and growth of the radius configured for the algorithm, and `z` is the zoom
level of the layer.[^radius] As we can see in this formula, the radius is
inversely proportional to the zoom level; as we process the next layer, zoom
level decreases which means the radius increases.

Next, we will start building up clusters. We go through each POI of the input
for the layer (i.e. previous layer's POIs/clusters) and search for neighbouring
POIs that's within the calculated radius and which is not already included in a
cluster for the current layer.

If the number of neighbours is greater than a preconfigured threshold, we form
a cluster from the selected POIs by calculating the weighted center from the
POIs. The library also allows the user to "reduce" clustered POIs to preserve
any properties from the original POIs in the new cluster. For example, any
labels on the POIs can be collected into the new cluster POI. Furthermore, the
id of the newly formed cluster is saved in the clustered POIs to allow
traversal of related clusters.

Finally, any POI that was not included in a cluster or did not have enough
neighbours to form a cluster by itself, are left as they are in the new layer,
allowing them to potentially be included in clusters in further processing as
the radius increases.

## Conclusion

The final result after running this algorithm is a hierarchy of layers with
sets of clusters for various zoom levels. Thanks to the efficiency of the
KD-tree datastructure, it is able to both build the clusters quickly as well as
provide fast look up when using it.

One limitation of the library worth noting is that the final result is static,
meaning if you want to update the data the process needs to be rerun to rebuild
the data.

[^radius]: The actual code uses slightly different formula, but the end result is the same.

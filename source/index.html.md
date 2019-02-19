---
title: Ananke

language_tabs: # must be one of https://git.io/vQNgJ
  - shell
  - python

toc_footers:
  - <a href='https://github.com/beiko-lab/ananke'>Ananke GitHub Repository</a>
  - <a href='https://github.com/lord/slate'>Documentation Powered by Slate</a>

includes:

search: true
---

# Introduction

Ananke is a utility for clustering and exploring time series from amplicon gene surveys. 

# Installation

```shell
pip install .
```

The source code can be retrieved from https://github.com/beiko-lab/ananke and installed with pip.

# Initialization


```python
adb = Ananke("ananke_file.h5")
```

First, an `Ananke` object must be created, with a storage file set to store the results:

```python
adb = Ananke("ananke_file.h5")
schema = {"series1": {"replicate1": {"samp1": 0,
                                     "samp2": 1,
                                     "samp3": 4},
                      "replicate2": {"samp4": 0,
                                     "samp5": 1,
                                     "samp6": 4}},
          "series2": {"replicate1": {"samp7": 0,
                                     "samp8": 1,
                                     "samp9": 4},
                      "replicate2": {"samp10": 0,
                                     "samp11": 1,
                                     "samp12": 4}}}
adb.initialize(schema)
```

A `schema` dictionary must be defined, that contains the `{series: {replicate: {sample: time_point} } }` structure of the data. Time point should be a numeric offset value.

# Importing Data From QIIME

```python
adb.import_from_qiime("table.qza")
```

A QIIME2 table .qza artifact can be imported, as long as the sample names in the table match the sample names in the `schema` used to initialize the `Ananke` object. The table artifact can contain a superset of the samples, and Ananke will take only the samples indicated in `schema`. This makes filtering out series or replicates as simple as removing them from the schema and rerunning.

# Importing Data From FASTA

```python
adb.import_from_fasta("seq.fasta", unique_fasta="unique_fasta.fasta", min_count=2)
```

The FASTA file must have labels of the form ">sample_*" where sample is the same as in the `schema` used to initialize the `Ananke` object.
The `unique_fasta` parameter specifies an output file of unique sequences with the labels matching the labels used in the Ananke object. This sequence file should be used for taxonomic classification, so that the results can be merged with the Ananke time series. After the first pass, all unique sequences seen fewer than `min_count` times are immediately rejected. This can be tweaked higher if memory footprint is an issue, or lower if it is not. However, features with only one count are unlikely to yield any useful time-series dynamics.

# Computing Time Series Distances

```python
dbs = adb.precompute_distances("sts", np.arange(0.1, 2.0, 0.1))
adb.save_cluster_result(dbs, "sts")
adb = Ananke("ananke_file.h5")
dbs = adb.load_cluster_result("sts")
```

The distances between time-series must be pre-computed across a range (supplied in the form of a Python `range` or numpy `np.arange` objects). These distances are stored in a series of bloom filter structures to minimize memory impact and maximize the number of features that can be clustered. To reduce computation time and memory usage, supply a smaller range, or a range with larger steps.

Distance measures available are: "sts","euclidean", "dtw", and "ddtw".

The `precompute_distances` function of the `Ananke` objects will return a `DBloomSCAN` object which must be captured into a local variable. This can then be saved to a file, if desired, and loaded later.

# Clustering Time Series

```python
dist = dbs.dist_range[0]
clust = dbs.DBSCAN(dist, expand_around=0, max_members=100)
```

The `DBloomSCAN` object `dbs` contains the relationships between the features at various distances. The DBSCAN algorithm can be run for any distance in the range supplied to `Ananke.precompute_distances()`. With the `expand_around` parameter, the algorithm will return only the single cluster containing the seed time-series indicated by seed index. This tends to be much faster than clustering the entire featurespace. The max_members parameter will return None as soon as the cluster exceeds max_members, which can save significant time if looping over distances and encountering uselessly large clusters, especially for plotting purposes.

# Visualizing Clusters

```python
adb.plot_feature_at_distance(dbs, featureid, eps, title_index=tax_index, max_members=100)
```

This function will return a Plotly Figure object that can be plotted in a Jupyter notebook with iplot(x). This runs DBSCAN on the cluster including featureid, with a point radius of eps, and returns a plot that can be static (in a Python script/shell) or interactive (in a Jupyter notebook).

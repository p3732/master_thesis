# Methods {#methods}
This chapter is comprised of three parts.
In the first, UMAP is profiled in order to determine which parts of the algorithm should be parallelized.
In the second, approaches to parallelizing the found algorithm steps are given.
The last section concludes the chapter with a mapping of the parallelized algorithm steps onto the pseudo code of UMAP.

## Profiling UMAP
This section first identifies UMAP's subtasks and profiles what impact on the running time they have.
Thereafter, the influence of the data set size and dimensionality on these times is explored.
Finally, the expected variance of measured times is estimated by repeated executions.

### Identifying Subtasks of UMAP
To efficiently parallelize an algorithm, it is important to know how well the individual steps of the algorithm perform.
Parallelizing a part that hardly contributes to the algorithm's running time, will not result in a noticeable speedup.
By profiling the algorithm, the parts that take longest to execute can be found.
A parallelization of these parts will have the biggest impact on the performance of the algorithm.

UMAP is a general dimensionality reduction algorithm and is thus applicable to a multitude of data sets.
Throughout the following profiling a wide selection of data sets is used, in order to capture how the performance varies for different inputs.
The data sets are:
the Iris flower data set [@iris],
the Pen Digits data set [@digits],
the COIL-20 [@coil20] and COIL-100 [@coil100] data sets,
the Labeled Faces in the Wild (LFW) data set [@lfw],
the MNIST data set [@mnist],
the Fashion-MNIST (F-MNIST) data set [@fashion-mnist],
the CIFAR-10 data set [@cifar10] and
two subsets of, aswell as the full GoogleNews word vectors data set [@googlenews].
For each data set the number of samples $N$ and dimensionality $M$ is given in the profiling table.
The majority of these data sets are also used to measure algorithm performances in the original UMAP publication [@umap].
Since this thesis focuses on using UMAP for visualization, all data sets are reduced to two dimensions.
All UMAP input parameters are set to the implementation's default values.
<!--, allowing for a validation of the publication's claimed execution times.-->

Before processing, all data sets were made continuous in the main memory, meaning that they were placed consecutively in memory without any fragmentation.
Initial test profilings did not assure the data to be continuous, which resulted in implausibly high execution times.
In these test runs a visualization of the MNIST data was performed 5 times slower.
A processing of the CIFAR-10 data set even took 25 times as long as the continuous version.

Profiling is performed on a system with two 2.3 GHz Intel® Xeon® Gold 5118 CPUs (24 cores) and 256 GB of main memory.
UMAP in version 0.3.8 is used along with Numba version 0.43.1.
The operating system is Ubuntu 16.04 with kernel version 4.4.0 and Python 3.7.3.

The standard Python profiler [^pyprof] is used for accurate timing.
It measures the time spent on each function of the program.
This enables the identification of program parts that take long to execute and therefore are good candidates for parallelization.

![Profiling of UMAP on a 500.000 points subset of GoogleNews.](figures/chapter3/tuna/umap_fit.png){#fig:tuna-gnews short-caption="Profiling of UMAP on a 500.000 points subset of GoogleNews." width="100%"}

@fig:tuna-gnews shows a visualization of profiling data generated by processing a 500.000 points subset of the GoogleNews data set.
The graphic is taken from the interactive output that was generated by the `tuna` Python package[^tuna_src] for the profiling data.
As shown, the total runtime is dominated by few functions.
The percentages in the graphic do not sum up to 100%, because they include loading times of the data set and libraries.
Nonetheless, more than half of the time is spent in the `simplicial_set_embedding` subtask.
It performs the initialization and optimization of the low-dimensional representation.
Other significant contributions are made by the `nearest_neighbors` subtask, which performs a KNN search, and the `fuzzy_simplicial_set` subtask, which normalizes distances between data points found by the KNN search (as illustrated in @fig:umap_radii).

UMAP uses Numba [@numba], a Just-In-Time (JIT) compiler.
It compiles source code "on the fly" during execution of the program.
Consequently the time taken for these compilations also accounts for parts of the run time.
This time is not prominently present in @fig:tuna-gnews, since it is comparatively low and compiling is done at several occasions during the execution.

\pagebreak

In Table \ref{umap_various_times}, the times measured in the UMAP publication (Publ.) are compared to the times of the profiling (Total).
The latter is further broken down into time spent on each of the dominating subtasks, `simplicial_set_embedding` (Embed), `nearest_neighbors` (KNN) and `fuzzy_simplicial_set` (Fuzzy), along with accumulated time spent on compiling (JIT).

Data set          N     M  Publ.   Total     KNN   Fuzzy   Embed    JIT
------------- ----- ----- ------ ------- ------- ------- ------- ------
Iris            150     4           9.26    0.01    6.31    2.58   8.76
COIL-20        1440 16384     12   13.10    0.09    6.77    5.64   8.48
Digits         1797    64      9   14.49    0.18    7.19    6.87   8.74
LFW           13233  2914          47.84   15.98   12.16   18.64  11.64
COIL-100       7200 49152     85   90.86   55.59    9.66   21.61  11.89
MNIST          70 K   784     87     162   34.24   39.10   86.00  11.70 
F-MNIST        70 K   784     65     164   35.44   38.85   87.58  11.57
CIFAR          60 K  3072            200   79.84   34.34   82.30  11.29
GoogleNews    200 K   300    361     585     142     110     326  11.96
GoogleNews    500 K   300           1577     427     242     894  11.45
GoogleNews      3 M   300          14226    6277    1614    6249  12.04

Table: Profiling of UMAP on various data sets. \label{umap_various_times}

The table shows differences between the times of the publication and the profiling.
This is mostly due to the difference in hardware being used.
Most parts of UMAP are sequential, therefore the times of the publication, that were measured on an ordinary consumer CPU, outperform the server hardware used for the profiling, since the publication used a CPU with a higher clock rate.
It can be noticed that the difference is smaller for data sets with high dimensionality.
This could be caused by recent changes to the implementation or those data sets requiring more computations that are parallelized for UMAP.

For the full GoogleNews data set, the KNN search and the embedding subtask take equally long, but for most smaller data sets the embedding subtasks dominates.
An exception to this is the COIL-100 data set, here the KNN subtask requires more than twice as long as the embedding.
It seems that the KNN subtask is strongly influenced by the dimensionality of the data.
The differences in JIT times are due to an optimization of the implementation, which uses a non-compiled KNN algorithm for small data sets.

![Profiling of UMAP on varying sizes and dimensions of CIFAR.](figures/chapter3/3d/plot_umap_cifar_total.png){#fig:prof_n_and_m_cifar width="80%"}

### Influence of Data Set Size and Dimensionality
To estimate the influence of the data set's number of samples and dimensions on the running time, a separate profiling is performed.
UMAP is hereby applied to reduced versions of the CIFAR data set.
A reduction is made in equidistant steps for both, the data set size and dimensionality.
The size is reduced in steps of 10.000 data points, the dimensionality is reduced in steps of 25% of the original number of dimensions.
This is done by dropping rows and columns of the data set respectively.
CIFAR was chosen, because it is big enough to allow for the creation of meaningful subsets through reduction, while still being small enough to be processed repetitively in a feasible time frame.

As the visualized profiling results of @fig:prof_n_and_m_cifar show, the total running time of UMAP is stronger influenced by the amount of samples, than by the dimensionality.
Processing only half of all data points reduces the running time more, than processing the same amount of data points with only a fourth of the dimensions.
@fig:prof_n_and_m_methods shows how this is reflected in the individual subtasks of UMAP.
It shows the contribution of each subtask that resulted in the total running time of @fig:prof_n_and_m_cifar.

<div id="fig:prof_n_and_m_methods" class="subfigures">
![`nearest_neighbors`](figures/chapter3/3d/plot_umap_cifar_NN.png){width=49% #fig:prof_n_and_m_methods_a}\hfill
![`fuzzy_simplicial_set`](figures/chapter3/3d/plot_umap_cifar_fuzzy_set.png){width=49% #fig:prof_n_and_m_methods_b}

![`simplicial_set_embedding`](figures/chapter3/3d/plot_umap_cifar_embed.png){width=49% #fig:prof_n_and_m_methods_c}\hfill
![JIT compiling](figures/chapter3/3d/plot_umap_cifar_compile.png){width=49% #fig:prof_n_and_m_methods_d}

Time spent by individual subtasks processing reduced CIFAR data sets.
</div>

Only the `nearest_neighbors` subtask is affected by the dimensionality of the data.
The `simplicial_set_embedding` subtask solely computes on the low-dimensional embedding and thus is independent of the dimensionality.
It shows linear growth with the data set size.
The `fuzzy_simplicial_set` subtask has the same properties, albeit growing slower.
It normalizes all distances between nearest-neighbors, which are stored in a sparse matrix.
Thus it is not affected by the dimensionality either.
JIT compiling always takes approximately the same amount of time and is not affected by the input data.

A comparison of these time measurements is given in @fig:prof_n_and_m_2d_cifar.
Here, solely the data set size has been altered, while the original dimensionality was kept.
The drop in time of the `simplicial_set_embedding` subtask is caused by UMAP performing a lower amount of embedding optimization iterations for data sets bigger than 100.000 data points.

![Comparison of UMAP subtasks on reduced CIFAR data sets.](figures/chapter3/plot_2d_cifar.png){width=90% #fig:prof_n_and_m_2d_cifar}

\pagebreak
### Variation of Execution Time
To analyze how deterministic the UMAP algorithm behaves, a final profiling is done.
Here, UMAP was performed on subsets of the CIFAR data set a total of 10 times.
From the measured execution times the standard deviation is calculated.
@fig:determinism displays the average execution time with the standard deviation displayed as a span on each measured data point. The variation seems to grow linearly with the total execution time.
This suggests that the runtime of UMAP is rather deterministic and only varies within a certain range of expected execution time.

![Standard deviation of UMAP execution times on reduced CIFAR data sets.](figures/chapter3/deterministic_cifar.png){width=80% #fig:determinism}

[^pyprof]: https://docs.python.org/3/library/profile.html, accessed 25.04.2019
[^tuna_src]: https://pypi.org/project/tuna/, accessed 25.04.2019
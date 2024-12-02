<!-- README.md is generated from README.Rmd. Please edit that file -->

# PathPinpointR

<!-- badges: start -->
<!-- badges: end -->

PathPinpointR identifies the position of a sample upon a trajectory.

##### *Assumptions:*

-   Sample is found upon the chosen trajectory.
-   Sample is from a distinct part of the trajectory. A sample with
    cells that are evenly distributed across the trajectory will have a
    predicted location of the centre of the trajectory.

# Example Workflow

This vignette will take you through the basics running PPR. The data
used here is generated in a similar fasion to the [Slingshot
vignette](https://bioconductor.org/packages/devel/bioc/vignettes/slingshot/inst/doc/vignette.html).

## Installation

#### Check and install required packages

Run the following code to load all packages neccecary for PPR & this
vignette.

    required_packages <- c("SingleCellExperiment", "Biobase", "fastglm", "ggplot2",
                           "monocle", "plyr", "RColorBrewer", "ggrepel", "ggridges",
                           "gridExtra", "devtools", "mixtools", "Seurat",
                           "parallel", "RColorBrewer", "mclust")

    ## for package "fastglm", "ggplot2", "plyr", "RColorBrewer",
    # "ggrepel", "ggridges", "gridExtra", "mixtools"
    new_packages <- required_packages[!(required_packages %in% installed.packages()[,"Package"])]
    if(length(new_packages)) install.packages(new_packages)

    ## for package "SingleCellExperiment", "Biobase", "slingshot".
    if (!requireNamespace("BiocManager", quietly = TRUE)) install.packages("BiocManager")
    new_packages <- required_packages[!(required_packages %in% installed.packages()[,"Package"])]
    if(length(new_packages)) BiocManager::install(new_packages)

    devtools::install_github("SGDDNB/GeneSwitches")

#### install PathPinpointR

You can install the development version of PathPinpointR using:

    devtools::install_github("moi-taiga/PathPinpointR")

### Load the required packages

    library(PathPinpointR)
    library(Seurat)
    library(ggplot2)
    library(SingleCellExperiment)
    library(slingshot)
    library(RColorBrewer)
    library(GeneSwitches)
    library(mclust)

## Generate the data

The data is generated in a similar fasion to the [Slingshot
vignette](https://bioconductor.org/packages/devel/bioc/vignettes/slingshot/inst/doc/vignette.html).
A subset of the reference data is used as a proxy for the sample data.

    load_all()
    ### Genertate the reference data
    reference_sce <- get_synthetic_data()
    # Generate the sample data by subsetting the reference data
    # in one sample inlcude c20-c45, in the other c90-c128.
    samples_sce <- list(reference_sce[,c(20:45)], reference_sce[,c(90:128)]) 

#### View the PCA plot from the reference data

Plot the reference data, colored by cluster.

    plot(reducedDims(reference_sce)$PCA,
                     col = brewer.pal(9,"Set1")[colData(reference_sce)$GMM],
                     pch=16, asp = 1)
    # add a legend
    legend("topright",
           legend = levels(colData(reference_sce)$clust_names),
           fill = brewer.pal(9,"Set1"))

<img src="./man/figures/SLING-reference-PCA-plot.png" width="100%" />
The plot shows the development of the cells.

## Run slingshot

Run slingshot on the reference data to produce pseudotime for each cell.

    # Slingshot
    reference_sce <- slingshot(reference_sce,
                     clusterLabels = 'GMM',
                     reducedDim = 'PCA')

    #Rename the Pseudotime column to work with GeneSwitches
    colData(reference_sce)$Pseudotime <- reference_sce$slingPseudotime_1

#### Plot the slingshot trajectory.

    # Generate colors
    colors <- colorRampPalette(brewer.pal(11, "Spectral")[-6])(100)
    plotcol <- colors[cut(reference_sce$slingPseudotime_1, breaks = 100)]
    # Plot the data
    plot(reducedDims(reference_sce)$PCA, col = plotcol, pch=16, asp = 1)
    lines(SlingshotDataSet(reference_sce), lwd = 2, col = "black")

<img src="./man/figures/SLING-slingshot.png" width="100%" /> The plot
shows the trajectory of the reference data, with cells colored by
pseudotime.

## Binarize the reference Expression Data

Using the package
[GeneSwitches](https://github.com/SGDDNB/GeneSwitches), binarize the
gene expression data of the reference data.

    # binarize the expression data of the reference
    reference_sce <- binarize_exp(reference_sce,
                                  fix_cutoff = TRUE,
                                  binarize_cutoff = 0.4,
                                  ncores = 1)

    # Find the switching point of each gene in the reference data
    reference_sce <- find_switch_logistic_fastglm(reference_sce,
                                                  downsample = TRUE,
                                                  show_warning = FALSE)

Note: both binatize\_exp() and find\_switch\_logistic\_fastglm() are
time consuming processes and may take tens of minutes to run.

## Visualise the switching genes

generate a list of switching genes, and visualise them on a pseudo
timeline.

    switching_genes <- filter_switchgenes(reference_sce,
                                          allgenes = TRUE,
                                          r2cutoff = 0)

    # Plot the timeline using plot_timeline_ggplot
    plot_timeline_ggplot(switching_genes,
                         timedata = colData(reference_sce)$Pseudotime,
                         txtsize = 3)

<img src="./man/figures/SLING-timeline-plot1.png" width="100%" /> The
distribution of the switching genes along the trajectory is shown.  
Due to the nature of this syntheic data the switching gnenes are not
evenly distributed.  
Note: The number of switching genes significantly affects the accuracy
of PPR.  
too many will reduce the accuracy by including uninformative
genes/noise.  
too few will reduce the accuracy by excluding informative genes.  

## Select a number of switching genes

Using the PPR function precision() an optimum number of switching genes
can be found.

    precision(reference_sce, n_sg_range = seq(0, 800, 100))

<img src="./man/figures/SLING-precision1_plot.png" width="100%" />

Narrow down the search to find the optimum number of switching genes.

    precision(reference_sce, n_sg_range = seq(700, 822, 2))

<img src="./man/figures/SLING-precision2_plot.png" width="100%" />

## Produce a filtered matrix of switching genes

The using precision(), 772 is found to be the optimum for this data.  
When using true biological data, the optimum number is likely to be
lower.  

    switching_genes <- filter_switchgenes(reference_sce,
                                          allgenes = TRUE,
                                          r2cutoff = 0,
                                          topnum = 772)

## Visualise the filtered switching genes

    # Plot the timeline using plot_timeline_ggplot
    plot_timeline_ggplot(switching_genes,
                         timedata = colData(reference_sce)$Pseudotime,
                         txtsize = 3)

<img src="./man/figures/SLING-timeline-plot2.png" width="100%" />

## Binarize the sample data

Binarize the gene expression data of the samples.

    # First reduce the sample data to only include the switching genes.
    samples_sce <- lapply(samples_sce, reduce_counts_matrix, switching_genes)

    # binarize the expression data of the samples
    samples_binarized <- lapply(samples_sce,
                                binarize_exp,
                                fix_cutoff = TRUE,
                                binarize_cutoff = 1,
                                ncores = 1)

## Predict Position

Produce an estimate for the position of each cell in each sample. The
prediction is stored as a PPR\_OBJECT.

    reference_ppr <- predict_position(reference_sce, switching_genes)

    # Iterate through each Seurat object in the predicting their positons,
    # on the reference trajectory, using PathPinpointR.
    samples_ppr <- lapply(samples_binarized, predict_position, switching_genes)

## Measure accuracy

We can calculate the accuracy of PPR in the given trajectory by
comparing the predicted position of the reference cells to their
pseudotimes defined by slingshot.

    accuracy_test(reference_ppr, reference_sce, plot = TRUE)

<img src="./man/figures/SLING-accuracy_plot.png" width="100%" />

## Plotting the predicted position of each sample:

plot the predicted position of each sample on the reference trajectory.

    # show the predicted position of the first sample
    # include the position of cells in the reference data, by a given label.
    ppr_plot() +
      reference_idents(reference_sce, "clust_names") +
      sample_prediction(samples_ppr[[1]], label = "Sample 1", col = "red")

    # show the predicted position of the second sample
    sample_prediction(samples_ppr[[2]], label = "Sample 2", col = "blue")

    # show the points at which selected genes switch.
    switching_times(c("G1172", "G1346", "G901"), switching_genes)

<img src="./man/figures/Sling-PPR.png" width="100%" />
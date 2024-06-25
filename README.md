
<!-- README.md is generated from README.Rmd. Please edit that file -->

# PathPinPointR

<!-- badges: start -->
<!-- badges: end -->

The goal of PathPinpointR is to identify the position of a sample upon a
trajectory.

### Limitations:

### Assumptions:

- Sample is found upon the chosen trajectory.
- Sample is from a distinct part of the trajectory. A sample with cells
  that are evenly distributed across the trajectory will have a
  predicted location of the centre of the trajectory.

# Example Workflow

#### This vignette will take you through running PPR. 

##### The data used here is an intregrated data-set of [blastocyst data](http://). 

## Installation

You can install the development version of PathPinpointR using:

``` r
# install.packages("devtools")
devtools::install_github("moi-taiga/PathPinPointR")
```

### Load neccecary packages

``` r
library(Seurat)
library(ggplot2)
library(SingleCellExperiment)
library(slingshot)
library(RColorBrewer)
library(GeneSwitches)
library(Parallel)
library(devtools)
devtools::load_all()
```

### Here we use this Reprogramming dataset as the reference

``` r
seu <- readRDS("~/data/blastocyst.rds")
```

#### View the reference UMAP

``` r
DimPlot(object = seu,
             reduction = "umap",
             group.by = "orig.ident",
             label = TRUE) +
     ggtitle("Reference")
```

### We use subsets of the Reprogramming dataset as queries.

``` r
# Split the reference data into samples

# Make a list of the unique sample names
sample_names <- unique(seu@meta.data$orig.ident)

# Make an empty list to store the samples
samples_seu <- list()

# Iterate through each sample name, make a list of subsets.
for (sample in sample_names){
  sample_seu <- subset(x = seu, subset = orig.ident %in% sample)
  samples_seu[[sample]] <- sample_seu
}
```

### Convert seurat objects to SingleCellExperiment objects

``` r
sce    <- SingleCellExperiment(assays = list(expdata = seu@assays$RNA$counts))
colData(sce) <- DataFrame(seu@meta.data)
reducedDims(sce)$UMAP <- seu@reductions$umap@cell.embeddings

for (sample in sample_names){
  sample.sce <- SingleCellExperiment(assays = list(expdata = get(paste0(sample, ".seu"))@assays$RNA$counts))
  assign(paste0(sample, ".sce"), sample.sce)
}
```

### Run slingshot on the reference data to produce a reprogramming trajectory.

``` r
sce  <- slingshot(sce,
                  clusterLabels = "seurat_clusters",
                  start.clus  = "2",
                  end.clus = "1",
                  reducedDim = "UMAP")

#Rename the Pseudotime column to work with GeneSwitches
colData(sce)$Pseudotime <- sce$slingPseudotime_1
```

### Plot the slingshot trajectory.

``` r
# Generate colors
colors <- colorRampPalette(brewer.pal(11, 'Spectral')[-6])(100)
plotcol <- colors[cut(sce$slingPseudotime_1, breaks = 100)]


# Plot the data
plot(reducedDims(sce)$UMAP, col = plotcol, pch = 16, asp = 1)
lines(SlingshotDataSet(sce), lwd = 2, col = 'black')
```

## GeneSwitches

### Choose a Binerization cutoff

``` r
# hist(as.matrix(assays(sce)$expdata),
#            breaks = 1e6)
#  hist(as.matrix(seu@assays$RNA$counts), breaks = 1e6)
#  
#  hist(as.matrix(assays(sce)$expdata),
#            breaks = 1e6,
#            xlim = c(0, 50))
#  
#  hist(as.matrix(seu@assays$RNA$counts), breaks = 1e6, xlim =c(0, 50))
#  
#  trimmed_expdata <- assays(sce)$expdata[assays(sce)$expdata < 10]
#  hist(as.matrix(trimmed_expdata))
#  abline(v=1, col = "blue")
#  
#  trimmed_expdata <- seu@assays$RNA$counts[seu@assays$RNA$counts < 10]
#  hist(as.matrix(trimmed_expdata))
#  abline(v=1, col = "blue")
#cutoff =1
 
```

### Binarize

``` r
#sce    <- binarize_exp(sce, fix_cutoff = TRUE, binarize_cutoff = 1, ncores = 64)
#Mole21a.sce   <- binarize_exp(Mole21a.sce, fix_cutoff = TRUE, binarize_cutoff = 1, ncores = 64)
#sce <- binarize_exp(sce, ncores = 64)
#Mole21a.sce   <- binarize_exp(Mole21a.sce, ncores = 64)
```

### CHECKPOINT

##### \*\*with how long binerizing takes you may want to choose to save objects for future use.

``` r
 #save the reference and sample objects
 #saveRDS(sce   , "/mainfs/ddnb/PathPinpointR/package/PathPinpointR/data/reference.rds")
 #for (sample in sample_names){
 #       saveRDS(get(paste0(sample, ".sce")), paste0("/mainfs/ddnb/PathPinpointR/package/PathPinpointR/data/binarized_", sample, "_sce.rds"))
 #}

# read the reference and sample objects
#sce <- readRDS("/mainfs/ddnb/PathPinpointR/package/PathPinpointR/data/reference.rds")
#sample_Mole21a_binarized <- readRDS("/mainfs/ddnb/PathPinpointR/package/PathPinpointR/data/binarized_Mole21a_sce.rds")

samples_binarized <- list()
for (sample in sample_names){
  samples_binarized[[sample]] <- readRDS(paste0("/mainfs/ddnb/PathPinpointR/package/PathPinpointR/data/binarized_", sample, "_sce.rds"))
}
```

### fit logistic regression and find the switching pseudo-time point for each gene

``` r
#sce <- find_switch_logistic_fastglm(sce, downsample = FALSE, show_warning = FALSE)
```

#### **This is another time consuming process, may want to choose to save objects for future use.**

``` r
## saveRDS(sce   , "~/R Packages/mini_data/Binerized/reference_glm.rds")
sce    <- readRDS("/mainfs/ddnb/PathPinpointR/package/PathPinpointR/data/switches_gastglm_blastocyst_reference_sce.rds")
```

### Filter to only include Switching Genes

``` r
switching_genes <- filter_switchgenes(sce, allgenes = TRUE,r2cutoff = 0.26)
```

##### View all of the switching genes

``` r
# Plot the timeline using plot_timeline_ggplot
plot_timeline_ggplot(switching_genes, timedata = colData(sce)$Pseudotime, txtsize = 3)
```

# Using PPR

##### View the selected switching genes

``` r
ppr_timeline_plot(switching_genes)
```

### Reduce the binary counts matricies of the query data to only include the selection os switching genes.

``` r
# filter the binary counts matricies to only include the switching genes
reference_reduced <- subset_switching_genes(sce, switching_genes)
#Mole21a_reduced   <- subset_switching_genes(sample_Mole21a_binarized, switching_genes)

# Define a list to store the results
samples_reduced <- list()

# Iterate through each Seurat object in the list
for (sample_name in names(samples_binarized)){
  sample_binarized <- samples_binarized[[sample_name]]
  # susbet each sample to only include the switching genes
  sample_reduced <- subset_switching_genes(sample_binarized, switching_genes)
   # Store the result in the new list
  samples_reduced[[sample_name]] <- sample_reduced
}
```

### Produce an estimate for the position on trajectory of each gene in each cell of a sample.

``` r
reference_ppr    <- ppr_predict_position(reference_reduced, switching_genes)
#Mole21a_ppr      <- ppr_predict_position(Mole21a_reduced, switching_genes)
# Define a list to store the ppr objects of the samples
samples_ppr <- list()

# Iterate through each Seurat object in the predicting their positons,
# on the reference trajectory, using PathPinpointR
for (sample_name in names(samples_reduced)){
  sample_reduced <- samples_reduced[[sample_name]]
  # predict the position of each gene in each cell of the sample
  sample_ppr <- ppr_predict_position(sample_reduced, switching_genes)
  # Store the result in the new list
  samples_ppr[[sample_name]] <- sample_ppr
}
```

### Accuracy

#### *As our samples are subsets of the reference dataset we can calculate the accuracy of GSS*

``` r

#ppr_accuracy_test(Mole21a_ppr, sce, plot = TRUE)
ppr_accuracy_test(reference_ppr, sce, plot = TRUE)


for (sample_name in names(samples_ppr)){
  sample_ppr <- samples_ppr[[sample_name]]
 
  ppr_accuracy_test(sample, sce, plot = TRUE)
  
}
```

## Plotting

### Plot the predicted position of each sample:

#### *Optional: include the switching point of some genes of interest:*

``` r



ppr_output_plot(reference_ppr, col = "red", overlay=FALSE, label = "Reference")


for (sample_name in names(samples_ppr)){
  sample_ppr <- samples_ppr[[sample_name]]
  
  ppr_output_plot(sample_ppr, col = "red", overlay=FALSE, label = sample_name)
  
}

pdf("/mainfs/ddnb/PathPinpointR/package/PathPinpointR/data/plots/samples_ppr_output.pdf")
ppr_output_plot(samples_ppr[[1]], col = "red", overlay=FALSE, label = names(samples_ppr)[1])
ppr_output_plot(samples_ppr[[2]], col = "blue", overlay=TRUE, label = names(samples_ppr)[2])
ppr_output_plot(samples_ppr[[4]], col = "purple", overlay=TRUE, label = names(samples_ppr)[4])
ppr_output_plot(samples_ppr[[5]], col = "#000000", overlay=TRUE, label = names(samples_ppr)[5])
ppr_output_plot(samples_ppr[[8]], col = "#4dd80c", overlay=TRUE, label = names(samples_ppr)[8])
# Close the PDF device
dev.off()
```

#### Plot an indervidual cell.

``` r
ppr_cell_plot(Mole21a_ppr, cell_idx = 1)
```

``` r
ppr_timeline_plot(switching_genes,
                  genomic_expression_traces = TRUE,
                  reduced_binary_counts_matrix = Mole21a_reduced,
                  cell_id = 1)
```

### Precision

##### Estimate the precision of PPR by running accuracy_test at a range of values

###### BUG!

``` r
ppr_precision(sce)
```

### Investigate the predicted position of indervidual cells:

``` r
#
plot(Mole21a_accuracy$true_position_of_cells_timeIDX, Mole21a_accuracy$inaccuracy,
     xlab = "True Position", ylab = "Inaccuracy", main = "Inaccuracy by True Positions")

#
boxplot(Mole21a_accuracy$inaccuracy ~ Mole21a_accuracy$true_position_of_cells_timeIDX,
        xlab = "True Position", ylab = "Inaccuracy", main = "Boxplots of Inaccuracy by True Positions")
true_position_bin <- cut(Mole21a_accuracy$true_position_of_cells_timeIDX,
                             breaks = c(0,5,10,15,20,25,30,35,40,45,50,55,60,65,70,75,80.85,90,95,100))
boxplot(Mole21a_accuracy$inaccuracy ~ true_position_bin,
        xlab = "True Position", ylab = "Inaccuracy", main = "Boxplots of Inaccuracy by True Positions")

#
plot(Mole21a_accuracy$true_position_of_cells_timeIDX,Mole21a_accuracy$predicted_position_of_cells_timeIDX,
     xlab = "True Position", ylab = "Predicted Position", main = "Predicted Positions by True Positions")
segments(0,0,100,100, lwd=2, col ="green")
segments(0,10,100,110, lwd=2, col ="blue")
segments(0,-10,100,90, lwd=2, col ="blue")
segments(0,40,100,140, lwd=2, col ="red")
segments(0,-40,100,60, lwd=2, col ="red")

#
boxplot(Mole21a_accuracy$predicted_position_of_cells_timeIDX ~ Mole21a_accuracy$true_position_of_cells_timeIDX,
        xlab = "True Position", ylab = "Predicted Position", main = "Boxplots of Predicted Positions by True Positions")
segments(0,0,100,100, lwd=2, col ="green")
segments(0,10,100,110, lwd=2, col ="blue")
segments(0,-10,100,90, lwd=2, col ="blue")
segments(0,40,100,140, lwd=2, col ="red")
segments(0,-40,100,60, lwd=2, col ="red")
```

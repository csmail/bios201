# Workshop 3: Differential expression analysis of RNA-seq data

## Introduction

Last week, we mapped RNA-seq reads from a 
[study](http://journals.plos.org/plosone/article/file?type=supplementary&id=info:doi/10.1371/journal.pone.0097550.s002) 
comparing lung tissues taken from diseased and healthy individuals. This week, we’ll pick up the analysis where 
we left off. 

We will identify genes differentially expressed between IPF and normal lung tissues. 
Broadly speaking, this might allow us to 1) come up with diagnostic genetic screens 
for IPF susceptibility or incidence, or 2) infer possible mechanisms of the disease for further follow-up.

We'll use the R package _DESeq2_ to identify differentially expressed genes.

## Goals

This workshop will enable you to do the following:

* Compare expression across samples
* Perform principal component analysis (PCA)
* Remove the effects of known experimental confounders
* Identify differentially expressed genes with DESeq2
* Verify and follow up DESeq2 results

## Setup

We'll be doing today's workshop entirely in RStudio; however, we first need to copy a few data files from
corn to your local machine. Using Command Prompt in Windows or Terminal in Mac, `cd` to the folder
in which you want to copy the data files. You should copy the whole workshop3 folder from corn:

On Mac:
```
scp -r <yourname>@corn.stanford.edu:/afs/.ir/class/bios201/workshop3 .
```

Or on Windows:
```
pscp -r <yourname>@corn.stanford.edu:/afs/.ir/class/bios201/workshop3 .
```

Now open RStudio. In case you want to rerun or modify your code, we recommend typing your code into
the Source panel. Remember, to execute a line of your code, type Ctrl-Enter or
Command-Enter. You can also execute the entire file at once by clicking the Source button in the
upper-right corner of the panel.

First, you need to install two R packages, DESeq2 and pheatmap. If you completed last week's workshop, you'll
already have both installed. If not, you can install them using the following code:

```
source("https://bioconductor.org/biocLite.R")
biocLite("DESeq2")
install.packages("pheatmap")
```

Then load the packages.

```
require(DESeq2)
require(pheatmap)
```

Now set the working directory to the one where you copied your data:

```
setwd("<your_data_directory>")
```


## Exploratory Data Analysis

Before we start testing for differential expression, it's always important to
first visualize our data and perform quality control checks. We'll start
by looking at `counts.txt`, which contains the log-transformed IPF gene expression from last week with
some simulated sequencing batch effects.

Read in the matrix of log-transformed gene expression counts. 
Verify using the `head` command that your counts matrix loaded properly.

```
log_counts = read.table("counts.txt")
head(log_counts)
```

As you saw at the end of the last workshop, one of our samples, "Norm7",
was not actually from lung tissue; this sample doesn't express the lung
surfactant proteins. We've already removed this sample from the analysis,
so you won't have to worry about it.

First, check the size of your data using `dim(log_counts)`.

:question: How many genes are measured in this dataset?

If there's a significant difference between gene expression in healthy and diseased
individuals, then we should see that samples of similar types are more correlated
with one another than with samples of different types. We can test whether this is the case by making 
a scatterplot in R.

```
# Compare counts in two normal samples
plot(log_counts[,1], log_counts[,3])
abline(0,1)
```

Now compute the correlation of gene expression between these two samples.
```
cor(log_counts[,1], log_counts[,3])
```

Now, modify the previous code to make a scatterplot comparing 
the expression in a healthy sample and a diseased sample. (Remember, you can
select the _n_-th column of your `log_counts` matrix by typing `log_counts[,n]`).

:question: Which pair of samples has greater correlation?

We can also compute the pairwise correlation between all the different samples at once:

```
cor(log_counts)
```

Now view this matrix as a heatmap:

```
log_cor <- cor(log_counts)
diag(log_cor) <- NA
pheatmap(log_cor, show_rownames = TRUE)
```

:question: Do the healthy and diseased individuals separate into distinct clusters?

Even if the healthy and diseases samples don't separate into perfect clusters, that's
not always a problem. But plotting heatmaps can help us to identify batch effects
in our data.

## Detecting batch effects

As we noticed in the last section, some of the similar samples cluster together,
but we don't observe two well-defined groups of diseased and healthy samples.
In this case, it's because we've spiked in a simulated sequencing batch effect, which
causes some of the samples from the same sequencing run to appear more
similar than they really are.

[Batch effects](http://www.molmine.com/magma/global_analysis/batch_effect.html) are
unwanted patterns in sequencing data that arise because of the way the samples
have been handled before or during sequencing. For example, if some of the samples
were left on the bench overnight but others were left in the freezer, then we might
observe serious batch effects. Similarly, if samples were analyzed in different sequencing
runs (or even different lanes within the same sequencing run) batch effects
may be visible.

If we're not careful, batch effects can completely overpower the signal we're interested
in observing. In some of the worst cases, researchers have mistaken batch effects for biological
signal and didn't notice this problem until long after publication. Luckily, there are several
precautions we can take to minimize the risk of false results due to batch effects.

### Loading covariates

It's always a good idea to note which samples were sequenced on which sequencing run,
as well as other important metadata. We've supplied a file `covariates.txt` that contains
some such information. Load it into R:

```
covariates = read.table("covariates.txt")
```

Take a look at the covariates.

:question: How many IPF females are included in this dataset?


### Principal Component Analysis

A method called [Principal Component Analysis](https://en.wikipedia.org/wiki/Principal_component_analysis), 
or PCA, is used to detect and visualize batch effects in sequencing data. 
When batch effects or other effects due to experimental covariates are present in the data,
they can cause major variation in many of the samples' gene expression; however, if we're lucky,
the same set of genes will be affected similarly across all the affected samples. (For example,
if you leave your samples out on the bench overnight, you might expect that certain RNAs will degrade
more rapidly than other long-lasting RNAs, so the former RNAs will appear underexpressed in all of the
affected samples.)

PCA is an algorithm that identifies sets of genes that vary similarly across samples. At a high level,
PCA finds sources of variation across samples, and then outputs values called _principal components_ that
represent the sets of varying genes and to what extent they are present in each sample. If batch effects
are present, they will often be detectable by looking at the first few principal components of the gene expression
matrix.

Let's try PCA on our RNA-seq data.

```
pca = prcomp(log_counts)
```

We can visualize the first two principal components as follows. Our points represent samples,
and are color-coded by disease status.

```
# A simple function for color-coding points
color_code <- function(x, vals, colors)
{
  i = which(x == vals)
  return(colors[i])
}


# Plot PC1 and PC2
color_status = sapply(substring(covariates$samples, 1, 3), FUN=color_code,
                      vals=c("IPF", "Nor"), colors=c("blue", "red"))
plot(pca$rotation[,1], pca$rotation[,2], col=color_status, pch=16)
```

We can also plot the samples color-coded by sequencing batch:
```
color_seqbatch = sapply(covariates$seq.batch, FUN=color_code,
                        vals=c(1, 2), colors=c("blue", "red"))
plot(pca$rotation[,1], pca$rotation[,2], col=color_seqbatch, pch=16)
```

:question: Which variable (disease status or sequencing batch) causes a stronger
clustering effect in our data?

:question: Which of the principal components (#1 or #2) separates the samples
by this variable?

As you can see, there's a clear batch effect in our data. However, we can still attempt
to explore the disease's influence on expression by simply ignoring the effects of these first few principal components.
Check out PC3 and PC4 instead.

```
# Plot PC3 and PC4
plot(pca$rotation[,3], pca$rotation[,4], col=color_status, pch=16)
```

:question: Now do you see any separation of points based on disease status?

If we plot the correlation heatmap after removing the first 2 principal components, we now
see that the samples cluster more nicely by disease status:
```
pca_mat = cor(t(as.matrix(pca$rotation[,3:6])))
diag(pca_mat) = NA
pheatmap(pca_mat)
```

Removing batch effects is an important first step in any gene expression analysis.
It can be particularly tricky when the experimental covariates are not recorded;
however, software tools such as [PEER](https://www.ncbi.nlm.nih.gov/pubmed/22343431) can 
be used to detect unknown covariates.


## Differential expression

Now that we've eliminated batch effects, we're ready to test for differentially expressed genes.
We achieve this using an R tool called [DESeq2](https://genomebiology.biomedcentral.com/articles/10.1186/s13059-014-0550-8)
that quantifies the extent to which gene expression changes between conditions.

As input, DESeq2 takes non-normalized counts values. DESeq2 statistically models the
probability of differential expression, which involves placing greater confidence on
differentially expressed genes for which a greater number of counts are observed across
samples. To simplify this part of the analysis, we've supplied the original read count data
from last week's exercise with simulated sequencing batch effects removed. It is possible that
other sequencing batch effects still remain, since we've never explicitly corrected these. Normally we
should attempt to detect these batch effects using a tool such as PEER for identifying "hidden" effects;
for today's workshop, we'll ignore this aspect of the analysis.

Load the denoised expression counts from `counts_denoised.txt` and use these counts for the remainder of the workshop.

```
counts = read.table("counts_denoised.txt")
```

Run DESeq using the following commands:

```
rownames(covariates) = covariates$samples
dds <- DESeqDataSetFromMatrix(countData <- counts,
                              colData <- covariates,
                              design = ~ factor(status, levels=c("Norm","IPF")))
dds <- DESeq(dds, betaPrior=FALSE)
res_nosex = results(dds)
plotMA(dds)
```

DESeq produces an _MA plot_ that shows the estimated log fold change in gene expression in
diseased individuals relative to healthy individuals, plotted against the mean expression of the gene across samples.
Points plotted in red are significantly differentially expressed between samples. 

:question: What is the relationship between the mean expression and the magnitude of log fold change?

:question: Do you see more highly overexpressed or underexpressed genes?

We can also directly correct for known covariates using DESeq2. Here we correct
any effects due to the individual's sex. Note that we've added an additional
term to the design formula.

```
dds <- DESeqDataSetFromMatrix(countData <- counts,
                              colData <- covariates,
                              design = ~ sex + factor(status, levels=c("Norm","IPF")))
dds <- DESeq(dds, betaPrior=FALSE)
res = results(dds)
plotMA(dds)
```

Compare the strength of results between the sex-corrected DESeq2 run and the
run with no covariates corrected.

```
plot(log(res_nosex$pvalue, base=10), log(res$pvalue, base=10), xlim=c(-30,0), ylim=c(-30,0),
            xlab="p-value, No sex correction", ylab="p-value, sex correction")
abline(a=0,b=1)
abline(a=-5,b=0,lty=3,col="red")
abline(v=-5,lty=3,col="red")
```

:question: Are there any genes that become significant after correcting for sex? (Use the
dashed red line as a cutoff for significance.) Do any that were initially significant become
insignificant after correction?

Let's see how many genes are differentially expresssed. DESeq provides two different p-values, `pvalue`
and `padj`. When searching for differential expression across many genes, you should always use `padj`,
which corrects for [multiple testing](http://www.stat.berkeley.edu/~mgoldman/Section0402.pdf).

```
sum(res_nosex$padj < 0.05, na.rm=TRUE)
sum(res$padj < 0.05, na.rm=TRUE)
```

:question: How many genes are differentially expressed at an adjusted p-value of 0.05 in
each DESeq2 run? Do we find more differentially expressed genes before or after removing
the sex effect?

For the remainder of this workshop, we'll use only the results with the sex effect removed.

What are our top differentially expressed genes?

```
head(res[order(res$pvalue),])
```

According to DESeq, one of our top differentially expressed genes is ENSG00000170962.
Make a boxplot to verify this.

```
status <- c(rep("Norm", 7), rep("IPF", 8))
boxplot(as.numeric(log_counts[rownames(log_counts)=="ENSG00000170962",]) ~ status)
```

### Following up results

ENSG00000170962 isn't a very descriptive name. Look up this gene on 
[GeneCards](http://www.genecards.org/cgi-bin/carddisp.pl?gene=PDGFD&keywords=ENSG00000170962)
to find its common name.

The authors of the IPF paper created an [IPF Browser](http://52.32.252.126:3838/geneExplorer/) for viewing gene expression across
other IPF datasets. Look this gene up on the browser.

:question: Is this gene differentially expressed in other IPF datasets?

[Previous work](http://journals.plos.org/plosone/article?id=10.1371/journal.pone.0092111#) 
has implicated MUC5B and DSP as differentially expressed genes in IPF. 
Test these genes to see if they're replicated in our analysis.

```
# MUC5B
res[rownames(res) == "ENSG00000117983",]$pvalue 
```

```
# Try the next one on your own. Look up the ENSG identifier for
# DSP online, and then get that row from the DESeq2 results.
```

:question: Which of these two genes has greater differential expression in our cohort?

To view the original code used by Tracy Nance for this analysis, see
https://github.com/datapixie/ipf/blob/master/runDESeq/runDESeq.R.

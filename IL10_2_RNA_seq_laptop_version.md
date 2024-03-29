---
title: "IL10_2_RNA_seq"
author: "Blake Mitchell"
date: "2023-01-31"
output: github_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

###################################################################################################
#
#     Set up working environment and import data
#
###################################################################################################


```{r}
## Load libraries (install if necessary)
#library(tximport)
library(readr)
library(GenomicFeatures)
library(biomaRt)
library(DESeq2)
library(ggplot2)
library(gplots)
library(ggrepel)
library(rtracklayer)
library(scales)
library(dplyr)
```

```{r}
## Working directory
setwd("Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq")
base_dir = getwd()
```

```{r}
## Create directory for output results
dir.create(file.path(base_dir, 'DESeq_output'), showWarnings = TRUE)


## Project name
work_dir = "DESeq_output/"
proj_name = "IL10_2_RNA_seq"


## Import sample and condition file
samples = read.csv("Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\metadata.csv", header = TRUE, stringsAsFactors=FALSE)
row.names(samples) <- samples$sample
samples$Condition <- factor(samples$Condition)
samples        # Prints the sample / condition list



## Make a TxDb object from transcript annotations
## - available as a GFF3 or GTF file
## - can be downloaded from gencode for mouse / human, ensembl for other species

gtf="Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\GCF_000001635.27_GRCm39_genomic.gff.gz"   
txdb=makeTxDbFromGFF(gtf,
                     format="gff", 
                     organism="Mus Musculus", 
                     taxonomyId=10090)

k <- keys(txdb, keytype = "GENEID")
df <- AnnotationDbi:::select(txdb, keys = k, keytype = "GENEID", columns = "TXNAME")
tx2gene <- df[, 2:1]
head(tx2gene)


## Import Salmon quant files and create counts table
files <- list.files(path = "Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\Salmon transcript quantificaion", pattern = ".tabular", full.names = T)
all(file.exists(files))        # Verify names of files match names in samples.csv, should return True
names(files)=samples$sample
txi <- tximport(files, 
                type = "salmon", 
                tx2gene = tx2gene)
head(txi$counts)               # This is the counts table for all of our samples


## Now to import the data into a DESeq Data Set (dds)
## Verify that sample names and colnames are the same
identical(samples$sample,colnames(txi$counts))
all(rownames(samples) == colnames(txi$counts))

## Create a DEseqDataSet from txi count table
## DO NOT USE DESeqDataSetFromTximport for 3'TAG-RNAseq
## Before proceeding, remove samples from cts that failed QC 
cts <- round(txi$counts) # round raw counts
cts <- cts[,(-c(8,10,11,19,24,16,14,18) )]
samples <- samples[c("IEC_Cont_101","IEC_Cont_102","IEC_Cont_103","IEC_IL10KO_3", "IEC_IL10KO_4","IEC_IL10KO_6","IEC_IL10KO_14","IEC_IL10KOZn_19","IEC_IL10KOZn_31"
,"INT_IL10KO_1","INT_IL10KO_8","INT_IL10KO_20","INT_IL10KOZn_55","INT_IL10KOZn_60","INT_IL10KOZn_37", "INT_IL10KOZn_42"),]

## Now, split into IEC and INT 
IEC <- cts[,(1:9)]
IECsamples <- samples[1:9,]
IECsamples %>% 
  mutate(across(where(is.character), str_trim))

INT <- cts[,(10:16)]
INTsamples <- samples[10:16,]

dds_IEC <- DESeqDataSetFromMatrix(countData = IEC,
                              colData = IECsamples,
                              design = ~ Condition)
dds_INT <- DESeqDataSetFromMatrix(countData = INT,
                              colData = INTsamples,
                              design = ~ Condition)
```

```{r}
###################################################################################################
#
#    EXPLORATORY DATA ANALYSIS
#
###################################################################################################
library(tidyverse)
library("RColorBrewer")
library(pheatmap)
library(dendextend)

## Perform a rlog transformation on count data (essentially a puts on a log2 scale)
## This helps our data assume a normal distribution and is good to do before these analyses
rld_IEC <- rlog(dds_IEC, blind=TRUE)
rld_INT <- rlog(dds_INT, blind = TRUE)


## Setup annotation file to show the conditions on the figures
treat_ann_IEC <- IECsamples
treat_ann_IEC$sample <- NULL
treat_ann_IEC

treat_ann_INT <- INTsamples
treat_ann_INT$sample <- NULL
treat_ann_INT

## Setup annotation file to show the conditions on the figures
#samples$condition <- factor(samples$condition, levels = c('Mock', 'DSS'))

callback = function(hc, ...){
  
  dendextend::rotate(hc, row.names(samples)) 
  
}


## SAMPLE TO SAMPLE DISTANCE & CORRELATION HEATMAPS
## Sample correlation heatmap
corr_samps_IEC <- cor(as.matrix(assay(rld_IEC)))
corr_samps_INT <- cor(as.matrix(assay(rld_INT)))
# Computes pairwise correlations between samples based on gene expression
colors <- colorRampPalette( (brewer.pal(9, "Purples")) )(100)
## IEC
callback = function(hc, ...){
  
  dendextend::rotate(hc, row.names(IECsamples)) 
  
}
png(filename="DESeq_output/DESeq_sampleCorr_IEC.png", units = 'in', width = 10, height = 10, res = 250)
pheatmap(corr_samps_IEC,
         annotation = treat_ann_IEC, 
         col=colors,
         main="IEC sample correlation", 
         cellheight= 10, cellwidth=10, fontsize = 10 ,treeheight_row = 25, treeheight_col = 25,
         clustering_callback = callback,
         show_rownames = T, show_colnames = F)
dev.off()
## INT
callback = function(hc, ...){
  
  dendextend::rotate(hc, row.names(INTsamples)) 
  
}
png(filename="DESeq_output/DESeq_sampleCorr_INT.png", units = 'in', width = 10, height = 10, res = 250)
pheatmap(corr_samps_INT,
         annotation = treat_ann_INT, 
         col=colors,
         main="INT sample correlation", 
         cellheight= 10, cellwidth=10, fontsize = 10 ,treeheight_row = 25, treeheight_col = 25,
         clustering_callback = callback,
         show_rownames = T, show_colnames = F)
dev.off()

# IEC Sample distance heatmap
callback = function(hc, ...){
  
  dendextend::rotate(hc, row.names(IECsamples)) 
  
}
sampleDists <- dist(t(assay(rld_IEC)))            # Computes Euclidean distance between samples based on gene expression
sampleDistMatrix <- as.matrix(sampleDists)
colors <- colorRampPalette( rev(brewer.pal(9, "Purples")) )(100)

png(filename="DESeq_output/DESeq_sampleDist_IEC.png", units = 'in', width = 10, height = 10, res = 250)
pheatmap(sampleDistMatrix,
         clustering_distance_rows=sampleDists,
         clustering_distance_cols=sampleDists,
         annotation = treat_ann_IEC,
         col=colors,
         main="IEC sample distance",
         cellheight= 10, cellwidth=10, fontsize = 10,treeheight_row = 25, treeheight_col = 25,
         clustering_callback = callback,
         show_rownames = T, show_colnames = F) 
#dev.off()
# INT Sample distance heatmap
callback = function(hc, ...){
  
  dendextend::rotate(hc, row.names(INTsamples)) 
  
}
sampleDists <- dist(t(assay(rld_INT)))            # Computes Euclidean distance between samples based on gene expression
sampleDistMatrix <- as.matrix(sampleDists)
colors <- colorRampPalette( rev(brewer.pal(9, "Purples")) )(100)

png(filename="DESeq_output/DESeq_sampleDist_INT.png", units = 'in', width = 10, height = 10, res = 250)
pheatmap(sampleDistMatrix,
         clustering_distance_rows=sampleDists,
         clustering_distance_cols=sampleDists,
         annotation = treat_ann_INT,
         col=colors,
         main="INT sample distance",
         cellheight= 10, cellwidth=10, fontsize = 10,treeheight_row = 25, treeheight_col = 25,
         clustering_callback = callback,
         show_rownames = T, show_colnames = F) 
dev.off()

## Principal Component Analysis
## Separates samples based on variation between sample's gene expression
## Greater variation will affect separation to a greater degree

## IEC
data <- plotPCA(rld_IEC, intgroup=c("Condition"), returnData=TRUE)
percentVar <- round(100 * attr(data, "percentVar"))


png('DESeq_output/DESeq_PCA_IEC.png', units='in', width=7, height=5, res=250)
ggplot(data, aes(PC1, PC2, color=Condition)) +
  geom_point(size=4, alpha = 0.85) +
  geom_text_repel(aes(label=name)) +
  theme_bw() +
  xlab(paste0("PC1: ",percentVar[1],"% variance")) +
  ylab(paste0("PC2: ",percentVar[2],"% variance")) +
  ggtitle("PCA") + 
  theme(text = element_text(size = 20),
        panel.grid.major = element_blank(), 
        panel.grid.minor = element_blank(),
        plot.title = element_text(hjust = 0),
        legend.position = 'right',
        legend.text.align = 0,
        legend.key.size = unit(1.5, 'lines'),
        panel.background = element_rect(colour = "black", fill = "white"),
        axis.text.y=element_text(color="black", size=18),
        axis.text.x=element_text(color="black", size=18))

dev.off()

## INT
data <- plotPCA(rld_INT, intgroup=c("Condition"), returnData=TRUE)
percentVar <- round(100 * attr(data, "percentVar"))


png('DESeq_output/DESeq_PCA_INT.png', units='in', width=7, height=5, res=250)
ggplot(data, aes(PC1, PC2, color=Condition)) +
  geom_point(size=4, alpha = 0.85) +
  geom_text_repel(aes(label=name)) +
  theme_bw() +
  xlab(paste0("PC1: ",percentVar[1],"% variance")) +
  ylab(paste0("PC2: ",percentVar[2],"% variance")) +
  ggtitle("PCA") + 
  theme(text = element_text(size = 20),
        panel.grid.major = element_blank(), 
        panel.grid.minor = element_blank(),
        plot.title = element_text(hjust = 0),
        legend.position = 'right',
        legend.text.align = 0,
        legend.key.size = unit(1.5, 'lines'),
        panel.background = element_rect(colour = "black", fill = "white"),
        axis.text.y=element_text(color="black", size=18),
        axis.text.x=element_text(color="black", size=18))

dev.off()
```

###################################################################################################
#
#   Get table to convert names
#
###################################################################################################
```{r}
## Convert gene ID to gene name
# read GTF file into ensemblGenome object
#read.gtf(ens, "gencode.v25.annotation.gtf")
gtf <- rtracklayer::import.gff("Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\GCF_000001635.27_GRCm39_genomic.gff.gz")
gtf_df=as.data.frame(gtf)

gtf_df <- gtf_df[,c('Name', 'gene')]
gtf_df <- gtf_df[!duplicated(gtf_df),]
gtf_df$Name <- sub("\\.[0-9]*", "", gtf_df$Name)
names(gtf_df) <- c("GENEID", "geneName")
gtf_df <- gtf_df[!duplicated(gtf_df),]
```

###################################################################################################
#
#    Normalized counts
#
###################################################################################################
```{r}
## DESeq = fx to calculate DE
## Combines multiple steps from DESeq
dds_IEC <- DESeq(dds_IEC)
dds_INT <- DESeq(dds_INT)

## Write normalized counts to file
IEC.normalized.counts <- as.data.frame(counts(dds_IEC, normalized=TRUE ))

INT.normalized.counts <- as.data.frame(counts(dds_INT, normalized =TRUE))

# Write count table
write.csv(IEC.normalized.counts, file = 'DESeq_output/DESeq2_RNAseq_IEC_normalized_counts.csv', quote = FALSE, sep = "\t", row.names = TRUE, col.names = NA)
write.csv(INT.normalized.counts, file = 'DESeq_output/DESeq2_RNAseq_INT_normalized_counts.csv', quote = FALSE, sep = "\t", row.names = TRUE, col.names = NA)

```

#############################################################
#
#    Volcano plot 
#
#############################################################

```{r}
# create a function
MAPlot <- function(df, pval_cut = .05, padj_cut = 0.2) {
  log2_lim <- 10
  bm <- 50
  fc_cut <- 0.5
  
  df_plt <- as.data.frame(df) %>%
    mutate(threshold = ifelse(pvalue <= pval_cut & padj <= padj_cut , 1, 0)) %>%
    filter(threshold != "NA") %>% 
    mutate(threshold = as.factor(threshold)) %>%
    mutate(shape = ifelse(abs(log2FoldChange) > log2_lim, 17, 16)) %>%
    mutate(log2FoldChange = ifelse(abs(log2FoldChange) > log2_lim, log2_lim * .98 * sign(log2FoldChange), log2FoldChange)) 
  
  
  upregulated <- df_plt %>% filter(pvalue <= pval_cut & log2FoldChange > fc_cut & padj <= padj_cut & baseMean > bm)
  downregulated <- df_plt %>% filter(pvalue <= pval_cut & log2FoldChange < -(fc_cut) & padj <= padj_cut & baseMean > bm)
  
  ##Construct the plot object
  g = ggplot(df_plt, aes(x= baseMean, y=log2FoldChange, color=threshold)) +
    geom_point(size=1, shape = df_plt$shape) +
    scale_colour_manual(values = c("grey90", "grey40")) +
    geom_point(data = upregulated, size=1, color = '#e6550d') +
    geom_point(data = downregulated, size=1, color = '#1d91c0') +
    geom_vline(xintercept = 50, linetype = 'dashed', size =0.25) +
    geom_hline(yintercept = -(fc_cut), linetype = 'dashed', size =0.25) +
    geom_hline(yintercept = fc_cut, linetype ='dashed', size = 0.25) +
    annotate('text', label = nrow(downregulated), x = max(df_plt$baseMean)*0.1, y = -log2_lim*0.8, vjust = 0, hjust = 0, size = 6, color = "#1d91c0") +
    annotate('text', label = nrow(upregulated), x = max(df_plt$baseMean)*0.1 , y = log2_lim*0.8, vjust = 0, hjust = 0, size = 6, color = "#e6550d") +
    ggtitle(paste0(condition1, ' vs. ', condition2)) +
    theme(plot.title = element_text(hjust = 0.5)) +
    scale_x_log10('mean of normalized count', 
                  breaks = trans_breaks("log10", function(x) 10^x),
                  labels = trans_format("log10", math_format(10^.x)), expand = c(0,0)) +
    scale_y_continuous("log2(FC)",
                       limit = c(-(log2_lim), log2_lim),
                       expand = c(0, 0)) +
    theme(plot.title = element_text(color='black',size = 20), 
          panel.grid.major = element_blank(), 
          panel.grid.minor = element_blank(),
          panel.background = element_rect(colour = "black", fill = "white"), 
          text = element_text(size = 20),
          legend.position = "none") +
    annotation_logticks(sides = "b")  
  
  return(g)
}
```

###################################################################################################
#
#    Differential expression analysis: IEC WT vs IEC IL10KO
#
###################################################################################################

```{r}
dds <- DESeqDataSetFromMatrix(countData = IEC,
                       colData = IECsamples,
                       design = ~ Condition)
dds <- DESeq(dds)

## Differential analysis
res <- results(dds, contrast = c("Condition", "IL10KO", "Control"))
condition1 <- "IEC_IL10KO"
condition2 <- "IEC_Control"

## Add gene name to res file
rownames(res)<-sub("\\.[0-9]*", "", rownames(res))
idx <- match( rownames(res), gtf_df$GENEID )
res$geneName <- gtf_df$geneName[ idx ]


## get differentially expressed genes
## threshold: baseMean 100, pvalue 0.05, adjusted P value 0.05
new.res <- res

nrow(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 & new.res$log2FoldChange > 0),]) #2
nrow(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 &  new.res$log2FoldChange < 0),]) #18

UP <- as.data.frame(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 &  new.res$log2FoldChange > 0),])
UP$gene <- rownames(UP)
DOWN <- as.data.frame(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 & new.res$log2FoldChange < 0),])
DOWN$gene <- rownames(DOWN)


## output file
write.csv(new.res, paste0('DESeq_output/DESeq2_RNAseq_', condition1, '_vs_', condition2, '.csv'), quote=F, row.names = F)
write.csv(UP, paste0('DESeq_output/UP_RNAseq_p0.05padj0.2', condition1, '_vs_', condition2, '.csv'), quote = FALSE, sep = "\t", row.names = F)
write.csv(DOWN, paste0('DESeq_output/DOWN_RNAseq_p0.05padj0.2', condition1, '_vs_', condition2, '.csv'), quote = FALSE, sep = "\t", row.names = F)


## Volcano plot
png(paste0('DESeq_output/MAPlot_FC_0.5_', condition1, '_vs_', condition2, '.png'), units = 'in', width = 6, height = 6, res = 250)
g <- MAPlot(new.res) + theme(plot.margin=unit(c(1,1,1.5,1.2),"cm"))
g
dev.off()

```

###################################################################################################
#
#    Differential expression analysis: IEC IL10KO vs IEC IL10KOZn
#
###################################################################################################

```{r}
dds <- DESeqDataSetFromMatrix(countData = IEC,
                       colData = IECsamples,
                       design = ~ Condition)
dds <- DESeq(dds)

## Differential analysis
res <- results(dds, contrast = c("Condition", "IL10KOZn", "IL10KO"))
condition1 <- "IEC_IL10KOZn"
condition2 <- "IEC_IL10KO"

## Add gene name to res file
rownames(res)<-sub("\\.[0-9]*", "", rownames(res))
idx <- match( rownames(res), gtf_df$GENEID )
res$geneName <- gtf_df$geneName[ idx ]


## get differentially expressed genes
## threshold: baseMean 100, pvalue 0.05, adjusted P value 0.05
new.res <- res

nrow(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 & new.res$log2FoldChange > 0),]) #2
nrow(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 &  new.res$log2FoldChange < 0),]) #18

UP <- as.data.frame(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 &  new.res$log2FoldChange > 0),])
UP$gene <- rownames(UP)
DOWN <- as.data.frame(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 & new.res$log2FoldChange < 0),])
DOWN$gene <- rownames(DOWN)


## output file
write.csv(new.res, paste0('DESeq_output/DESeq2_RNAseq_', condition1, '_vs_', condition2, '.csv'), quote=F, row.names = F)
write.csv(UP, paste0('DESeq_output/UP_RNAseq_p0.05padj0.2', condition1, '_vs_', condition2, '.csv'), quote = FALSE, sep = "\t", row.names = F)
write.csv(DOWN, paste0('DESeq_output/DOWN_RNAseq_p0.05padj0.2', condition1, '_vs_', condition2, '.csv'), quote = FALSE, sep = "\t", row.names = F)


## Volcano plot
png(paste0('DESeq_output/MAPlot_FC_0.05_', condition1, '_vs_', condition2, '.png'), units = 'in', width = 6, height = 6, res = 250)
g <- MAPlot(new.res) + theme(plot.margin=unit(c(1,1,1.5,1.2),"cm"))
g
dev.off()

```

###################################################################################################
#
#    Differential expression analysis: IEC WT vs IEC IL10KOZn
#
###################################################################################################

```{r}
## Differential analysis
res <- results(dds, contrast = c("Condition", "IL10KOZn", "Control"))
condition1 <- "IEC_IL10KOZn"
condition2 <- "IEC_Control"


## get differentially expressed genes
## threshold: baseMean 100, pvalue 0.05, adjusted P value 0.05
new.res <- res

nrow(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 & new.res$log2FoldChange > 0),]) #2
nrow(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 &  new.res$log2FoldChange < 0),]) #18

UP <- as.data.frame(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 &  new.res$log2FoldChange > 0),])
UP$gene <- rownames(UP)
DOWN <- as.data.frame(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 & new.res$log2FoldChange < 0),])
DOWN$gene <- rownames(DOWN)


## output file
write.csv(new.res, paste0('DESeq_output/DESeq2_RNAseq_', condition1, '_vs_', condition2, '.csv'), quote=F, row.names = F)
write.csv(UP, paste0('DESeq_output/UP_RNAseq_p0.05padj0.2', condition1, '_vs_', condition2, '.csv'), quote = FALSE, sep = "\t", row.names = F)
write.csv(DOWN, paste0('DESeq_output/DOWN_RNAseq_p0.05padj0.2', condition1, '_vs_', condition2, '.csv'), quote = FALSE, sep = "\t", row.names = F)


## Volcano plot
png(paste0('DESeq_output/MAPlot_FC_0.05_', condition1, '_vs_', condition2, '.png'), units = 'in', width = 6, height = 6, res = 250)
g <- MAPlot(new.res) + theme(plot.margin=unit(c(1,1,1.5,1.2),"cm"))
g
dev.off()
```

###################################################################################################
#
#    Differential expression analysis: INT IL10KO vs INT IL10KOZn
#
###################################################################################################

```{r}
dds <- DESeqDataSetFromMatrix(countData = INT,
                       colData = INTsamples,
                       design = ~ group)
dds <- DESeq(dds)

## Differential analysis
res <- results(dds, contrast = c("group", "Zn", "Control"))
condition1 <- "INT_IL10KOZn"
condition2 <- "INT_IL10KO"


## get differentially expressed genes
## threshold: baseMean 100, pvalue 0.05, adjusted P value 0.05
new.res <- res

nrow(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 & new.res$log2FoldChange > 0),]) #2
nrow(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 &  new.res$log2FoldChange < 0),]) #18

UP <- as.data.frame(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 &  new.res$log2FoldChange > 0),])
UP$gene <- rownames(UP)
DOWN <- as.data.frame(new.res[which(new.res$pvalue < 0.05 & new.res$padj < 0.2 & new.res$log2FoldChange < 0),])
DOWN$gene <- rownames(DOWN)


## output file
write.csv(new.res, paste0('DESeq_output/DESeq2_RNAseq_', condition1, '_vs_', condition2, '.csv'), quote=F, row.names = F)
write.csv(UP, paste0('DESeq_output/UP_RNAseq_p0.05padj0.2', condition1, '_vs_', condition2, '.csv'), quote = FALSE, sep = "\t", row.names = F)
write.csv(DOWN, paste0('DESeq_output/DOWN_RNAseq_p0.05padj0.2', condition1, '_vs_', condition2, '.csv'), quote = FALSE, sep = "\t", row.names = F)


## Volcano plot
png(paste0('DESeq_output/MAPlot_FC_0.05', condition1, '_vs_', condition2, '.png'), units = 'in', width = 6, height = 6, res = 250)
g <- MAPlot(new.res) + theme(plot.margin=unit(c(1,1,1.5,1.2),"cm"))
g
dev.off()




```


```{r}
## Graph GO results (done in Toppgene web app)
UP <- read.csv("C:\\Users\\sbm228\\OneDrive - Cornell University\\Documents\\IL10_2_RNA_seq\\Toppgene IL10KO vs IL10 KOZn UP.csv")

DOWN <- read.csv("C:\\Users\\sbm228\\OneDrive - Cornell University\\Documents\\IL10_2_RNA_seq\\Toppgene IL10KO vs IL10 KOZn DOWN.csv")

library(ggplot2)
library(stringr)
theme_set(theme_bw())
pal = "Set1"
scale_colour_discrete <-  function(palname=pal, ...){
  scale_colour_brewer(palette=palname, ...)
}
scale_fill_discrete <-  function(palname=pal, ...){
  scale_fill_brewer(palette=palname, ...)
}

UP$neg_log_adj_pval <- -log(UP$FDRB.H)
UP$Name <- paste0(UP$Source," (",UP$Name,")")

DOWN$neg_log_adj_pval <- -log(DOWN$FDR.B.H)
DOWN$Name <- paste0(DOWN$Source, " (",DOWN$Name,")")


U <- ggplot(UP, aes(x = neg_log_adj_pval, y = reorder(Name, neg_log_adj_pval))) +
  geom_bar(stat = 'identity', color="black", fill="#e6550d") +
  theme_bw() +
  theme(panel.grid.minor = element_blank(),
        panel.border = element_blank(),
        axis.line = element_line(color = "black"),
        text = element_text(size = 10),
        axis.ticks = element_blank(),
        axis.title = element_text(size = 10, face = "bold"),
        axis.text.x = element_text(size = 10, hjust = 1, family = "sans", face = "bold"),
        axis.text.y = element_text(size = 12, hjust = 1, family = "sans", face = "bold"),
        plot.margin = unit(c(.5,.5,.5,.5), "cm"), 
        legend.position = "none"
        )+
  labs(title = "Upregulated", x = "-log(adjusted p-value)", y = "")+
  scale_y_discrete(labels = function(x) str_wrap(x, width = 50)) + 
  geom_vline(xintercept = 3,linetype=2)

ggsave(path = "C:\\Users\\sbm228\\OneDrive - Cornell University\\Documents\\IL10_2_RNA_seq", filename ="GO results IL10KO vs IL10KOZn Upregulated.tiff", height=6, width=9, dpi = 600, device="tiff")

D <- ggplot(DOWN, aes(x = neg_log_adj_pval, y = reorder(Name, neg_log_adj_pval))) +
  geom_bar(stat = 'identity', color="black", fill="#1d91c0") +
  theme_bw() +
  theme(panel.grid.minor = element_blank(),
        panel.border = element_blank(),
        axis.line = element_line(color = "black"),
        text = element_text(size = 10),
        axis.ticks = element_blank(),
        axis.title = element_text(size = 10, face = "bold"),
        axis.text.x = element_text(size = 10, hjust = 1, family = "sans", face = "bold"),
        axis.text.y = element_text(size = 12, hjust = 1, family = "sans", face = "bold"),
        plot.margin = unit(c(.5,.5,.5,.5), "cm"), 
        legend.position = "none"
        )+
  labs(title = "Downregulated", x = "-log(adjusted p-value)", y = "")+
  scale_y_discrete(labels = function(x) str_wrap(x, width = 50)) + 
  geom_vline(xintercept = 3,linetype=2)

ggsave(path = "C:\\Users\\sbm228\\OneDrive - Cornell University\\Documents\\IL10_2_RNA_seq", filename ="GO results IL10KO vs IL10KOZn Downregulated.tiff", height=6, width=9, dpi = 600, device="tiff")
```

```{r}
## Bar graphs 
# Read in differentiall expressed genes: DOWN in IL10 KO vs WT and UP in IL10 KO Zn vs IL10 KO. Also, UP in IL10 KO vs WT and DOWN in IL10 KO Zn vs IL10 KO
UPZnvsKO <- read.csv("Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\DESeq_output\\Intestine Epithelial cells\\IL10KOZn vs IL10KO\\UP_RNAseq_p0.05padj0.2IEC_IL10KOZn_vs_IEC_IL10KO.csv")
UPZnvsKO <- UPZnvsKO[,8]
DOWNZnvsKO <- read.csv("Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\DESeq_output\\Intestine Epithelial cells\\IL10KOZn vs IL10KO\\DOWN_RNAseq_p0.05padj0.2IEC_IL10KOZn_vs_IEC_IL10KO.csv")
DOWNZnvsKO <- DOWNZnvsKO[,8]
UPKOvsWT <- read.csv("Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\DESeq_output\\Intestine Epithelial cells\\IL10KO vs WT\\UP_RNAseq_p0.05padj0.2IEC_IL10KO_vs_IEC_Control.csv")
UPKOvsWT <- UPKOvsWT[,8]
DOWNKOvsWT <- read.csv("Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\DESeq_output\\Intestine Epithelial cells\\IL10KO vs WT\\DOWN_RNAseq_p0.05padj0.2IEC_IL10KO_vs_IEC_Control.csv")
DOWNKOvsWT <- DOWNKOvsWT[,8]
UPZnvsWT <- read.csv("Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\DESeq_output\\Intestine Epithelial cells\\IL10KOZn vs WT\\UP_RNAseq_p0.05padj0.2IEC_IL10KOZn_vs_IEC_Control.csv")
UPZnvsWT <- UPZnvsWT[,7]
DOWNZnvsWT <- read.csv("Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\DESeq_output\\Intestine Epithelial cells\\IL10KOZn vs WT\\DOWN_RNAseq_p0.05padj0.2IEC_IL10KOZn_vs_IEC_Control.csv")
DOWNZnvsWT <- DOWNZnvsWT[,7]

library(VennDetail)
## Genes that are DOWN in IL10 KO vs WT and UP in IL10 KO Zn vs IL10 KO
ven <- venndetail(list(down_IL10KO = DOWNKOvsWT, up_IL10KO_Zn = UPZnvsKO))
ven_list <- getSet(ven, subset = "Shared")
names(ven_list) <- c("Subset", "Gene")
write.csv(ven_list, file = "Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\DESeq_output\\Intestine Epithelial cells\\IL10KOZn vs WT\\DOWN in IL10 KO & UP in IL10KO Zn.csv")
png(paste0('DESeq_output/DOWN in IL10KO vs WT AND UP in IL10 KO Zn vs IL10KO.png'), units = 'in', width = 10, height = 10, res = 500)
g <- plot(ven) + theme(plot.margin=unit(c(1,100,1.5,1.2),"cm"))
g
dev.off()

## Genes that are UP in IL10 KO vs WT and DOWN in IL10 KO Zn vs IL10 KO
ven <- venndetail(list("Up in IL10KO vs WT" = UPKOvsWT, "Down in IL10KO_Zn vs IL10KO" = DOWNZnvsKO))
ven_list <- getSet(ven, subset = "Shared")
names(ven_list) <- c("Subset", "Gene")
write.csv(ven_list, file = "Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\DESeq_output\\Intestine Epithelial cells\\IL10KOZn vs WT\\UP in IL10 KO & DOWN in IL10KO Zn.csv")
png(paste0('DESeq_output/UP in IL10KO vs WT AND DOWN in IL10 KO Zn vs IL10KO.png'), units = 'in', width = 15, height = 15, res = 500)
g <- plot(ven) + theme(plot.margin=unit(c(1,100,1.5,1.2),"cm"))
g
dev.off()

## Genes that are UP in IL10 KO vs WT and DOWN in IL10 KO Zn vs WT
ven <- venndetail(list("Up in IL10KO vs WT" = UPKOvsWT, "Down in IL10KO_Zn vs WT" = DOWNZnvsWT))
ven_list <- getSet(ven, subset = "Shared")
names(ven_list) <- c("Subset", "Gene")
write.csv(ven_list, file = "Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\DESeq_output\\Intestine Epithelial cells\\IL10KOZn vs WT\\UP in IL10 KO vs WT & DOWN in IL10KO Zn vs WT.csv")
png(paste0('DESeq_output/UP in IL10KO vs WT AND DOWN in IL10 KO Zn vs WT.png'), units = 'in', width = 15, height = 15, res = 500)
g <- plot(ven) + theme(plot.margin=unit(c(1,100,1.5,1.2),"cm"))
g
dev.off()

## Genes that are DOWN in IL10 KO vs WT and UP in IL10 KO Zn vs WT
ven <- venndetail(list("DOWN in IL10KO vs WT" = DOWNKOvsWT, "UP in IL10KO_Zn vs WT" = UPZnvsWT))
ven_list <- getSet(ven, subset = "Shared")
names(ven_list) <- c("Subset", "Gene")
write.csv(ven_list, file = "Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\DESeq_output\\Intestine Epithelial cells\\IL10KOZn vs WT\\DOWN in IL10 KO vs WT & UP in IL10KO Zn vs WT.csv")
png(paste0('DESeq_output/DOWN in IL10KO vs WT AND UP in IL10 KO Zn vs WT.png'), units = 'in', width = 15, height = 15, res = 500)
g <- plot(ven) + theme(plot.margin=unit(c(1,100,1.5,1.2),"cm"))
g
dev.off()

```

```{r}
## Read in list of reciprocally regulated genes
genes <- read.csv("Z:\\Aydemir Lab Members\\Blake Mitchell\\Experiments\\IL10 2 RNA seq\\DESeq_output\\Intestine Epithelial cells\\Opposite genes.csv")
rownames(genes) <- genes$Gene
IEC.normalized.counts$Gene <- rownames(IEC.normalized.counts)
genes <- left_join(genes, IEC.normalized.counts, by = 'Gene')
rownames(genes) <- genes$Gene
genes <- genes[,-1]
genes <- as.data.frame(t(genes))
genes <- mutate_all(genes, function(x) as.numeric(as.character(x)))
genes$Sample <- rownames(genes)
genes[,"condition"] <- c("Ctrl", "Ctrl","Ctrl","IL10 KO","IL10 KO","IL10 KO","IL10 KO","IL10 KO + Zn","IL10 KO + Zn")


lgenes <- genes %>%
  pivot_longer(
    cols = -(74:75), # by using a `-`, we can exclude the first two columns
    values_to = c("normalized_counts"),
    names_to = c("Gene"),
    #names_sep  = "_"
  ) %>%
  group_by(Gene, condition) %>%
  summarise(n = n(),
            mean = mean(normalized_counts),
            sd = sd(normalized_counts))
################################################################################
## Quarantine for plot loop that doesnt work
goi <- c(colnames(genes))

var_list <- goi
plot_list <- list()
for (i in var_list) {
  p = ggplot(lgenes, aes(x = condition, y = mean)) +
  geom_col() +
  geom_linerange(aes(
    x = condition,
    ymin = mean - sd,
    ymax = mean + sd
  ))
  plot_list[[i]] = p}
for (i in var_list) {
  file_name = paste(i, ".tiff", sep="")
  tiff(file_name)
  print(plot_list[[i]])
  dev.off()
}
################################################################################
lgenes %>%
  filter(Gene %in% goi) %>%
  ggplot(aes(x = condition, y = mean)) +
  geom_col() +
  geom_linerange(aes(
    x = condition,
    ymin = mean - sd,
    ymax = mean + sd
  )) +
  facet_wrap( ~ Gene, scales = 'free')


```


Walkthrough with all data analyses that comes with the manuscript
Comparing published FPKM/RPKM values and reprocessed FPKM values.
========================================================

The first part of this document defines some helper functions and then shows in detail how data were downloaded and combined. However, most of the material related to downloading and combining is commented out - it is just left for reference. 

Prepare by loading packages etc.
```{r prepare}
library(pheatmap)
library(reshape)
library(gplots)
library(ops)
library(calibrate)
library(biomaRt)
library(sva)
```
Function definitions
--------------------
Define functions for (a) computing and plotting p-values for correlations between PCA scores and various sample features and (b) quantifying and plotting the importance of various sample features using ANOVA.

**Correlations between PCs and experimental factors.**
```{r}
 print_PCA_corrs <- function(data,sampleinfo,caption="PCA correlations",include.quant=F){
pca <- prcomp(t(data[,]))
rot <- pca$r
x <- pca$x

if(include.quant){
  pc1 <- rep(1,8)
names(pc1) <- c("Raw reads","Mapped reads", "Tissue", "Library prep", "Study", "Read type", "Read length","Quantification method")
pc2 <- rep(1,8)
}
else{pc1 <- rep(1,7)
names(pc1) <- c("Raw reads","Mapped reads", "Tissue", "Library prep", "Study", "Read type", "Read length")
pc2 <- rep(1,7)
}
names(pc2) <- names(pc1)

# Test correlations between number of seq'd reads and PCs 1-4 from prcomp
pval.nraw.pc1 <- cor.test(x[,1], sampleinfo$NumberRaw,method="spearman")$p.value
pval.nraw.pc2 <- cor.test(x[,2], sampleinfo$NumberRaw,method="spearman")$p.value
pval.nraw.pc3 <- cor.test(x[,3], sampleinfo$NumberRaw,method="spearman")$p.value

cat(sprintf("Number_of_rawreads~PCAs: PCA1=\"%f\"PCA2=\"%f\"PCA3=\"%f\n", pval.nraw.pc1,pval.nraw.pc2,pval.nraw.pc3))

pc1[1] <- pval.nraw.pc1
pc2[1] <- pval.nraw.pc2

pval.nmapped.pc1 <- cor.test(x[,1], sampleinfo$Numbermapped,method="spearman")$p.value
pval.nmapped.pc2 <- cor.test(x[,2], sampleinfo$Numbermapped,method="spearman")$p.value
pval.nmapped.pc3 <- cor.test(x[,3], sampleinfo$Numbermapped,method="spearman")$p.value

cat(sprintf("Number_of_mappedreads~PCAs: PCA1=\"%f\"PCA2=\"%f\"PCA3=\"%f\n", pval.nmapped.pc1,pval.nmapped.pc2,pval.nmapped.pc3))

pc1[2] <- pval.nmapped.pc1
pc2[2] <- pval.nmapped.pc2

# For tissue, use kruskal.test which handles ordinal variables 
pval.tissue.pc1<-kruskal.test(x[,1], sampleinfo$Tissue)$p.value
pval.tissue.pc2<-kruskal.test(x[,2], sampleinfo$Tissue)$p.value
pval.tissue.pc3<-kruskal.test(x[,3], sampleinfo$Tissue)$p.value

cat(sprintf("Tissues~PCAs: PCA1=\"%f\"PCA2=\"%f\"PCA3=\"%f\"\n", pval.tissue.pc1,pval.tissue.pc2,pval.tissue.pc3))

pc1[3] <- pval.tissue.pc1
pc2[3] <- pval.tissue.pc2

# Library prep 
pval.prep.pc1<-kruskal.test(x[,1], sampleinfo$Preparation)$p.value
pval.prep.pc2<-kruskal.test(x[,2], sampleinfo$Preparation)$p.value
pval.prep.pc3<-kruskal.test(x[,3], sampleinfo$Preparation)$p.value

cat(sprintf("LibPrep~PCAs: PCA1=\"%f\"PCA2=\"%f\"PCA3=\"%f\"\n", pval.prep.pc1,pval.prep.pc2,pval.prep.pc3))

pc1[4] <- pval.prep.pc1
pc2[4] <- pval.prep.pc2

# Study  
pval.study.pc1<-kruskal.test(x[,1], sampleinfo$Study)$p.value
pval.study.pc2<-kruskal.test(x[,2], sampleinfo$Study)$p.value
pval.study.pc3<-kruskal.test(x[,3], sampleinfo$Study)$p.value

cat(sprintf("Study~PCAs: PCA1=\"%f\"PCA2=\"%f\"PCA3=\"%f\"\n", pval.study.pc1,pval.study.pc2,pval.study.pc3))

pc1[5] <- pval.study.pc1
pc2[5] <- pval.study.pc2

# Layout
pval.layout.pc1<-kruskal.test(x[,1], sampleinfo$Readtype)$p.value
pval.layout.pc2<-kruskal.test(x[,2], sampleinfo$Readtype)$p.value
pval.layout.pc3<-kruskal.test(x[,3], sampleinfo$Readtype)$p.value

cat(sprintf("Study~PCAs: PCA1=\"%f\"PCA2=\"%f\"PCA3=\"%f\"\n", pval.layout.pc1,pval.layout.pc2,pval.layout.pc3))

pc1[6] <- pval.layout.pc1
pc2[6] <- pval.layout.pc2

# Read length
pval.readlength.pc1<-cor.test(x[,1], sampleinfo$readlength)$p.value
pval.readlength.pc2<-cor.test(x[,2], sampleinfo$readlength)$p.value
pval.readlength.pc3<-cor.test(x[,3], sampleinfo$readlength)$p.value

cat(sprintf("ReadType~PCAs: PCA1=\"%f\"PCA2=\"%f\"PCA3=\"%f\"\n", pval.layout.pc1,pval.layout.pc2,pval.layout.pc3))

library("ggplot2")

pc1[7] <- pval.readlength.pc1
pc2[7] <- pval.readlength.pc2

# Quantification
if(include.quant){
pval.quant.pc1<-kruskal.test(x[,1], as.factor(sampleinfo$quantification))$p.value
pval.quant.pc2<-kruskal.test(x[,2], as.factor(sampleinfo$quantification))$p.value
pval.quant.pc3<-kruskal.test(x[,3], as.factor(sampleinfo$quantification))$p.value

cat(sprintf("QuantificationMethod~PCAs: PCA1=\"%f\"PCA2=\"%f\"PCA3=\"%f\"\n", pval.layout.pc1,pval.layout.pc2,pval.layout.pc3))

pc1[8] <- pval.quant.pc1
pc2[8] <- pval.quant.pc2
}
par(mfrow=c(2,1))
barplot(-log(pc1),las=2,main=paste(caption, "PC1"),ylab="-log(p)")
barplot(-log(pc2),las=2,main=paste(caption, "PC2"),ylab="-log(p)")
}
```

** ANOVA for estimating the influence of different factors.**
```{r}
do_anova <- function(data, sampleinfo, caption="ANOVA", include.quant=F){
m <- melt(data)
colnames(m) <- c("sample_ID","RPKM")
if (include.quant){
  meta <- sampleinfo[,c("Study","Tissue","Preparation","NumberRaw","Numbermapped","Readtype","readlength","quantification")]}
else{
  meta <- sampleinfo[,c("Study","Tissue","Preparation","NumberRaw","Numbermapped","Readtype","readlength")]}
rownames(meta) <- colnames(data)
tissue <- rep(meta$Tissue, each=nrow(data))
study <- rep(meta$Study, each=nrow(data))
prep <- rep(meta$Preparation, each=nrow(data))
layout <- rep(meta$Readtype, each=nrow(data))
raw <- rep(meta$NumberRaw, each=nrow(data))
mapped <- rep(meta$Numbermapped, each=nrow(data))
readlen <- rep(meta$readlength, each=nrow(data))
if(include.quant){
quant <- rep(meta$quantification, each=nrow(data))
matrix <- data.frame(m, tissue=tissue, study=study, prep=prep, layout=layout, readlen=readlen, nraw=raw,nmapped=mapped, quant=quant)
fit <- lm(RPKM ~ layout + readlen + prep + nraw + quant + study + tissue, data=matrix)
}
else{
matrix <- data.frame(m, tissue=tissue, study=study, prep=prep, layout=layout, readlen=readlen, nraw=raw,nmapped=mapped)
fit <- lm(RPKM ~ layout + readlen + prep + nraw + study + tissue, data=matrix)
}

a <- anova(fit)
nfac <- length(a[,1])-1
maxval = 100
barplot(100*a$"Sum Sq"[1:nfac]/sum(a$"Sum Sq"[1:nfac]),names.arg=rownames(a[1:nfac,]),main=caption,ylim=c(0,maxval))
}
```

**Function for plotting the highest gene loadings in PCA.**
```{r}
plot.highest.loadings <- function(p, fpkm.table, caption="Highest loading"){
      par(mfrow=c(3,1))
      for(i in 1:3){
      load <- p$rotation[,i][order(p$rotation[,i])]
      extreme <- c(tail(load), head(load))
      extreme.ensg <- names(extreme)
      ensembl = useMart("ensembl", dataset = "hsapiens_gene_ensembl") #select the ensembl database
      extreme.symbols <- getBM(attributes=c("ensembl_gene_id", "hgnc_symbol"), 
                           filters = "ensembl_gene_id",
                           values=extreme.ensg,
                           mart=ensembl)
      q <- extreme.symbols[,2]
      names(q) <- extreme.symbols[,1]
      fpkm <- cbind(q[extreme.ensg],fpkm.table[extreme.ensg,])
      names(fpkm)[names(fpkm) == 'q[extreme.ensg]'] <- 'Gene Symbol'
      barplot(extreme, names.arg=q[extreme.ensg],las=2,main=paste0(caption, ", PC", i))
      print(fpkm)
     }
}
```

Downloading public data sets
----------------------------
Data from four different public sources are downloaded and brain, heart and kidney samples are extracted. Please uncomment the next blocks to download; we have commented this code out because the downloading of the GTEx data in particular takes a fair amount of time.

Start with Human Protein Atlas (HPA):

```{r:HPA download}
#temp <- tempfile()
#download.file(url="http://www.proteinatlas.org/download/rna.csv.zip",destfile=temp)
#hpa <- read.csv(unz(temp, "rna.csv"))
#unlink(temp)

#hpa.heart <- hpa[hpa$Sample=="heart muscle", c("Gene", "Value")]
#hpa.brain <- hpa[hpa$Sample=="cerebral cortex", c("Gene", "Value")]
#hpa.kidney <- hpa[hpa$Sample=="kidney", c("Gene", "Value")]

#hpa.fpkms <- merge(hpa.heart, hpa.brain, by="Gene")
#hpa.fpkms <- merge(hpa.fpkms, hpa.kidney, by="Gene")
#colnames(hpa.fpkms) <- c("ENSG_ID", "HPA_heart", "HPA_brain", "HPA_kidney")
```

Check if the identifiers are unique and write table to file.

```{r:HPA check identifiers}
#length(hpa.fpkms[,1])
#length(unique(hpa.fpkms[,1]))

#write.table(hpa.fpkms,file="hpa_fpkms.txt",quote=F,sep="\t")
```

The next dataset, "AltIso", is from the article "Alternative isoform regulation in human tissue transcriptomes.." by Wang et al.

```{r:AltIso download}
#temp <- tempfile()
#download.file(url="http://genes.mit.edu/burgelab/Supplementary/wang_sandberg08/hg18.ensGene.CEs.rpkm.txt",destfile=temp)
#altiso <- read.delim(temp, sep="\t")
#unlink(temp)
```

There is no kidney sample here, so just use heart and brain.

```{r}
#altiso.fpkms <- altiso[,c("X.Gene","heart","brain")]
#colnames(altiso.fpkms) <- c("ENSG_ID", "AltIso_heart", "AltIso_brain")
```

Check uniqueness of IDs.

```{r:AltIso check identifiers}
#length(altiso.fpkms[,1])
#length(unique(altiso.fpkms[,1]))

#write.table(altiso.fpkms,file="altiso_fpkms.txt",quote=F,sep="\t")
```

The next dataset is derived from "GTEx": Genotype-Tissue Expression.

This is a big download: 337.8 Mb (as of 2014-02-04)
We also add some code to randomly select one sample from each tissue type; there are many biological replicates in this data set.

```{r:GTEx download}
#temp <- tempfile()
#download.file(url="http://www.gtexportal.org/home/rest/file/download?portalFileId=175729&forDownload=true",destfile=temp)
#header_lines <- readLines(temp, n=2)
#gtex <- read.delim(temp, skip=2, sep="\t")
#unlink(temp)

#write.table(gtex, file="gtex_all.txt",   quote=F, sep="\t")

#download.file(url="http://www.gtexportal.org/home/rest/file/download?portalFileId=175707&forDownload=true",destfile="GTEx_description.txt")

#metadata <- read.delim("GTEx_description.txt", sep="\t")
```

The metadata table seems to contain entries that are not in the RPKM table.

```{r}
#samp.id <- gsub('-','.',metadata$SAMPID)
#eligible.samples <- which(samp.id %in% colnames(gtex))
#metadata <- metadata[eligible.samples,]
```

Select random heart, kidney and brain samples.

```{r}
#random.heart <- sample(which(metadata$SMTS=="Heart"), size=1)
#random.heart.samplename <- gsub('-','.',metadata[random.heart, "SAMPID"])
#gtex.heart.fpkm <- as.numeric(gtex[,random.heart.samplename])

#random.brain <- sample(which(metadata$SMTS=="Brain"), size=1)
#random.brain.samplename <- gsub('-','.',metadata[random.brain, "SAMPID"])
#gtex.brain.fpkm <- as.numeric(gtex[,random.brain.samplename])

#random.kidney <- sample(which(metadata$SMTS=="Kidney"), size=1)
#random.kidney.samplename <- gsub('-','.',metadata[random.kidney, "SAMPID"])
#gtex.kidney.fpkm <- as.numeric(gtex[,random.kidney.samplename])
```

Get gene IDs on same format as the other data sets by removing the part after the dot; check ID uniqueness and write to file.

```{r}
#gtex.names <- gtex[,"Name"]
#temp_list <- strsplit(as.character(gtex.names), split="\\.")
#gtex.names.nodot <- unlist(temp_list)[2*(1:length(gtex.names))-1]

#gtex.fpkms <- data.frame(ENSG_ID=gtex.names.nodot, GTEx_heart=gtex.heart.fpkm, GTEx_brain=gtex.brain.fpkm,GTEx_kidney=gtex.kidney.fpkm)

#length(gtex.fpkms[,1])
#length(unique(gtex.fpkms[,1]))

#write.table(gtex.fpkms,file="gtex_fpkms.txt",quote=F,sep="\t")
```

RNA-seq Atlas data.

```{r:Atlas download}
#temp <- tempfile()
#download.file(url="http://medicalgenomics.org/rna_seq_atlas/download?download_revision1=1",destfile=temp)
#atlas <- read.delim(temp, sep="\t")
#unlink(temp)

#atlas.fpkms <- atlas[,c("ensembl_gene_id","heart","hypothalamus","kidney")]
#colnames(atlas.fpkms) <- c("ENSG_ID","Atlas_heart","Atlas_brain","Atlas_kidney")
#write.table(atlas.fpkms,file="atlas_fpkms.txt",quote=F,sep="\t")
```

Combining F/RPKM values from public data sets
---------------------------------------------

We will join the data sets on ENSEMBL ID:s, losing a lot of data in the process - but joining on gene symbols or something else would lead to an even worse loss. 

```{r LoadPackagesAndData}
library(org.Hs.eg.db) # for transferring gene identifiers
library(data.table) # for collapsing transcript RPKMs
library(pheatmap) # for nicer visualization
library(edgeR) # for TMM normalization

#hpa.fpkms <- read.delim("hpa_fpkms.txt")
#altiso.fpkms <- read.delim("altiso_fpkms.txt")
#gtex.fpkms <- read.delim("gtex_fpkms.txt")
#atlas.fpkms <- read.delim("atlas_fpkms.txt")
```

The RNA-seq Atlas data set uses many different identifiers, while the other all use ENSG as the primary identifier.

**Approach 1**: Merge on ENSEMBL genes (ENSG) as given in RNA-seq Atlas. Note that there are repeated ENSG ID:s in RNA-seq Atlas, as opposed to the other data sets, so we need to do something about that. In this case, we just sum the transcripts that belong to each ENSG gene. We use data.table for this.

```{r SumTranscripts}
#data.dt <- data.table(atlas.fpkms)
#setkey(data.dt, ENSG_ID)
#temp <- data.dt[, lapply(.SD, sum), by=ENSG_ID]
#collapsed <- as.data.frame(temp)
#atlas.fpkms.summed <- collapsed[,2:ncol(collapsed)] 
#rownames(atlas.fpkms.summed) <- collapsed[,1]

#atlas.fpkms.summed <- atlas.fpkms.summed[2:nrow(atlas.fpkms.summed),]
```

Finally, combine all the data sets into a data frame.

```{r Combine}
#fpkms <- merge(hpa.fpkms, altiso.fpkms, by="ENSG_ID")
#fpkms <- merge(fpkms, gtex.fpkms, by="ENSG_ID")
#fpkms <- merge(fpkms, atlas.fpkms.summed, by.x="ENSG_ID", by.y=0)
#gene_id <- fpkms[,1]
#f <- fpkms[,2:ncol(fpkms)]
#rownames(f) <- gene_id
```

Check how many ENSG IDs we have left.

```{r}
#dim(f)
```

**Approach 2**: Try to map Entrez symbols to ENSEMBL to recover more ENSG IDs than already present in the table. 

```{r RemapENSEMBL}
#m <- org.Hs.egENSEMBL
#mapped_genes <- mappedkeys(m)
#ensg.for.entrez <- as.list(m[mapped_genes])
#remapped.ensg <- ensg.for.entrez[as.character(atlas$entrez_gene_id)]

#atlas.fpkms$remapped_ensg <- as.character(remapped.ensg)

# And add expression values
#data.dt <- data.table(atlas.fpkms[,2:ncol(atlas.fpkms)])
#setkey(data.dt, remapped_ensg)
#temp <- data.dt[, lapply(.SD, sum), by=remapped_ensg]
#collapsed <- as.data.frame(temp)
#atlas.fpkms.summed <- collapsed[,2:ncol(collapsed)] 
#rownames(atlas.fpkms.summed) <- collapsed[,1]
```

Combine data sets again.

```{r Combine2}
#fpkms <- merge(hpa.fpkms, altiso.fpkms, by="ENSG_ID")
#fpkms <- merge(fpkms, gtex.fpkms, by="ENSG_ID")
#fpkms <- merge(fpkms, atlas.fpkms.summed, by.x="ENSG_ID", by.y=0)
#gene_id <- fpkms[,1]
#f <- fpkms[,2:ncol(fpkms)]
#rownames(f) <- gene_id
#write.table(f, file = 'published_rpkms.txt', quote=F)

#instead of downloading everytime:
```

Analyzing public RNA-seq expression data
----------------------------------------

Here is where the actual data analysis starts - after the public data has been downloaded, combined, and written to a single file called published_rpkms.txt

Download the data and sample descriptions from local files:

```{r:published download}
published <- read.delim("published_rpkms.txt",sep=" ")
sampleinfo_published <- read.table("sample_info_published.txt",header=TRUE)
```

Analyses and plots related to Figure 1
--------------------------------------
Figure 1 deals with untransformed (but combined) published data.

The published FPKM values are first filtered by removing all lines where FPKM is less or equal to 0.01 in all samples:

```{r:published filter zeros}
published.nozero <- published[-which(rowSums(published[,])<=0.01),]
```

Heatmap of Spearman correlations between published expression profiles (# genes = 13,323), **Figure 1b**:

```{r:published heatmap spearman}
pheatmap(cor(published.nozero, method="spearman")) 
```
Alternatively, one could use Pearson correlation (not shown in paper):
```{r:published heatmap pearson}
pheatmap(cor(published.nozero))
```
PCA analysis of published FPKM values, **Figure 1c**:
```{r:published PCA}
colors <- c("indianred", "dodgerblue", "forestgreen",
            "indianred", "dodgerblue",
            "indianred", "dodgerblue", "forestgreen", 
            "indianred", "dodgerblue", "forestgreen")
  
p <- prcomp(t(published.nozero))
shapes <- c(rep(15,3),rep(16,2),rep(17,3),rep(8,3))
plot(p$x[,1],p$x[,2],pch=shapes,cex=1.5,col=colors,xlab=paste("PC1 58% of variance"),ylab=paste("PC2 13% of variance"),main="Published FPKM/RPKM values \n n=13,323")
legend("bottomleft",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20)
legend("top",legend=c("HPA","AltIso","GTEx","Atlas"),col="black",pch=c(15,16,17,8),ncol=2)
```

We can plot all pairwise combinations of principal components 1 to 5. (not shown in paper):
```{r:published pairwise PCA}
par(mfrow=c(4,4))
for (i in 1:6){
  for(j in 1:6){
  	if (i<j){ 
		plot(p$x[,i],p$x[,j],pch=20,col=colors,xlab=paste("PC",i),ylab=paste("PC",j),main="Published FPKM values \n n=13,323")
		}
	}
}

```
Look a bit closer at PCs 1-3 in prcomp (not shown in paper):
```{r:published PC 1-3}
plot.highest.loadings(p,published.nozero,caption="Published F/RPKM")
```
             
Try Anova on a "melted" expression matrix with some metadata, **Figure 1d**:
```{r:published anova}
do_anova(published.nozero,sampleinfo_published,"ANOVA, published data")
```
Analyses and plots related to Figure 2
--------------------------------------
Figure 2 deals with the effects of log-transforming the published data.

```{r:published log transform}
pseudo <- 1
published.log <- log2(published.nozero + pseudo)
```
Heatmap of Spearman correlations between published expression profiles with log2 values, **Figure 2a**:

```{r:published log heatmap spearman}
pheatmap(cor(published.log),method="spearman")
```
PCA analysis of log2 transformed published FPKM values, **Figure 2b**:

```{r:published log PCA 1&2}
colors <- c("indianred", "dodgerblue", "forestgreen",
            "indianred", "dodgerblue",
            "indianred", "dodgerblue", "forestgreen", 
            "indianred", "dodgerblue", "forestgreen")
p.log <- prcomp(t(published.log))
shapes <- c(rep(15,3),rep(16,2),rep(17,3),rep(8,3))
plot(p.log$x[,1],p.log$x[,2],pch=shapes,col=colors,xlab=paste("PC1 31% of variance"),ylab=paste("PC2 27% of variance"),main="log2 Published FPKM/RPKM values \n n=13,323")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
legend("top",legend=c("HPA","AltIso","GTEx","Atlas"),col="black",pch=c(15,16,17,8),ncol=2)
```
**Figure 2c**, PCs 2 and 3 for log transformed public data:

```{r:published log PCA 2&3}
plot(p.log$x[,2],p.log$x[,3],pch=shapes,col=colors,xlab=paste("PC2 27% of variance"),ylab=paste("PC3 19% of variance"),main="log2 Published FPKM values \n n=13,323")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
legend("topright",legend=c("HPA","AltIso","GTEx","Atlas"),col="black",pch=c(15,16,17,8),ncol=2)
```
Cross-validation by leaving out one of the studies, in this case AltIso, **Figure 2d**: 

```{r:leave-one-out-pca}
p.loo <- published.log[,-c(4,5)]
colors.loo <- colors[-c(4,5)]
p.loo.log <- prcomp(t(p.loo))
p.add <- published.log[,c(4,5)]
projection <- t(p.add) %*% p.loo.log$rotation
p.original.plus.new <- rbind(p.loo.log$x, projection)
col.original.plus.new <- c(colors.loo, colors[c(4,5)])
plot(p.original.plus.new[,2],p.original.plus.new[,3],pch=c(rep(15,3),rep(17,3),rep(8,3),rep(22,nrow(projection))),col=col.original.plus.new,xlab="PC2",ylab="PC3",main="log2 Published FPKM/RPKM values; AltIso projected onto existing PCs \n n=13,323",xlim=c(-150,100))
legend("bottomleft",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
legend("top",legend=c("HPA","GTEx","Atlas","AltIso"),col="black",pch=c(15,17,8,22),ncol=2)
```

In order to convince ourselves that there is no obviously superior combination of PCs, can plot all pairwise combinations of principal components 1 to 5. (not shown in paper):

```{r:published log PCA pairwise}
par(mfrow=c(4,4))
for (i in 1:6){
  for(j in 1:6){
    if (i<j){ 
		plot(p.log$x[,i],p.log$x[,j],pch=shapes,col=colors,xlab=paste("PC",i),ylab=paste("PC",j),main="log2 Published FPKM values \n n=13323")
		}
	}
}
```

Loadings for PCs 1-3 after taking logs (not shown in paper):

```{r:published log PC 1-3}
plot.highest.loadings(p.log,published.log,caption="Published log F/RPKM")
```

To further validate the above results, indicating that tissue specificity appears mainly in PC 2 and 3, we will extract the 500 genes with highest loadings in each component and plot the corresponding published FPKM values in a heatmap (not shown in paper):

```{r:published log top 500 loadings heatmap}
     load.pc1 <- abs(p.log$rotation[,1])[order(abs(p.log$rotation[,1]),decreasing=TRUE)]
     top.pc1 <- names(load.pc1[1:500]) 

     load.pc2 <- abs(p.log$rotation[,2])[order(abs(p.log$rotation[,2]),decreasing=TRUE)]
     top.pc2 <- names(load.pc2[1:500])

     load.pc3 <- abs(p.log$rotation[,3])[order(abs(p.log$rotation[,3]),decreasing=TRUE)]
     top.pc3 <- names(load.pc3[1:500])

     pheatmap(cor(published[top.pc1,]),method="spearman")
     pheatmap(cor(published[top.pc2,]),method="spearman")
     pheatmap(cor(published[top.pc3,]),method="spearman")
```

Perform Anova on logged values to estimate influence of different sample properties (**Figure 2e**):

```{r:published log anova}
do_anova(published.log,sampleinfo_published,"ANOVA, published data (log)")
```
Analyses relating to Figure 3
-----------------------------

Combat analysis is performed on log2 values (n=13,323):

```{r:published log combat}
meta <- data.frame(study=c(rep("HPA",3),rep("AltIso",2),rep("GTex",3),rep("Atlas",3)),tissue=c("Heart","Brain","Kidney","Heart","Brain","Heart","Brain","Kidney","Heart","Brain","Kidney"))
batch <- meta$study
design <- model.matrix(~1,data=meta)
combat <- ComBat(dat=published.log,batch=batch,mod=design,numCovs=NULL,par.prior=TRUE)
```
Heatmap of Spearman correlations between published expression profiles after combat run (# genes = 13,323), **Figure 3a**:

```{r:published log combat heatmap spearman}
pheatmap(cor(combat, method="spearman")) 
```

PCA analysis of published FPKM values after ComBat run, **Figure 3b**:
```{r:published log combat PCA}
colors <- c("indianred", "dodgerblue", "forestgreen",
            "indianred", "dodgerblue",
            "indianred", "dodgerblue", "forestgreen", 
            "indianred", "dodgerblue", "forestgreen")
p.combat <- prcomp(t(combat))
shapes <- c(rep(15,3),rep(16,2),rep(17,3),rep(8,3))
plot(p.combat$x[,1],p.combat$x[,2],pch=shapes,col=colors,xlab=paste("PC1 54% of variance"),ylab=paste("PC2 38% of variance"),main="Published FPKM values \n COMBAT \n n=13,323")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
legend("topright",legend=c("HPA","AltIso","GTEx","Atlas"),col="black",pch=c(15,16,17,8),ncol=2)
```
Loadings of PCs 1-3 after ComBat (not shown in paper):

```{r:published log combat PC 1-3}
plot.highest.loadings(p.combat,combat,caption="Published ComBat adj log F/RPKM")
```

Revisit Anova with "combated"" values, **Figure 3c**:

```{r:published log combat anova}
do_anova(combat, sampleinfo_published, caption="ANOVA, ComBat adj log F/RPKM")
```
Analyses relating to Figure 4 (data consistently reprocessed from FASTQ)
------------------------------------------------------------------------

All of the preceding analysis was for studies with published F/RPKM values.
We now turn to FPKMs for FASTQ files reprocessed with TopHat and Cufflinks:

```{r:cufflinks download data}
cufflinks <- read.delim("fpkm_table_tophat.txt")
sampleinfo_cufflinks <- read.delim("sample_info_reprocessed.txt")
```
First, we will restrict the data set to only include protein coding genes using the ensembl based R package biomaRt:

```{r:cufflinks filter protein coding}
gene_ids <- as.vector(cufflinks[,1])
ensembl = useMart("ensembl", dataset = "hsapiens_gene_ensembl") #select the ensembl database
gene_type <- getBM(attributes=c("ensembl_gene_id", "gene_biotype"), 
                   filters = "ensembl_gene_id",
                   values=gene_ids,
                   mart=ensembl)
pc <- subset(gene_type[,1],gene_type[,2]=="protein_coding")
cufflinks_pc <- cufflinks[match(pc,cufflinks[,1]),]
```

Let's remove all lines where FPKM is close to zero in all samples before we proceed with this version of the data set:

```{r:cufflinks filter zeros}
cufflinks_pc_nozero <- cufflinks_pc[-which(rowSums(cufflinks_pc[,3:16])<=0.01),]
```
Heatmap of Spearman correlations between reprocessed expression profiles (# genes = 19,475), **Figure 4a**:
```{r:cufflinks heatmap spearman}
pheatmap(cor(cufflinks_pc_nozero[,3:16], method="spearman")) 
```

Let's look at a few PCA plots, **Figure 4b**:

```{r:cufflinks PCA}
cufflinks_fpkms <- cufflinks_pc_nozero[,3:16]
rownames(cufflinks_fpkms) <- cufflinks_pc_nozero[,1]
p.cufflinks <- prcomp(t(cufflinks_fpkms))
colors <- c("dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred")          
shapes_cufflinks <- c(rep(11,3),rep(8,3),rep(17,3),rep(15,3),rep(16,2))
plot(p.cufflinks$x[,1],p.cufflinks$x[,2],pch=shapes_cufflinks,col=colors,xlab=paste("PC1 87% of variance"),ylab=paste("PC2 7.7% of variance"),main="Reprocessed FPKM values \n n=19,475")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
legend("top",legend=c("EoGE","Atlas","BodyMap","HPA","AltIso"),col="black",pch=c(11,8,17,15,16),ncol=2)
```

All pairwise combinations of principal components 1 to 5 (not shown in paper):

```{r:cufflinks PCA pairwise}
colors <- c("dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred")
par(mfrow=c(4,4))
for (i in 1:6){
  for(j in 1:6){
    if (i<j){ 
      plot(p.cufflinks$x[,i],p.cufflinks$x[,j],pch=shapes_cufflinks,col=colors,xlab=paste("PC",i),ylab=paste("PC",j),main="Cufflinks FPKM values \n n=19475")
		}
	}
}
```

Look at PCA loadings for PC1-3 (not shown in paper):

```{r:cufflinks PC 1-3}
plot.highest.loadings(p.cufflinks,cufflinks_fpkms,caption="Cufflinks FPKM")    
```
Mitochondrially encoded genes have relatively high expression levels, FPKM values of several thousands.

Anova analysis of different sample properties, **Figure 4c**:

```{r:cufflinks anova}
do_anova(cufflinks_fpkms,sampleinfo_cufflinks,caption="ANOVA, Cufflinks FPKM")
```
Try log2 transformation of the reprocessed FPKM values:
```{r:cufflinks log transform}
pseudo <- 1
cufflinks_log <- log2(cufflinks_fpkms + pseudo)
```
Heatmap of Spearman correlations between log2 reprocessed cufflinks FPKM values, **Figure 4d**:

```{r:cufflinks log heatmap spearman }
pheatmap(cor(cufflinks_log) ,method="spearman")
```
PCA analysis of log2 reprocessed cufflinks FPKM values, **Figure 4e**:
```{r:cufflinks log PCA 1&2}
colors <- c("dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred")          
p.log.cufflinks <- prcomp(t(cufflinks_log))
shapes_cufflinks <- c(rep(11,3),rep(8,3),rep(17,3),rep(15,3),rep(16,2))
plot(p.log.cufflinks$x[,1],p.log.cufflinks$x[,2],pch=shapes_cufflinks,col=colors,xlab=paste("PC1 33% of variance"),ylab=paste("PC2 25% of variance"),main="log2 reprocessed cufflinks FPKM values \n n=19,475")
legend("bottomleft",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
legend("top",legend=c("EoGE","Atlas","BodyMap","HPA","AltIso"),col="black",pch=c(11,8,17,15,16),ncol=2)
```
**Figure 4f**:

```{r:cufflinks log PCA 2&3}
plot(p.log.cufflinks$x[,2],p.log.cufflinks$x[,3],pch=shapes_cufflinks,col=colors,xlab=paste("PC2 25% of variance"),ylab=paste("PC3 22% of variance"),main="log2 reprocessed cufflinks FPKM values \n n=19,475")
legend("bottomleft",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
legend("topleft",legend=c("EoGE","Atlas","BodyMap","HPA","AltIso"),col="black",pch=c(11,8,17,15,16),ncol=2)
```
All pairwise combinations of principal components 1 to 5. (not shown in paper):
```{r:cufflinks log PCA pairwise}
par(mfrow=c(4,4))
for (i in 1:6){
  for(j in 1:6){
    if (i<j){ 
  	plot(p.log.cufflinks$x[,i],p.log.cufflinks$x[,j],pch=shapes_cufflinks,col=colors,xlab=paste("PC",i),ylab=paste("PC",j),main="log2 reprocessed FPKM values \n n=19475")
		}
	}
}
```
Cross-validation by leaving out one of the studies, in this case AltIso, **Figure 4g**: 
```{r:leave-one-out-pca cufflinks}
colors <- c("dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred", "forestgreen",
            "dodgerblue", "indianred") 
p.loo <- cufflinks_log[,-c(13,14)]
colors.loo <- colors[-c(13,14)]
p.loo.log <- prcomp(t(p.loo))
p.add <- cufflinks_log[,c(13,14)]
projection <- t(p.add) %*% p.loo.log$rotation
p.original.plus.new <- rbind(p.loo.log$x, projection)
col.original.plus.new <- c(colors.loo, colors[c(13,14)])
plot(p.original.plus.new[,2],p.original.plus.new[,3],pch=c(shapes_cufflinks[1:12],rep(22,nrow(projection))),col=col.original.plus.new,xlab="PC2",ylab="PC3",main="log2 Cufflinks FPKM values; AltIso projected onto existing PCs \n n=19,475,",xlim=c(-150,100))
legend("bottomleft",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
legend("topleft",legend=c("HPA","GTEx","Atlas","AltIso"),col="black",pch=c(11,8,17,15,22),ncol=2)
```
Loadings for PCs 1-3 for logged FPKM values from Cufflinks (not shown in paper):
```{r:cufflinks log PC 1-3}
plot.highest.loadings(p.log.cufflinks,cufflinks_log,caption="Cufflinks log FPKM")
```
To further validate the above results, indicating that tissue specificity appears mainly in PC 3, we will extract the 500 genes with highest loadings in each component and plot the corresponding cufflinks FPKM values in a heatmap (not shown in paper):

```{r:cufflinks log top 500 loadings heatmap}
     cufflinks_values <- cufflinks_pc_nozero[,3:16]
     rownames(cufflinks_values) <- cufflinks_pc_nozero[,1]
     load.pc1 <- abs(p.log.cufflinks$rotation[,1])[order(abs(p.log.cufflinks$rotation[,1]),decreasing=TRUE)]
     top.pc1 <- names(load.pc1[1:500])
     load.pc2 <- abs(p.log.cufflinks$rotation[,2])[order(abs(p.log.cufflinks$rotation[,2]),decreasing=TRUE)]
     top.pc2 <- names(load.pc2[1:500])
     load.pc3 <- abs(p.log.cufflinks$rotation[,3])[order(abs(p.log.cufflinks$rotation[,3]),decreasing=TRUE)]
     top.pc3 <- names(load.pc3[1:500])
     
     pheatmap(cor(cufflinks_values[top.pc1,]),method="spearman")
     pheatmap(cor(cufflinks_values[top.pc2,]),method="spearman")
     pheatmap(cor(cufflinks_values[top.pc3,]),method="spearman")

```
Anova on logged Cufflinks values (not shown in paper):

```{r:cufflinks log anova}
do_anova(cufflinks_log,sampleinfo_cufflinks,caption="ANOVA, Cufflinks log FPKM")
```
Combat analysis for removal of batch effects (n=19,475):
```{r:cufflinks log combat}
meta <- data.frame(study=c(rep("EoGE",3),rep("Atlas",3),rep("BodyMap",3),rep("HPA",3),rep("AltIso",2)),tissue=c("Brain","Heart","Kidney","Brain","Heart","Kidney","Brain","Heart","Kidney","Brain","Heart","Kidney","Brain","Heart"),prep=c(rep("poly-A",3),rep("rRNA-depl",3),rep("poly-A",8)),layout=c(rep("PE",3),rep("SE",3),rep("PE",6),rep("SE",2)))
batch <- meta$study
design <- model.matrix(~1,data=meta)
combat.cufflinks <- ComBat(dat=cufflinks_log,batch=batch,mod=design,numCovs=NULL,par.prior=TRUE)
rownames(combat.cufflinks) <- rownames(cufflinks_log)
```

Heatmap of Spearman correlations between reprocessed cufflinks FPKM values after ComBat run, **Figure 4h**:

```{r:cufflinks log combat heatmap spearman}
pheatmap(cor(combat.cufflinks),method="spearman")
```

PCA analysis on reprocessed cufflinks FPKM values after ComBat run, **Figure 4i**:
```{r:cufflinks log combat PCA}
p.combat.cufflinks <- prcomp(t(combat.cufflinks))
shapes_cufflinks <- c(rep(11,3),rep(8,3),rep(17,3),rep(15,3),rep(16,2))
plot(p.combat.cufflinks$x[,1],p.combat.cufflinks$x[,2],pch=shapes_cufflinks,col=colors,xlab=paste("PC1 54% of variance"),ylab=paste("PC2 37% of variance"),main="Cufflinks FPKM values \n COMBAT \n n=19,475")
legend("bottomleft",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
legend("top",legend=c("EoGE","Atlas","BodyMap","HPA","AltIso"),col="black",pch=c(11,8,17,15,16),ncol=2)
```
Loadings for PCs 1-3 in prcomp (not shown in paper):
```{r:cufflinks log combat PC 1-3}
plot.highest.loadings(p.combat.cufflinks,combat.cufflinks,caption="Cufflinks ComBat adj log FPKM")
```
Revisit ANOVA analysis on reprocessed Cufflinks FPKM values after ComBat run, **Figure 4j**:
```{r:cufflinks log combat anova}
do_anova(combat.cufflinks,sampleinfo_cufflinks,caption="ANOVA, Cufflinks ComBat adj log FPKM")
```
Perform a joint analysis of both published and reprocessed data to try to quantify the influence of using different bioinformatics pipelines.
```{r:joint}
j <- merge(published.nozero, cufflinks_pc_nozero, by.x=0, by.y="ENSEMBL_ID")
j <- j[,-which(colnames(j)=="Gene_ID"),]
rown <- j[,1]
j <- j[,2:ncol(j)]
rownames(j) <- rown
```
Heatmap and PCA of untransformed joint data (not shown in paper).
```{r}
pheatmap(cor(j))
```
Here, RNA-seq Atlas forms its own group, whereas the other studies form tissue-specific groups.

PCA (not shown):
```{r}
#par(mfrow=c(2,2))
colors <- c("indianred", "dodgerblue", "forestgreen","indianred", "dodgerblue", "indianred", "dodgerblue", "forestgreen","indianred", "dodgerblue", "forestgreen","dodgerblue","indianred","forestgreen","dodgerblue","indianred","forestgreen","dodgerblue","indianred","forestgreen","dodgerblue","indianred","forestgreen","dodgerblue","indianred")
shapes <- c(rep(0,3),rep(1,2),rep(8,3),rep(2,3),rep(6,3),rep(17,3),rep(5,3),rep(15,3),rep(16,2))
p.joint <- prcomp(t(j))
plot(p.joint$x[,1],p.joint$x[,2],col=colors,pch=shapes,main="Raw F/RPKM PC1/2")
legend("topright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
legend("top",legend=c("HPA-P","HPA-R","AltIso-P","AltIso-R", "Atlas-P", "Atlas-R", "GTeX-P","EoGE-P", "BodyMap-R"),col="black",pch=c(0,15,1,16,2,17,8,6,5),ncol=2)
```

After log transformation (not shown):
```{r}
j.log <- log2(j+1)
p.joint.log <- prcomp(t(j.log))
plot(p.joint.log$x[,1],p.joint.log$x[,2],pch=shapes,col=colors,main="log F/RPKM PC1/2")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
legend("bottomleft",legend=c("HPA-P","HPA-R","AltIso-P","AltIso-R", "Atlas-P", "Atlas-R", "GTeX-P","EoGE-P", "BodyMap-R"),col="black",pch=c(0,15,1,16,2,17,8,6,5),ncol=2)
plot(p.joint.log$x[,2],p.joint.log$x[,3],pch=shapes,col=colors,main="log F/RPKM PC2/3")
legend("bottomright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
legend("bottomleft",legend=c("HPA-P","HPA-R","AltIso-P","AltIso-R", "Atlas-P", "Atlas-R", "GTeX-P","EoGE-P", "BodyMap-R"),col="black",pch=c(0,15,1,16,2,17,8,6,5),ncol=2)
```

ComBat (not shown):
```{r}
meta <- data.frame(study=c(rep("HPA",3),rep("AltIso",2),rep("GTex",3),rep("Atlas",3),rep("EoGE",3),rep("Atlas",3),rep("BodyMap",3),rep("HPA",3),rep("AltIso",2)),tissue=c("Heart","Brain","Kidney","Heart","Brain","Heart","Brain","Kidney","Heart","Brain","Kidney","Brain","Heart","Kidney","Brain","Heart","Kidney","Brain","Heart","Kidney","Brain","Heart","Kidney","Brain","Heart"),quantification=c(rep("other",11),rep("tophatcufflinks",14)))
batch <- meta$study
design <- model.matrix(~1,data=meta)
combat.j <- ComBat(dat=j.log,batch=batch,mod=design,numCovs=NULL,par.prior=TRUE)
p.com <- prcomp(t(combat.j))
plot(p.com$x[,1],p.com$x[,2],pch=shapes,col=colors,main="ComBat PC1/2")
legend("topright",legend=c("Heart","Brain","Kidney"),col=c("indianred", "dodgerblue", "forestgreen"),cex=1.5,pch=20,bty="n")
legend("bottom",legend=c("HPA-P","HPA-R","AltIso-P","AltIso-R", "Atlas-P", "Atlas-R", "GTeX-P","EoGE-P", "BodyMap-R"),col="black",pch=c(0,15,1,16,2,17,8,6,5),ncol=2)
```

ANOVAs including quantification method (**Figure 4k**?).
```{r}
sampleinfo_cufflinks$quantification <- "topcuff"
sampleinfo_published$quantification <- c(rep("topcuff",3),rep("custom_AltIso",2),rep("fluxcapacitor",3),rep("custom_Atlas",3))
studylabels_cuff <- c("EoGE_brain","EoEG_heart","EoEG_kidney","Atlas_brain","Atlas_heart","Atlas_kidney","BodyMap_brain","BodyMap_heart","BodyMap_kidney","HPA_brain","HPA_heart","HPA_kidney","AltIso_brain","AltIso_heart")
sampleinfo_cufflinks <- data.frame(Study_labels=studylabels_cuff, sampleinfo_cufflinks)
sampleinfo <- rbind(sampleinfo_published, sampleinfo_cufflinks)
par(mfrow=c(3,1))
do_anova(j, sampleinfo, caption="Joint analysis, raw F/RPKM", include.quant=T)
do_anova(j.log, sampleinfo, caption="Joint analysis, log F/RPKM", include.quant=T)
do_anova(combat.j, sampleinfo, caption="Joint analysis, ComBat adj log F/RPKM", include.quant=T)
```
Analyses relating to Supplementary Figure X.
--------------------------------------------
Correlations between PCs 1 and 2 and various factors (Supplementary figure X).

```{r}
print_PCA_corrs(published,sampleinfo_published,caption="Published F/RPKM")
print_PCA_corrs(published.log,sampleinfo_published,caption="Published log F/RPKM")
print_PCA_corrs(combat,sampleinfo_published,caption="Published ComBat adj log F/RPKM")

print_PCA_corrs(cufflinks_fpkms,sampleinfo_cufflinks,caption="Reprocessed F/RPKM")
print_PCA_corrs(cufflinks_log,sampleinfo_cufflinks,caption="Reprocessed log F/RPKM")
print_PCA_corrs(combat.cufflinks,sampleinfo_cufflinks,caption="Reprocessed ComBat adj log F/RPKM")

print_PCA_corrs(j,sampleinfo,caption="Joint analysis, F/RPKM",include.quant=T)
print_PCA_corrs(j.log,sampleinfo,caption="Joint analysis, log F/RPKM",include.quant=T)
print_PCA_corrs(combat.j,sampleinfo,caption="Joint analysis, ComBat adj log F/RPKM",include.quant=T)
```
Analyses relating to Supplementary Figure Y.
--------------------------------------------
Let's compare the FPKM distribution in the samples before and after merge

```{r:FPKM distribution for merged data vs non-merged data}

#-----------DOWNLOAD DATA BEFORE MERGE-----------
altIso_all <- read.delim("altiso_fpkms.txt",sep="\t",stringsAsFactors=FALSE)
atlas_all <- read.delim("atlas_fpkms.txt",sep="\t",stringsAsFactors=FALSE)
gtex_all <- read.delim("gtex_fpkms.txt",sep="\t",stringsAsFactors=FALSE)
HPA_all <- read.delim("hpa_fpkms.txt",sep="\t",stringsAsFactors=FALSE)

#filter out all protein coding genes from the non merged lists:

ensembl = useMart("ensembl",dataset= "hsapiens_gene_ensembl")
getPcs <- function(x) {
            type <- getBM(attributes=c("ensembl_gene_id","gene_biotype"),filters = "ensembl_gene_id",values = x$ENSG_ID,mart = ensembl)
            results <- x[which(type[,2]=="protein_coding"),]
            return(results)
          }

altIso_pc <- getPcs(altIso_all)
atlas_pc <- getPcs(atlas_all)
gtex_pc <- getPcs(gtex_all)
HPA_pc <- getPcs(HPA_all)


#-----------DOWNLOAD MERGED DATA-----------
merged_data <- read.delim("published_rpkms.txt",sep=" ")

#-----------DOWNLOAD REPROCESSED DATA-----------
reprocessed_data <- read.table("fpkm_table_tophat.txt",sep="\t",header=T)
ensembl = useMart("ensembl",dataset= "hsapiens_gene_ensembl")
getPcs <- function(x) {
            type <- getBM(attributes=c("ensembl_gene_id","gene_biotype"),filters = "ensembl_gene_id",values = x$ENSEMBL_ID,mart = ensembl)
            results <- x[which(type[,2]=="protein_coding"),]
            return(results)
          }

reprocessed_data_pc <- getPcs(reprocessed_data)

#-----------ADJUST DATA-----------
get_data <- function(x,y,z,k) {
  
  if(k==1){
    if(y=="heart"){
              data <- data.frame(x[grep("heart", names(x), value = TRUE)])
              }
            else if(y=="brain"){
                data <- data.frame(x[grep("brain", names(x), value = TRUE)])
              }
            else{
              data <- data.frame(x[grep("kidney", names(x), value = TRUE)])
            }
            colnames(data) <- "FPKM"
            data$study <- z
            return(data)
  }
  #om det är det mergade datat:
  else if(k==2){
    if(y=="heart"){
              tissue_data <- data.frame(x[grep("heart", names(x), value = TRUE)])
              data <- data.frame(tissue_data[grep(z, names(tissue_data), value = TRUE)])
              }
            else if(y=="brain"){
                tissue_data <- data.frame(x[grep("brain", names(x), value = TRUE)])
                data <- data.frame(tissue_data[grep(z, names(tissue_data), value = TRUE)])
              }
            else{
              tissue_data <- data.frame(x[grep("kidney", names(x), value = TRUE)])
              data <- data.frame(tissue_data[grep(z, names(tissue_data), value = TRUE)])
            }
            colnames(data) <- "FPKM"
            data$study <- z
            return(data)
  }
  }
           
#-----------NON-MERGED DATA-----------
all_altIso_heart <- get_data(altIso_pc,"heart","AltIso",1)
all_atlas_heart <- get_data(atlas_pc,"heart","atlas",1)
all_gtex_heart <- get_data(gtex_pc,"heart","gtex",1)
all_HPA_heart <- get_data(HPA_pc,"heart","HPA",1)

all_altIso_brain <- get_data(altIso_pc,"brain","AltIso",1)
all_atlas_brain <- get_data(atlas_pc,"brain","atlas",1)
all_gtex_brain <- get_data(gtex_pc,"brain","gtex",1)
all_HPA_brain <- get_data(HPA_pc,"brain","HPA",1)

all_atlas_kidney <- get_data(atlas_pc,"kidney","atlas",1)
all_gtex_kidney <- get_data(gtex_pc,"kidney","gtex",1)
all_HPA_kidney <- get_data(HPA_pc,"kidney","HPA",1)

#-----------MERGED DATA-----------
merg_altIso_heart <- get_data(altIso_pc,"heart","AltIso",2)
merg_atlas_heart <- get_data(atlas_pc,"heart","Atlas",2)
merg_gtex_heart <- get_data(gtex_pc,"heart","GTEx",2)
merg_HPA_heart <- get_data(HPA_pc,"heart","HPA",2)

merg_altIso_brain <- get_data(altIso_pc,"brain","AltIso",2)
merg_atlas_brain <- get_data(atlas_pc,"brain","Atlas",2)
merg_gtex_brain <- get_data(gtex_pc,"brain","GTEx",2)
merg_HPA_brain <- get_data(HPA_pc,"brain","HPA",2)

merg_atlas_kidney <- get_data(atlas_pc,"kidney","Atlas",2)
merg_gtex_kidney <- get_data(gtex_pc,"kidney","GTEx",2)
merg_HPA_kidney <- get_data(HPA_pc,"kidney","HPA",2)

#--------------RE-PROCESSED DATA-----------------------------

get_data <- function(x,y,z) {
    if(y=="heart"){
              tissue_data <- data.frame(x[grep("heart", names(x), value = TRUE)])
              data <- data.frame(tissue_data[grep(z, names(tissue_data), value = TRUE)])
              }
            else if(y=="brain"){
                tissue_data <- data.frame(x[grep("brain", names(x), value = TRUE)])
                data <- data.frame(tissue_data[grep(z, names(tissue_data), value = TRUE)])
              }
            else{
              tissue_data <- data.frame(x[grep("kidney", names(x), value = TRUE)])
              data <- data.frame(tissue_data[grep(z, names(tissue_data), value = TRUE)])
            }
            colnames(data) <- "FPKM"
            data$study <- paste(z,"_repr.",sep ="")
            return(data)
  }
  

repr_altIso_heart <- get_data(reprocessed_data_pc,"heart","AltIso")
repr_atlas_heart <- get_data(reprocessed_data_pc,"heart","Atlas")
repr_Bodymap_heart <- get_data(reprocessed_data_pc,"heart","BodyMap")
repr_EoGE_heart <- get_data(reprocessed_data_pc,"heart","EoGE")
repr_HPA_heart <- get_data(reprocessed_data_pc,"heart","HPA")

repr_altIso_brain <- get_data(reprocessed_data_pc,"brain","AltIso")
repr_atlas_brain <- get_data(reprocessed_data_pc,"brain","Atlas")
repr_Bodymap_brain <- get_data(reprocessed_data_pc,"brain","BodyMap")
repr_EoGE_brain <- get_data(reprocessed_data_pc,"brain","EoGE")
repr_HPA_brain <- get_data(reprocessed_data_pc,"brain","HPA")

repr_atlas_kidney <- get_data(reprocessed_data_pc,"kidney","Atlas")
repr_Bodymap_kidney <- get_data(reprocessed_data_pc,"kidney","BodyMap")
repr_EoGE_kidney <- get_data(reprocessed_data_pc,"kidney","EoGE")
repr_HPA_kidney <- get_data(reprocessed_data_pc,"kidney","HPA")

#--------------PLOT DISTRIBUTIONS NON-MERGED DATA-----------------------------

all_altiso <- rbind(all_altIso_brain,all_altIso_heart)
all_atlas <- rbind(all_atlas_brain,all_atlas_heart,all_atlas_kidney)
all_gtex <- rbind(all_gtex_brain,all_gtex_heart,all_gtex_kidney)
all_hpa <- rbind(all_HPA_brain,all_HPA_heart,all_HPA_kidney)

all_data_beforemerge <- rbind(all_altiso,all_atlas,all_gtex,all_hpa)

brain_data_beforemerge <- rbind(all_altIso_brain,
                                all_atlas_brain,
                                all_gtex_brain,
                                all_HPA_brain)

heart_data_beforemerge <- rbind(all_altIso_heart,
                                all_atlas_heart,
                                all_gtex_heart,
                                all_HPA_heart)


kidney_data_beforemerge <- rbind(all_atlas_kidney,
                                 all_gtex_kidney,
                                 all_HPA_kidney)



ggplot(all_data_beforemerge, aes(log2(FPKM+1), colour = study)) + geom_density(alpha = 0.2) + xlim(0, 7) + theme_bw()
ggplot(brain_data_beforemerge, aes(log2(FPKM+1), colour = study)) + geom_density(alpha = 0.2) + xlim(0, 7) + theme_bw()
ggplot(heart_data_beforemerge, aes(log2(FPKM+1), colour = study)) + geom_density(alpha = 0.2) + xlim(0, 7) + theme_bw()
ggplot(kidney_data_beforemerge, aes(log2(FPKM+1), colour = study)) + geom_density(alpha = 0.2) + xlim(0, 7) + theme_bw()


#--------------PLOT DISTRIBUTIONS MERGED DATA-----------------------------
merg_altiso <- rbind(merg_altIso_brain,merg_altIso_heart)
merg_atlas <- rbind(merg_atlas_brain,merg_atlas_heart,merg_atlas_kidney)
merg_gtex <- rbind(merg_gtex_brain,merg_gtex_heart,merg_gtex_kidney)
merg_hpa <- rbind(merg_HPA_brain,merg_HPA_heart,merg_HPA_kidney)

merged_data <- rbind(merg_altiso,merg_atlas,merg_gtex,merg_hpa)

brain_data_merged <- rbind(merg_altIso_brain,
                           merg_atlas_brain,
                           merg_gtex_brain,
                           merg_HPA_brain)

heart_data_merged <- rbind(merg_altIso_heart,
                           merg_atlas_heart,
                           merg_gtex_heart,
                           merg_HPA_heart)

kidney_data_merged <- rbind(merg_atlas_kidney,
                            merg_gtex_kidney,
                            merg_HPA_kidney)



ggplot(merged_data, aes(log2(FPKM+1), colour = study)) + geom_density(alpha = 0.2) + xlim(0,7) + theme_bw()
ggplot(brain_data_merged, aes(log2(FPKM+1), colour = study)) + geom_density(alpha = 0.2) + xlim(0, 7) + theme_bw()
ggplot(heart_data_merged, aes(log2(FPKM+1), colour = study)) + geom_density(alpha = 0.2) + xlim(0, 7) + theme_bw()
ggplot(kidney_data_merged, aes(log2(FPKM+1), colour = study)) + geom_density(alpha = 0.2) + xlim(0, 7) + theme_bw()

#--------------PLOT DISTRIBUTIONS  DATA-----------------------------

repr_altiso <- rbind(repr_altIso_brain,repr_altIso_heart)
repr_atlas <- rbind(repr_atlas_brain,repr_atlas_heart,repr_atlas_kidney)
repr_BodyMap <- rbind(repr_Bodymap_brain,repr_Bodymap_heart,repr_Bodymap_kidney)
repr_EoGE <- rbind(repr_EoGE_brain,repr_EoGE_heart,repr_EoGE_kidney)
repr_HPA <- rbind(repr_HPA_brain,repr_HPA_heart,repr_HPA_kidney)


repr_data <- rbind(repr_altiso,repr_atlas,repr_BodyMap,repr_EoGE,repr_HPA)

brain_data_repr <- rbind(repr_altIso_brain,
                           repr_atlas_brain,
                           repr_Bodymap_brain,
                           repr_EoGE_brain,
                           repr_HPA_brain)

heart_data_repr <- rbind(repr_altIso_heart,
                           repr_atlas_heart,
                           repr_Bodymap_heart,
                           repr_EoGE_heart,
                           repr_HPA_heart)

kidney_data_repr <- rbind(repr_atlas_kidney,
                           repr_Bodymap_kidney,
                           repr_EoGE_kidney,
                           repr_HPA_kidney)

ggplot(repr_data, aes(log2(FPKM+1), colour = study)) + geom_density(alpha = 0.2) + xlim(0,7) + theme_bw()
ggplot(brain_data_repr, aes(log2(FPKM+1), colour = study)) + geom_density(alpha = 0.2) + xlim(0, 7) + theme_bw()
ggplot(heart_data_repr, aes(log2(FPKM+1), colour = study)) + geom_density(alpha = 0.2) + xlim(0, 7) + theme_bw()
ggplot(kidney_data_repr, aes(log2(FPKM+1), colour = study)) + geom_density(alpha = 0.2) + xlim(0, 7) + theme_bw()



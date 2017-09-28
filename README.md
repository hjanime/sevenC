# chromloop
R package to predict chromatin looping interactions from ChIP-seq data and 
sequnece motifs.

Chromatin looping is an importent feature of
  eukaryotic genomes and can bring regulatory sequences, such as enhancers or 
  transcription factor binding sites, in close physical proximity of regulated target genes.
  Here, we provide a tool that uses protein binding signals from ChIP-seq and
  sequence motif information to predict chromatin looping events. 
  Cross-linking of proteins binding close to loop anchors result in ChIP-seq 
  signals at both anchor loci. These signals are used at CTCF motif pairs 
  together their distance and orientation to each other to predict chromatin 
  looping interactions. The looping interactions can associate enhancers or 
  transcription factor binding sites (ChIP-seq peaks) to regulated genes.

## Intallation

The chromloop package depends on the follwing R packages from Bioconductor:

- rtracklayer (>= 1.34.1),
- InteractionSet (>= 1.2.0),

```R
source("https://bioconductor.org/biocLite.R")
biocLite("rtracklayer", "InteractionSet")
```

Install chromloop from github using devtools:

```R
#install.packages("devtools")
devtools::install_github("ibn-salem/chromloop")
```

## Usage example
Here we show how to use the package to predict chromatin looping interactions 
among CTCF moif locations on chromosome 22. 
As input only a single coverage file is used from a STAT1 ChIP-seq experiment 
in human GM12878 cells. 

### Load motifs and ChIP-seq data
```R
library(chromloop)

# load provided CTCF motifs
motifs <- motif.hg19.CTCF.chr22

# use example ChIP-seq coverage file
bigWigFile <- system.file("extdata", "GM12878_Stat1.chr22_1-18000000.bigWig", 
  package = "chromloop")

# add ChIP-seq coverage
motifs <- addCovToGR(motifs, bigWigFile)

```

### Get pairs and correlation of ChIP-seq coverage
```R
# get pairs of motifs as GInteraction object
gi <- getCisPairs(motifs, maxDist = 10^6)

# add motif orientation
gi <- addStrandCombination(gi)

# add motif score
gi <- addMotifScore(gi, "sig")

# compute correlation of ChIP-seq profiles
gi <- applyToCloseGI(gi, "cov", fun = cor)
```

### Train predictor with known loops
```R
# parse known loops
knownLoopFile <- system.file("extdata", 
  "GM12878_HiCCUPS.chr22_1-18000000.loop.txt", package="chromloop")

knownLoops <- parseLoopsRao(knownLoopFile)

# add known loops
gi <- addInteractionSupport(gi, knownLoops)

# train model 
fit <- glm(loop ~ cor + dist + strandOrientation + score_min, 
  data = mcols(gi), 
  family = binomial())
```

### Predict loops
```R
# predict loops
gi$pred <- predict(fit, type = "response", newdata = mcols(gi)) 

# plot prediction score 
boxplot(gi$pred ~ gi$loop)

```

### Write predicted loops to output file
Since looping interactions are stored as `GInteraction` objects internaly, they 
can be exported as 
[BED-PE](http://bedtools.readthedocs.io/en/latest/content/general-usage.html#bedpe-format) 
files using functions from 
[`GenomicInteractions`](https://bioconductor.org/packages/release/bioc/html/GenomicInteractions.html) 
package.

```R
require(GenomicInteractions)

# export to output file
export.bedpe(gi, "interactions.bedpe", score = "pred")

```


--- 
published: true
title: R packages
layout: post
author: John
category: articles
tags: 
- projects
- R packages

---

I've started to organize my commonly used R code into libraries, and have made them available on GitHub. Note that these are not yet well-commented and don't have unit tests. Please open issues for any bugs you find.

The available packages are:

* <a href="https://github.com/jdidion/knitrtools">knitrtools</a>: Some useful tools for use with knitr document generation. Of particular interest is the `figr.ref` function, which relies on <a href="https://github.com/mkoohafkan/kfigr">kfigr</a> and provides nice formatting of intra-document references.
* <a href="https://github.com/jdidion/fancyplots">fancyplots</a>: Implementation of specific plot types, mostly for genetic/genomic data including phylogenetic trees (see `radial.phylog`), spatial data (see `plot.genome.data`), and GWAS results.
* <a href="https://github.com/jdidion/statTests">statTests</a>: Implementation of various statistical tests and distributions.
* <a href="https://github.com/jdidion/intensities">intensities</a>: Methods for transformation and plotting of hybridization array intensity data. Includes a single-method implementation of <a href="https://baseplugins.thep.lu.se/wiki/se.lu.onk.IlluminaSNPNormalization">tQN normalization</a>.
*  <a href="https://github.com/jdidion/geneticsFormats">geneticsFormats</a>: Methods for converting between different genetics formats.
* <a href="https://github.com/jdidion/DEUtils">DEUtils</a>: Convenience methods for working with RNA-Seq differential expression R packages, including DESeq2, DEXSeq, and GO enrichment.
* <a href="https://github.com/jdidion/miscUtils">miscUtils</a>: Some miscellaneous functions, mostly for working with IO and collection data types.
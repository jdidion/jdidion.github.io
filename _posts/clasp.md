--- 
published: true
title: CLASP
layout: post
author: John
category: articles
tags: 
- projects
- R Packages

---

# CLASP

## Cell Line Authentication by SNP Profiling

## R Package: <a href="https://github.com/jdidion/clasp">CLASP</a>

### Summary

I helped to develop an Illumina genotyping array for the mouse called <a href="http://www.neogen.com/Genomics/pdf/Slicks/MegaMUGAFlyer.pdf">MegaMUGA. We used this array to genotype a large number of laboratory strains and also cell lines derived from those strains. I developed an R package to build a database from the genotype data (both raw intensities and genotype calls) and use that data to develop an optimized assay for validating the genetic background of the cell lines. The package also estimates cross-contamination and copy number.

### Reference

1.	Didion, J. P. et al. SNP array profiling of mouse cell lines identifies their strains of origin and reveals cross-contamination and widespread aneuploidy. BMC Genomics 15, 847 (2014).
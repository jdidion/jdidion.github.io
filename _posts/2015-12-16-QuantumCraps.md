--- 
published: true
title: Quantum Craps
layout: post
author: John
category: articles
tags: 
- projects
- science

---

It's said that god doesn't play dice with the universe, but what if she instead simultaneously rolls an infinite number of infinitely-sided dice and chooses the subset of dice that represent the outcome she prefers? I think most people would call this cheating; yet if she rolled only a single set of dice and got the same outcome by chance, it would instead be called luck. Is there any substantial difference between the two situations?

Let me put this philosophical discussion in context. Hypothetically, I have a relatively large number of biological samples (e.g. 1000). These samples are both quantity-limited and precious (i.e. I can't get any more if I run out). I want to run an assay on these samples. This assay gets run in batches of 92 samples. Typically, that assay requires a relatively large amount of input material, but my samples are so precious that I am not willing to use up the typical amount of material. Instead, I run a pilot project with a single batch of my samples that have large quantities of DNA available to determine that the assay generates good results with lower input (which it does). Before I have a chance to run the assay on the rest of my samples, a new and much improved version of the assay comes out. In the meantime, I have realized that my original experimental design was a poor one, because it confounds several covariates (individual and tissue type) with assay batch. I decide that I want to now run version 2 of the assay on all of my samples using a randomized design. However, version 2 of the assay is so new that the company I have contracted to run the assay has not tested it. They claim that since the chemistry and workflow are identical to version 1 of the assay, it should work identically well. And here is my skeptical face: 

![](https://freethinku.com/experience/wp-content/uploads/2013/09/skeptical-baby.jpg)

In any case, they're not willing to do any validation prior to running our samples. They are also not willing to allow us to run a smaller batch of samples (or, rather, they would, but would still charge us for a full batch). We could wait for someone else to be the Guinea pig, but that would unacceptably delay the timeframe of the experiment.

The solution I arrived at is as follows:

1. Randomize the samples a large number of times (100,000).
2. Chose the randomization that maximizes the number of samples from my original pilot experiment that I can run in a single batch.
3. Also use a control sample (a well-studied cell line with lots of existing data from version 1 of the assay). Add two technical replicates of the control to each batch, to allow me to regress out batch effects during data processing.
4. Run the assay on the first batch of samples, and wait for the data to perform validation against my pilot data.
5. Assuming everything goes well, run the assay on the remaining samples.

So, yes, I'm definitely cheating in steps 1-2. But is this form of cheating something that will compromise the integrity of my data? Is it something on which the reviewers of my paper or future grants will call me out? Should I feel bad about myself for doing this?
+++
author = "OpenGWAS"
title = "A new OpenGWAS: what is changing, and when"
date = "2026-11-27"
description = "We are rebuilding OpenGWAS. The new system holds more data, answers queries faster, is built on hg38, and is much more efficient to run. But it is not a drop-in replacement. Positions, variant IDs, LD reference data, dataset IDs and the R and Python packages all change, so some of your code will need updating, and your results will not always match what you got before."
tags = [
    "announcement",
    "roadmap",
]
+++

This post sets out what will change for you first, then why we are doing it and what it makes possible. The dates:

- From **14 December 2026** you can try the new system. Nothing changes unless you choose to.
- From 25 January 2027 a new TwoSampleMR release uses the new system by default.
- On 31 March 2027 the current API is switched off, and code that still calls it will stop returning data.

{{< og-timeline >}}

## What will change for existing scripts

If you use OpenGWAS through TwoSampleMR, ieugwasr or ieugwaspy, or a third-party package that uses the OpenGWAS API, these are likely **breaking changes**. Some will stop old code from running, which is easy to spot. Others will change your results without any error, and those need more care. In general, it must not be assumed that identical results will be obtained when running old scripts on the new system.

### Variants and positions

- **Variants are identified by ID, not position:** Every variant now has one ID, written `chr:pos:A1:A2`, and every effect size is given against `A1`. For SNPs, `A1` is the alphabetically first allele. Two datasets that share a variant share its ID without needing an external reference, and their effects already point the same way. You can still look variants up by rsID.
- **All positions change to hg38:** Everything is on GRCh38 (hg38). Studies reported on hg19 are lifted over when they are built, so any position you have saved from the current system will be different. Positions you query are exact, so there is no search radius to set.
- **Some variants drop out and others appear:** Every dataset goes through quality control and liftover. Variants that fail either are removed, and variants a study did not report can now be imputed (see below). The set of variants available for a given study will not be the same as before.
- **One rsID can return several rows:** Where an rsID matches more than one variant, you get all of them back. Code that assumes one row per SNP may count some SNPs twice.

### LD, proxies and clumping

- **LD estimates will differ:** The LD reference panels are much larger than before, and LD is estimated after adjusting for genetic principal components. Clumped results will differ, and so will any pre-clumped top hits.
- **Clumping defaults have changed:** The old default of r² = 0.001 within 10 Mb is refused. The new default is r² of around 0.03 within 10 Mb, so the number of instruments you get will change.
- **Clumping uses the study's own ancestry:** LD is taken from the ancestry assigned to each study, with no fallback to European LD. A study whose ancestry cannot be assigned confidently is marked unassigned.
- **Proxies are replaced by imputation:** Before, when an instrument was missing from your outcome GWAS, TwoSampleMR looked for a proxy SNP in LD with it while you waited. Now the genome-wide summary statistics for each study are imputed against a reference panel matched to its ancestry when the Store is built, so variants the study did not report have an estimated association. Imputed results are labelled, so you can keep them or leave them out, and your results will include a new column saying which is which. In one of our tests, all 100 BMI instruments were found in a coronary artery disease GWAS, 7 of them imputed. The accuracy of summary imputation is much better than using LD proxies. You can still ask for LD proxies if you want them.

### The data

- **Dataset IDs change:** Each analysis now has an ID made of its Store and the analysis, such as `OGS-00009:ukb-b-19953`. Old IDs are mapped to their new equivalents wherever possible, but not all of them will resolve. Check any script that hard-codes dataset IDs.
- **Some datasets are replaced or reorganised:** Where a source has published a newer release, we carry the new one and do not port the old one across. FinnGen moves to release 13, for example, and requests for the older FinnGen IDs point to the newer release with a warning. Some studies that used to be held as complete genome-wide results now sit in ragged Stores, which keep a smaller set of associations for each analysis, so some variants you could look up before will not be there.
- **Effect sizes are rescaled:** Every effect size now states its scale: per standard deviation for continuous traits, and log odds or log hazard ratio for binary and time-to-event outcomes. Continuous effects reported in natural units are converted to per-SD. Your code will still run, but MR estimates involving those traits will change size and units. `opengwasr` can convert back to the original units if you need them.
- **Metadata looks different:** Trait names, sample sizes, ancestry and licences come from the new catalogue. The ancestry shown is the one we assign from the data, which can differ from what the study reported. If you filter the list of datasets by these fields, you may select different studies.
- **GWAS-VCF files are no longer supported:** The new system does not use GWAS-VCF. Data is held in GWAS Stores, which you query through the API or, soon, download whole.

### Software

- **`opengwasr` and `opengwaspy` replace `ieugwasr` and `ieugwaspy`:** The new R and Python packages work the same way on data held by the API and on Stores downloaded to your own computer, and their syntax has changed to allow for that. The `opengwasr` [migration guide](https://blog.opengwas.io/opengwasr/articles/migrating-from-ieugwasr.html) lists each ieugwasr function and its replacement, and `opengwaspy` has a layer that works like ieugwaspy to ease the move. TwoSampleMR uses `opengwasr` under the hood from the new release. `opengwaspy` will also come with an MCP server, so AI assistants can query OpenGWAS on your behalf.
- **The API has changed:** If you call the OpenGWAS API directly, or maintain a package that does, its endpoints and what they return are different. The new API describes itself on its `/docs` and `/redoc` pages, for anyone building their own tools.
- **You need a new token:** Sign in with the same account and generate a new token for the new system.
- **Column names may differ:** The data frames returned by functions such as `extract_instruments()` and `extract_outcome_data()` may not have exactly the same columns as before.
- **Some old options may go:** Some ieugwasr-style arguments, such as `pop`, may be dropped or behave differently.
- **Allowances may change:** The allowance and the cost of each query on the new API may not match the current ones.
- **LD score regression comes later:** TwoSampleMR's LD score functions have not moved across yet.

### Bridging the gap

Where we can, we will close these gaps. Tell us during the alpha which differences cause you problems, and we will try to bridge them in TwoSampleMR, `opengwasr` and `opengwaspy`. But many of the old defaults existed because of limits in the old system: European LD for everyone, proxies instead of imputation, effect sizes in whatever units each study happened to use. Moving off them is likely to improve your analyses, so before forcing the old behaviour back, check whether the new one gives a better answer.

## Why we are rebuilding

The plainest reason is cost. The current system keeps every dataset several times over: once as a GWAS-VCF file, and again in each format the API needs to answer queries quickly. Hosting all of that costs more each year than we can sustain, and adding new data makes it worse. Left as it was, the service was at risk of being switched off.

That cost also kept the data still. Many datasets have not been updated for a long time, and the large new collections of summary statistics now being published would not have fit.

Underneath the cost were problems you will have run into yourselves:

- Everything is on hg19, and variants are found by position, which breaks down at multi-allelic sites and when assemblies differ.
- Looking up LD proxies for missing variants is slow, and it uses European LD whoever the study was in.
- Queries are slow, and pulling a large set of SNPs (for LD score regression, say) means downloading whole files and filtering them yourself.
- Large analyses on your own computer are hard, because there is no way to take the data with you in a form you can query.
- Datasets with thousands of molecular traits, such as eQTL and pQTL studies, have no good home.
- Some of the metadata cannot be trusted: effect sizes come on mixed scales, the reported ancestry is sometimes wrong, and trait ontology terms and licences are often missing.

We decided to fix these together rather than patch them one at a time, and that is why so much changes at once.

## How the new system fits together

{{< og-system >}}

Here is each part in turn.

We start from published summary statistics: the GWAS Catalog, UK Biobank, FinnGen, the Million Veteran Program (in several ancestries), eQTLGen, large metabolomics studies, and the datasets already in OpenGWAS.

`opengwasdb` is the open-source engine that turns those files into something you can query. It lifts every study to hg38, lines up alleles, fills in variants a study did not report, works out each study's ancestry from its allele frequencies, records the scale of every effect size, and runs checks before anything is published.

What it produces are **GWAS Stores**. A GWAS Store holds one collection of studies (all of FinnGen release 13, for example), harmonised the same way and published as a numbered release that never changes once it is out. Alongside them are LD Stores, which hold linkage disequilibrium for each reference panel and ancestry.

You can reach the data in two ways. Remotely, through the OpenGWAS API: TwoSampleMR, `opengwasr` (R) and `opengwaspy` (Python) all use it, and you can call it from your own code. Or locally: download whole Stores and query them on your own computer with `opengwasr`, `opengwaspy` or `ogquery`, using the same functions you would use against the API. [opengwas.io](https://opengwas.io) is the website for browsing and searching.

All of the software is open source under the GPL-3.0 licence.

## What the new system makes possible

### Efficiency, and room to grow

The new storage format is compact. For genome-wide studies, a GWAS Store is between seven and fifteen times smaller than the files it was built from, and each dataset is kept once, in one format, instead of several times over. That makes OpenGWAS much more efficient to run: the same funding goes further, and there is room for the new data coming in.

OpenGWAS remains free for non-commercial research. We will also offer a licence for commercial use, where each dataset's own terms allow it. That income goes towards keeping the service running.

### More reliable performance

When the platform gets busy in the current system, the API can start to return errors. We have implemented a more robust method of handling large amounts of queries. For the most part this doesn't change the way in which you perform queries, but it should be more resilient to high demand.

### Up-to-date data

Each source now has a repeatable pipeline. When the GWAS Catalog or FinnGen publish a new release, we build a new release of that Store, rather than curating it again by hand. The first collections cover the GWAS Catalog, UK Biobank, FinnGen release 13, the Million Veteran Program, eQTLGen and several metabolomics studies. Most of the datasets in the current system will be there on 14 December, and more will follow before 31 March.

### Thousands of molecular traits

A GWAS Store can be laid out in two ways, or a mix of both. In a dense Store, every analysis covers the same variants, which suits a biobank run of thousands of traits. In a ragged Store, each analysis keeps only its own associations, which suits QTL studies where each of thousands of genes has a different set of nearby variants. All 19,250 gene-expression analyses from eQTLGen fit in 2.6 GB, and looking up the variants near one gene takes about 50 milliseconds.

### Faster queries

Some measured times, on our server at the end of November 2026, before network time:

| Query | Time |
|---|---|
| One variant across all ~2,500 analyses in the UK Biobank Store (by rsID) | under 0.1 s |
| Top hits for one analysis | about 0.01 s |
| The ~1.2 million HapMap3 variants for one analysis | about 0.5 s |
| A 1 Mb region across every analysis in a Store | about 2 s |

Fetching the LD needed to clump a quarter of a million hits takes one to two seconds.

### Large SNP sets for LD score regression

The HapMap3 variants used by LD score regression are indexed in every densely laid-out Store, so extracting them for an analysis takes about half a second. You no longer have to download a whole file and filter it.

### Sample overlap, pre-calculated

When two GWAS share participants, their results are correlated even at variants that affect neither trait, and MR and other cross-trait methods need to allow for that. This correlation, rho, reflects both how much the samples overlap and how correlated the traits are. For every pair of analyses in a densely laid-out Store, we pre-calculate rho from variants with no detectable association in either study, so you can look it up instead of estimating it yourself.

### Analyses on your own computer

Whole GWAS Stores and LD Stores will be downloadable, one file per Store, each with a record of where it came from and the licences of its sources. Unpack one or more into a folder and point `opengwasr` (R), `opengwaspy` (Python) or `ogquery` (Python) at it, and the functions you use against the API run on your own machine instead. You can run a PheWAS across a whole Store, or extract genome-wide data for thousands of traits, without an allowance or a network connection.

### Effect sizes you can compare

Because every effect size states its scale, you can combine studies without guessing their units. Where a study reported continuous effects in natural units, we convert them to per-SD using the trait's standard deviation, taken from the study or estimated from the data, and record which method we used. We also check each study's stated scale against its allele frequencies, which catches mislabelled studies.

### Ancestry worked out from the data

We no longer rely on the ancestry a study reports. Each analysis's allele frequencies are compared with reference populations from around the world, and we assign the ancestry that fits. If the fit is not clear, the analysis is marked unassigned rather than guessed. That assigned ancestry is what picks the LD used for imputation and clumping.

## Still under way

Some of this work is not finished but is part of the roadmap:

- **Ontology mapping:** Traits are being mapped to ontology terms such as EFO. Studies from the GWAS Catalog come with them. For the rest, including the UK Biobank batch, mapping is in progress, and most analyses do not have a term yet.
- **Licences:** Every analysis has a licence field that links to the provider's terms, but many of these are still empty. We are working through them source by source. This matters for downloads and commercial use, which depend on it.
- **Binary traits in the UK Biobank batch:** Many of the `ukb-b` binary traits were analysed with linear models. They are labelled as such but have not been converted to log odds.
- **LD score regression in TwoSampleMR:** The fast HapMap3 extract is in place, but TwoSampleMR's LD score functions move over to the new system later.
- **Downloads:** The design is settled, and we will announce when downloads open.

## Trying the alpha

From 14 December 2026, anyone can try the new system.

1. Sign in with your existing OpenGWAS account and generate a new token for the new system (we will add the link here before 14 December).
2. Install the alpha release of TwoSampleMR in R:

   ```r
   remotes::install_github("MRCIEU/TwoSampleMR@0.8.0")
   ```

   In Python, install the alpha of `opengwaspy` (we will add instructions here before 14 December).

3. Run your usual analyses and tell us what breaks, what is slow and what looks wrong on [TwoSampleMR's GitHub issues](https://github.com/MRCIEU/TwoSampleMR/issues), or [opengwaspy's](https://github.com/opengwas/opengwaspy/issues) for Python.

The current system stays the default until 25 January, so the alpha will not change anything you do not ask it to. Things may change during testing. The most useful thing you can do is run an existing pipeline on both systems and compare the results: it shows you the differences that matter to your work before the switch, and tells us which gaps to bridge.

## After 31 March 2027

On 31 March the current OpenGWAS API is switched off for good. ieugwasr, ieugwaspy and older versions of TwoSampleMR call that API, so from then on they will stop returning data. If you have scripts or pipelines that use them, please move them to the new TwoSampleMR, `opengwasr` or `opengwaspy` before then. The new TwoSampleMR release will be out on 25 January, which leaves two months to switch.

GWAS-VCF files will stay downloadable for a limited time after the shutdown. We will give separate notice before they are removed.

After that the work carries on: new releases of each Store as sources publish new data, the ontology and licence work above, whole-Store downloads, and more hosts.

## Help us host OpenGWAS

At the moment OpenGWAS runs from one place (Bristol, UK), which makes it fragile and potentially slow for users in other parts of the world. The new format is small enough to copy whole, and every release is self-contained, so other institutions can host it too. We are looking for partners who can:

- host a full copy, so that OpenGWAS stays up if one site goes down;
- host datasets that cannot leave their own country or region, and serve them alongside the rest;
- contribute funding or compute.

If your institution could help, please email us at [ieu-opengwas@bristol.ac.uk](mailto:ieu-opengwas@bristol.ac.uk).

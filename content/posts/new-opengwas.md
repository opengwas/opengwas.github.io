+++
author = "OpenGWAS"
title = "A new OpenGWAS: what is changing, and when"
date = "2026-11-30"
description = "We are replacing the system behind OpenGWAS. The new one opens for testing on 14 December 2026, becomes the default on 25 January 2027, and the current system is retired on 31 March 2027. This post explains why, what is different, and what you need to do."
tags = [
    "announcement",
    "roadmap",
]
+++

We are replacing the system that runs OpenGWAS. The new one holds more data, answers queries faster, is built on hg38, and costs far less to run. It opens for testing in two weeks, and the current system will be switched off at the end of March.

If you use OpenGWAS through TwoSampleMR or ieugwasr, here is what that means for you:

- From **14 December 2026** you can try the new system. Nothing changes unless you choose to.
- From 25 January 2027 a new TwoSampleMR release uses the new system by default.
- On 31 March 2027 the current API is switched off, and code that still calls it will stop returning data.

{{< og-timeline >}}

Below we explain why we are doing this, how the new system fits together, what has changed for each problem people have raised with us, what is unfinished, and how you can help.

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

We decided to fix these together rather than patch them one at a time.

## How the new system fits together

{{< og-system >}}

Here is each part in turn.

We start from published summary statistics: the GWAS Catalog, UK Biobank, FinnGen, the Million Veteran Program (in several ancestries), eQTLGen, large metabolomics studies, and the datasets already in OpenGWAS.

`opengwasdb` is the open-source engine that turns those files into something you can query. It lifts every study to hg38, lines up alleles, fills in variants a study did not report, works out each study's ancestry from its allele frequencies, records the scale of every effect size, and runs checks before anything is published.

What it produces are **GWAS Stores**. A GWAS Store holds one collection of studies (all of FinnGen release 13, for example), harmonised the same way and published as a numbered release that never changes once it is out. Each analysis in it has an ID made of the Store and the analysis, such as `OGS-00009:ukb-b-19953`. Alongside them are LD Stores, which hold linkage disequilibrium for each reference panel and ancestry.

You reach the data in three ways. The OpenGWAS API is what TwoSampleMR, `opengwasr` (R) and `opengwaspy` (Python) talk to. [opengwas.io](https://opengwas.io) is the website for browsing and searching. And you will be able to download whole Stores and query them on your own computer with the same functions.

All of the software is open source under the GPL-3.0 licence.

## What has changed

### Cost, and room to grow

The new storage format is compact. For genome-wide studies, a GWAS Store is between seven and fifteen times smaller than the files it was built from, and each dataset is kept once, in one format, instead of several times over. That is what brings the cost of running OpenGWAS down to something we can sustain, and it leaves room for the new data coming in.

OpenGWAS remains free for non-commercial research. We will also offer a licence for commercial use, where each dataset's own terms allow it. That income goes towards keeping the service running.

### More reliable performance

When the platform gets busy in the current system, the API can start to return errors. We have implemented a more robust method of handling large amounts of queries. For the most part this doesn't change the way in which you perform queries, but it should be more resilient to high demand.

### Up-to-date data

Each source now has a repeatable pipeline. When the GWAS Catalog or FinnGen publish a new release, we build a new release of that Store, rather than curating it again by hand. The first collections cover the GWAS Catalog, UK Biobank, FinnGen release 13, the Million Veteran Program, eQTLGen and several metabolomics studies. Most of the datasets in the current system will be there on 14 December, and more will follow before 31 March.

### Thousands of molecular traits

A GWAS Store can be laid out in two ways, or a mix of both. In a dense Store, every analysis covers the same variants, which suits a biobank run of thousands of traits. In a ragged Store, each analysis keeps only its own associations, which suits QTL studies where each of thousands of genes has a different set of nearby variants. All 19,250 gene-expression analyses from eQTLGen fit in 2.6 GB, and looking up the variants near one gene takes about 50 milliseconds.

### hg38

Everything is on GRCh38 (hg38). Studies reported on hg19 are lifted over when their Store is built, and the build stops if more than 1% of variants fail to lift. Positions you query are exact, so there is no search radius to set.

### Variant IDs instead of positions

Every variant has one ID, written `chr:pos:A1:A2` on hg38, and every effect size is given against `A1`, which is chosen to be the alphabetically first allele. Two datasets that share a variant share its ID automatically without needing an external reference, and their effects already point the same way, so you do not have to flip alleles when you combine them. You can still look variants up by rsID. Where one rsID matches more than one variant, you get all of them back and can choose.

### Missing variants and LD proxies

Before, when an instrument was missing from your outcome GWAS, TwoSampleMR looked for a proxy SNP in LD with it while you waited. Now that work happens once, when a Store is built. The genome-wide summary statistics for each study are imputed against a reference panel matched to its ancestry, and variants it did not report before now will have an estimated association. Imputed results are labelled as imputed, so you can keep them or leave them out. In one of our tests, all 100 BMI instruments were found in a coronary artery disease GWAS, 7 of them imputed. The accuracy of summary imputation is much better than using LD proxies.

Clumping uses LD from the study's own ancestry. Fetching the LD needed to clump a quarter of a million hits takes one to two seconds. You can still ask for LD proxies if you want them.

### Faster queries

Some measured times, on our server at the end of November 2026, before network time:

| Query | Time |
|---|---|
| One variant across all ~2,500 analyses in the UK Biobank Store (by rsID) | under 0.1 s |
| Top hits for one analysis | about 0.01 s |
| The ~1.2 million HapMap3 variants for one analysis | about 0.5 s |
| A 1 Mb region across every analysis in a Store | about 2 s |

### Large SNP sets for LD score regression

The HapMap3 variants used by LD score regression are indexed in every densely laid-out Store, so extracting them for an analysis takes about half a second (see the table above). You no longer have to download a whole file and filter it.

### Analyses on your own computer

Whole GWAS Stores and LD Stores will be downloadable, one file per Store, each with a record of where it came from and the licences of its sources. Unpack one or more into a folder and point `opengwasr` (R) or `ogquery` (Python) at it, and the functions you use against the API run on your own machine instead. You can run a PheWAS across a whole Store, or extract genome-wide data for thousands of traits, without an allowance or a network connection.

### Effect sizes on a known scale

Every effect size now says what scale it is on: per standard deviation for continuous traits, and log odds or log hazard ratio for binary and time-to-event outcomes. Where a study reported continuous effects in natural units, we convert them to per-SD using the trait's standard deviation, taken from the study or estimated from the data, and record which method we used. `opengwasr` can convert back to the original units. We also check each study's stated scale against its allele frequencies, which catches mislabelled studies.

### Ancestry worked out from the data

We no longer rely on the ancestry a study reports. Each analysis's allele frequencies are compared with reference populations from around the world, and we assign the ancestry that fits. If the fit is not clear, the analysis is marked unassigned rather than guessed. That assigned ancestry is what picks the LD used for imputation and clumping.

## Still under way

Some of this work is not finished but it part of the roadmap:

- **Ontology mapping:** Traits are being mapped to ontology terms such as EFO. Studies from the GWAS Catalog come with them. For the rest, including the UK Biobank batch, mapping is in progress, and most analyses do not have a term yet.
- **Licences:** Every analysis has a licence field that links to the provider's terms, but many of these are still empty. We are working through them source by source. This matters for downloads and commercial use, which depend on it.
- **Binary traits in the UK Biobank batch:** Many of the `ukb-b` binary traits were analysed with linear models. They are labelled as such but have not been converted to log odds.
- **LD score regression in TwoSampleMR:** The fast HapMap3 extract is in place, but TwoSampleMR's LD score functions move over to the new system later.
- **Downloads:** The design is settled, and we will announce when downloads open.
- **Old dataset IDs:** IDs from the current system, such as `ukb-b-19953`, are mapped to their new equivalents wherever possible.

## Trying the alpha

From 14 December 2026, anyone can try the new system.

1. Sign in with your existing OpenGWAS account and generate a new token for the new system at (coming soon).
2. Install the alpha release of TwoSampleMR:

   ```r
   remotes::install_github("MRCIEU/TwoSampleMR@0.8.0")
   ```

3. Run your usual analyses and tell us what breaks, what is slow and what looks wrong on [TwoSampleMR's GitHub issues](https://github.com/MRCIEU/TwoSampleMR/issues).

The current system stays the default until 25 January, so the alpha will not change anything you do not ask it to. Things may change during testing.

## After 31 March 2027

On 31 March the current OpenGWAS API is switched off for good. ieugwasr and older versions of TwoSampleMR call that API, so from then on they will stop returning data. If you have scripts or pipelines that use them, please move them over before then. The new TwoSampleMR release will be out on 25 January, which leaves two months to switch.

GWAS-VCF files will stay downloadable for a limited time after the shutdown. We will give separate notice before they are removed.

After that the work carries on: new releases of each Store as sources publish new data, the ontology and licence work above, whole-Store downloads, and more hosts.

## Help us host OpenGWAS

At the moment OpenGWAS runs from one place (Bristol, UK), which makes it fragile and potentially slow for users in other parts of the world. The new format is small enough to copy whole, and every release is self-contained, so other institutions can host it too. We are looking for partners who can:

- host a full copy, so that OpenGWAS stays up if one site goes down;
- host datasets that cannot leave their own country or region, and serve them alongside the rest;
- contribute funding or compute.

If your institution could help, please email us at [ieu-opengwas@bristol.ac.uk](mailto:ieu-opengwas@bristol.ac.uk).

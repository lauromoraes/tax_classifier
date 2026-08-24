# SILVA 138.2 V3--V4 QIIME 2 Classifier

A small, reproducible workflow for preparing a **SILVA 138.2 SSU Ref
NR99** taxonomic classifier for the **V3--V4 region of the 16S rRNA
gene** using **QIIME 2**, **RESCRIPt**, and `q2-feature-classifier`.

The workflow is implemented as a Jupyter notebook and was designed for
amplicons generated with the **341F/806R** primer pair.

## Overview

Taxonomic classification of 16S rRNA amplicons depends not only on the
reference database but also on the region of the marker gene targeted by
sequencing. This project prepares a region-specific Naive Bayes
classifier by extracting the expected V3--V4 amplicon from SILVA
reference sequences before classifier training.

The workflow performs the following steps:

1.  imports SILVA 138.2 SSU Ref NR99 reference sequences;
2.  converts SILVA RNA reference sequences to DNA;
3.  imports the SILVA taxonomy map, rank definitions, and taxonomy tree;
4.  constructs fixed-rank taxonomy with RESCRIPt;
5.  extracts the V3--V4 region in silico using the 341F/806R primers;
6.  trains a Naive Bayes taxonomic classifier with
    `q2-feature-classifier`;
7.  classifies the extracted reference sequences as an internal
    consistency check;
8.  evaluates expected versus observed taxonomy with RESCRIPt.

Two classifier variants can be generated:

-   a classifier using taxonomy through the **genus** rank;
-   an exploratory classifier including **species labels**.

## Target Region and Primers

The workflow targets the V3--V4 region of the 16S rRNA gene.

  Parameter                  Value
  -------------------------- ------------------------
  Marker                     16S rRNA
  Region                     V3--V4
  Forward primer             341F
  Forward sequence           `CCTACGGGRSGCAGCAG`
  Reverse primer             806R
  Reverse sequence           `GGACTACHVGGGTWTCTAAT`
  Minimum extracted length   350 bp
  Maximum extracted length   550 bp
  Reference database         SILVA 138.2
  Reference collection       SSU Ref NR99

The primer sequences and extraction limits are parameters in the
notebook and can be modified for other experimental designs.

## Requirements

The workflow requires a QIIME 2 environment containing the following
components:

-   QIIME 2;
-   `q2-feature-classifier`;
-   RESCRIPt;
-   scikit-learn;
-   Joblib;
-   Jupyter Notebook or JupyterLab.

The exact versions should be recorded when building a classifier.
`TaxonomicClassifier` artifacts contain serialized scikit-learn objects,
so classifiers should be used with a compatible software environment.

## Reference Data

The workflow uses **SILVA 138.2 SSU Ref NR99**.

The sequence reference file is expected to be:

``` text
SILVA_138.2_SSURef_NR99_tax_silva.fasta.gz
```

The SILVA taxonomy resources used by RESCRIPt are:

``` text
taxmap_slv_ssu_ref_nr_138.2.txt.gz
tax_slv_ssu_138.2.txt.gz
tax_slv_ssu_138.2.tre.gz
```

Reference data can be obtained automatically with RESCRIPt when network
access is reliable, or downloaded separately and placed in the project
directory.

## Configuration

Machine-specific settings are intentionally separated from scientific
project parameters.

At the beginning of the notebook, configure the project and temporary
directories:

``` python
PROJECT_DIR = Path("/path/to/silva-138.2-dev").resolve()
TEMP_DIR = Path("/path/to/large/tmp").resolve()

JOBLIB_TEMP_DIR = TEMP_DIR / "joblib"
QIIME_CACHE = TEMP_DIR / "qiime2-cache"
```

`PROJECT_DIR` contains the reference files and generated artifacts.

`TEMP_DIR` should point to a filesystem with sufficient free space.
Building and using large SILVA classifiers can require substantial
temporary storage.

The notebook redirects the relevant temporary locations:

``` python
os.environ["TMPDIR"] = str(TEMP_DIR)
os.environ["TMP"] = str(TEMP_DIR)
os.environ["TEMP"] = str(TEMP_DIR)
os.environ["JOBLIB_TEMP_FOLDER"] = str(JOBLIB_TEMP_DIR)
```

This is particularly important for `classify-sklearn`, because Joblib
may create large memory-mapped temporary files when multiple workers are
used.

## Usage

Activate the QIIME 2 environment and start Jupyter:

``` bash
conda activate <qiime2-environment>
jupyter lab
```

Open the classifier preparation notebook and configure `PROJECT_DIR` and
`TEMP_DIR` before running the workflow.

The notebook deliberately keeps the QIIME 2 commands explicit, for
example:

``` bash
qiime feature-classifier fit-classifier-naive-bayes \
    --i-reference-reads silva-138.2-v3v4-341f-806r-seqs.qza \
    --i-reference-taxonomy silva-138.2-ssu-nr99-tax.qza \
    --o-classifier silva-138.2-v3v4-341f-806r-nb-classifier.qza \
    --verbose
```

This makes each processing step directly inspectable and simplifies
adaptation of the commands for command-line execution.

## Main Outputs

The principal outputs include:

``` text
silva-138.2-v3v4-341f-806r-seqs.qza
silva-138.2-v3v4-341f-806r-stats.qza

silva-138.2-ssu-nr99-tax.qza
silva-138.2-ssu-nr99-species-tax.qza

silva-138.2-v3v4-341f-806r-nb-classifier.qza
silva-138.2-v3v4-341f-806r-nb-species-classifier.qza

silva-138.2-v3v4-341f-806r-nb-observed-taxonomy.qza
silva-138.2-v3v4-341f-806r-nb-species-observed-taxonomy.qza

silva-138.2-v3v4-341f-806r-nb-evaluation.qzv
silva-138.2-v3v4-341f-806r-nb-species-evaluation.qzv
```

The `.qzv` files can be inspected with QIIME 2 View.

## Evaluation

The notebook applies the trained classifier to the same V3--V4 SILVA
reference sequences used during training and compares the predicted
taxonomy with the expected taxonomy using RESCRIPt.

This procedure is intended as an **internal consistency check**. It can
help identify problems involving reference preparation, taxonomy
alignment, classifier construction, serialization, or prediction.

It is **not an independent validation of classifier accuracy**, because
the evaluated sequences are also part of the training data. Performance
obtained from this step should therefore not be interpreted as an
unbiased estimate of generalization to experimental sequences.

Independent reference sequences, cross-validation, or an appropriate
hold-out design would be required for a formal assessment of predictive
performance.

## Species-Level Classification

The repository includes an optional workflow that retains SILVA species
labels when constructing the taxonomy.

Species-level results should be interpreted cautiously. Resolution of a
short 16S amplicon may be insufficient to distinguish closely related
species, and the reliability and completeness of reference species
labels may also limit classification.

For this reason, the genus-level classifier should generally be
considered the primary classifier unless species-level assignments are
independently supported by the study design and validation strategy.

## Temporary Storage and Parallelization

Large classifiers can consume substantial temporary disk space during
both training and classification.

Before running computationally intensive steps, verify the available
storage:

``` bash
df -h
```

For `classify-sklearn`, the number of parallel workers is controlled by:

``` text
--p-n-jobs
```

If multiprocessing produces errors involving Joblib, Loky, memory
mapping, pickling, or:

``` text
OSError: [Errno 28] No space left on device
```

verify that both `TMPDIR` and `JOBLIB_TEMP_FOLDER` point to a filesystem
with sufficient free space.

Reducing the number of workers can also be useful for diagnosis.

## Reproducibility

When distributing a generated classifier or reporting its use in a
publication, record at least:

-   SILVA release;
-   SILVA reference collection;
-   target 16S region;
-   primer sequences;
-   amplicon extraction limits;
-   QIIME 2 version;
-   RESCRIPt version;
-   `q2-feature-classifier` version;
-   scikit-learn version;
-   taxonomy configuration, including whether species labels were
    retained.

QIIME 2 artifacts contain provenance information internally, but
preserving the notebook and computational environment provides an
additional human-readable record of the workflow.

## References

Bokulich NA, Kaehler BD, Rideout JR, et al. Optimizing taxonomic
classification of marker-gene amplicon sequences with QIIME 2's
q2-feature-classifier plugin. *Microbiome*. 2018;6:90.
https://doi.org/10.1186/s40168-018-0470-z

Quast C, Pruesse E, Yilmaz P, et al. The SILVA ribosomal RNA gene
database project: improved data processing and web-based tools. *Nucleic
Acids Research*. 2013;41:D590--D596. https://doi.org/10.1093/nar/gks1219

Robeson MS II, O'Rourke DR, Kaehler BD, et al. RESCRIPt: Reproducible
sequence taxonomy reference database management. *PLoS Computational
Biology*. 2021;17:e1009581. https://doi.org/10.1371/journal.pcbi.1009581

Yilmaz P, Parfrey LW, Yarza P, et al. The SILVA and "All-species Living
Tree Project (LTP)" taxonomic frameworks. *Nucleic Acids Research*.
2014;42:D643--D648. https://doi.org/10.1093/nar/gkt1209

## License

No license is specified by this README. Add a `LICENSE` file appropriate
for the intended distribution of the repository and verify the terms
applicable to any redistributed SILVA data or derived resources.


## Comprehensive analysis of mutational signatures in pediatric cancers

Please refer to [Comprehensive analysis of mutational signatures reveals distinct patterns and molecular processes across 27 pediatric cancers](https://www.nature.com/articles/s43018-022-00509-4) by Thatikonda et al, 2023 for additional details.

Following guide is a step by step explanation of mutational signature extraction with SigProfiler, SignatureAnalyzer and HDP. 

### SigProfiler

Following steps describe the step by step extraction of mutational signatures using SigProfiler method with scripts under SigProfiler folder

#### Step-1: Mutational catalogue generation from VCF files

Generates a matrix of mutational catalogues from a set of VCF files

```
~/anaconda3/bin/python matrix_generator-vt10112020.py --help

Usage: matrix_generator-vt10112020.py [OPTIONS]

Options:
  --project_name TEXT  name for the output matrix files  [required]
  --input_vcf PATH     input path of vcf files  [required]
  --plot               add --plot if the per-sample profile plots are needed
  --seq_info           add --seq_info to write each mutations context into a
                       file
  --exome              add --exome if the mutational catalogues are from exome
                       sequencing
  --help               Show this message and exit.

```

#### Step-2: Extraction of de novo signatures,  decomposition into COSMIC v3 signatures and attribution

Extracts de novo signatures and decomposes them to COSMIC reference signatures. Finally attributes mutations from each tumor sample to each detected signature.

```
~/anaconda3/bin/python extractor_v18_vt22042020-noplots.py --help

Usage: extractor_v18_vt22042020-noplots.py [OPTIONS]

Options:
  --input_catalogue PATH  input mutational catalogue  [required]
  --output TEXT           output path to store results  [required]
  --min_sig INTEGER       minimum number of signatures to extract  [required]
  --max_sig INTEGER       maximum number of signatures to extract  [required]
  --max_iter INTEGER      maximum number of iterations to perform  [required]
  --cat_channel TEXT      one of 96, 1536, DINUC, ID - depending on the
                          mutational catalogue  [required]
  --exome                 add --exome if the mutational catalogues are from
                          exome sequencing  [required]
  --help                  Show this message and exit.
```

#### Step-3: Custom decomposition of de novo signatures & attribution

Computational selection of optimal de novo signatures by the method is not optimal. Therefore, visual inspection of the `K selection` plots by above script is necessary. Once optimal `K` is selected based on `mean cosine distance` and `average stability` for different `K`s, following script can be used to decompose `K` de novo signatures into COSMIC signatures using non-negative lease squares (NNLS)

```
~/anaconda3/bin/python SigProfiler_Decomposition_nnls_v02.py --help

Usage: SigProfiler_Decomposition_nnls_v02.py [OPTIONS]

Options:
  --input_catalogue PATH       input mutational catalogue  [required]
  --denovo_signatures PATH     denovo signatures file  [required]
  --reference_signatures PATH  COSMIC reference signatures  [required]
  --output TEXT                output path to store results  [required]
  --help                       Show this message and exit.
```

NOTE:


1. This script can be used with de novo signatures generated by any method to decompose them into COSMIC signatures.

2. It is always a good idea to decompose +/-1 de novo signatures from optimal `K` and further check biological relevance of results.

3. This script also attributes mutations from each tumor sample to decomposed signatures.

#### Step-4: Supervised attribution of mutations to known signatures

If a set of signatures operating in a given cancer type are known, following script can be used to attribute mutations from each tumor sample to each of these known signature

```
~/anaconda3/bin/python SP_SupervisedAttributions.py --help

Usage: SP_SupervisedAttributions.py [OPTIONS]

Options:
  --input_catalogue PATH       input mutational catalogue  [required]
  --output TEXT                output path to store results  [required]
  --reference_signatures PATH  SBS/ID reference signatures  [required]
  --help                       Show this message and exit.
```

### SigAnalyzer

Mutational catalogues generated by in above SigProfiler - Step-1 can also be used with SignatureAnalyzer. SigAnalyzer method has been suggested to use with COMPOSITE mutational catelogues. In the paper, we have used the combination of SBS1536 + ID83 catalogues as COMPOSITE mutational catalogue as input.

#### Step-1: COMPOSITE signature extraction

```
signatureanalyzer SBS1536_ID83_mutCatalogue.txt --type spectra --nruns 10 --outdir SBS1536_ID83_Signatures --reference pcawg_SBS_ID --verbose --max_iter 2000000 --objective poisson --random_seed 42
```

SigAnalyzer writes results into HDF5 format. Functions provided in `utils` folder can be used to extract required results from these HDF5 files. This function is described in the last section of this tutorial.

#### Step-2: Decomposition of COMPOSITE signatures to COSMIC SBS96 signatures

de novo signatures are often times a mixture of one or multiple known COSMIC signatures. Therefore, it is always a good idea to decompose each de novo signature.

After extracting SBS-1536 probabilities for each signatures from HDF5 files using functions in `utils` folder, collapse them to SBS96 profiles by summing tri-nulceotide probabilities for each signature. The resulting SBS96 de novo signatures can be decomposed with the script SigProfiler Step-3 script.


#### Step-3: Attribution of mutations with ARD-NMF method

The above step can also attributes mutations in each tumor sample to each decomposed COSMIC signature by default. However, if you want to perform attribution with ARD-NMF using mutational catalogue and above decomposed COSMIC signatures, following script can be used

```
python SA_SupervisedARDNMF.py --help

Usage: SA_SupervisedARDNMF.py [OPTIONS]

Options:
  --input_catalogue PATH  input mutational catalogue, SA compatible
                          [required]

  --req_cosmic TEXT       required COSMIC signatures for supervised ARD-NMF
                          [required]

  --ref_signatures PATH   COSMIC reference signatures, either SBS96 or ID83
                          [required]

  --max_iter INTEGER      maximum number of iteration to run ARD-NMF
                          [required]

  --context_type TEXT     context type e.g. 1536, 96, 83  [required]
  --output TEXT           output path to store results  [required]
  --help                  Show this message and exit.
```

SigAnalyzer is known to often overfit and attribute a small number of mutations to non-existing signatures in a tumor sample. Therefore, we applied cut-off that a signature is present in a sample if it contributes to >=5% of overall mutations.

### HDP (Heirarchical Dirichlet Process)

Following function in `utils/_fns.R` can be used to run HDP method with priors. HDP can attribute mutations to given priors in each sample and at the same time simultaneously discovers novel signatures, if there are any.

```
# df_spectra - data frame of mutational catalogue
# sp_ref - Reference signatures from COSMIC
# priors - prior signatures to include, e.g. SBS1, SBS5 can be included by default. In addition to biologically relevant signatures dependencing on the cancer type

run_hdp_supervised(df_spectra, sp_ref, priors)
```

### PCAWG style signature overview visualization

In the paper, we visualized the signature overviews as in PCAWG mutational signatures manuscript. Following functions are useful to compute median exposures/megabase, fraction of tumor samples within each cohort with signature.


Following function compute required medians and fractions
```
# df = input data frame with following SBS or ID columns
# PID CANCER_TYPE SBS1 SBS2..
# context = either SBS or ID

process_exposure_table(df, context = 'SBS')

```

Following function formats the output from above function and prepares for plotting

```
# df = output from function `process_exposure_table()`

fmt_sigtable_forplotting(df)
```

Finally, following function can be used to make the overview figure

```
# df = result from function `fmt_sigtable_forplotting()`

# cl_limits = numerical limits for color legend, eg. c(0, 1)

# cl_breaks = breaks on color legend for color variance to differentiate signature exposures, e.g. c(0, 0.001, 0.002, 0.003, 0.004)

plot_signature_overview(df, cl_limits, cl_breaks)
```

### Other useful functions used in the paper

Format & plot different variants of SBS-96 and ID-83 profiles.



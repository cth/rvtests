## Contents

-   [Introduction](#Introduction)
-   [Quick Tutorial](#Quick_Tutorial)
-   -   [Single variant test](#Single_variant_test)
    -   [Group test](#Group_test)
    -   [Options in association tests](#Options_in_association_tests)
    -   [Phenotype file](#Phenotype_file)
    -   [Covariate file](#Covariate_file)
    -   [Use gene/set based rare-variant tests](#Use_gene.2Fset_based_rare-variant_tests)

-   [Contact](#Contact)


* * * * *

# Introduction

Rvtests, which stands for Rare Variant tests, is a flexible software package for genetic association studies. It is designed to support unrealted individual or related (family-based) individuals. Both quantitative trait and binary trait are supported. It includes a variety of association tests (e.g. single variant score test, burden test, variable threshold test, SKAT test, fast linear mixed model score test). It takes [VCF][vcf] format as genotype input file and takes PLINK format phenotype file and covariate file. From our practice, it is capable to analyze 8,000 related individuals using less than 400 Mb memory. 

[vcf]: http://www.1000genomes.com/

# Quick Tutorial

Here is a quick example of how to use *rvtests* software in typical use cases.

## Single variant tests

    rvtests --inVcf input.vcf --pheno phenotype.ped --out output --single wald,score

This specifies single variant Wald and score test for association
tests for every variant in the `input.vcf` file. The 6th column of the phenotype file, `phenotype.ped`, which is in PLINK format, is used. Rvtests will automatically check whether the phenotype is binary trait or quantitative trait.

For other types of association tests, you can refer to [Models](#Models)

## Groupwise tests
Groupwise tests includes three major kinds of tests.

* Burden tests: group variants, which are usually less than 1% or 5% rare variants, for association tests. The category includes: CMC test, Zeggini test, Madsen-Browning test, CMAT test, and rare-cover test.
* Variable threshold tests: group variants under different frequency thresholds.
* Kernel methods: suitable to tests rare variants having different directions of effects. These includes SKAT test and KBAC test. 

All above tests requires to group variants into a unit. The simplist case is to use gene as grouping unit. For different grouping method, see [Grouping](#Grouping). 

To perform rare variant tests by gene, you need to use `--geneFile` to specify the gene range in a refFlat format. We provided different gene definitions in the [Resources](#Resources) section. You can use `--gene` to specify which gene(s) to test. For example, specify `--gene CFH,ARMS2` will perform association tests on CFH and ARMS2 genes. If there is no providing `--gene` option, all genes will be tests.

The following command line demonstrate how to use CMC method, variable threshold method(proposed by Price) and kernel based method (SKAT by Shawn Lee and KBAC by
Dajiang Liu) to test every gene listed in *refFlat\_hg19\_uniq\_gene.txt.gz*.

    rvtests --inVcf input.vcf --pheno phenotype.ped --out output --geneFile refFlat_hg19_uniq_gene.txt.gz --burden cmc --vt price --kernel skat,kbac


## Related individual tests

To test related individuals, you will need to first create a kinship matrix:

    vcf2kinship --inVcf input.vcf --bn --out output

The option `--bn` means calculating empirical kinship using Balding-Nicols method. You can specifiy `--ibs` to obtain IBS kinship or use `--pedigree input.ped` to calculate kinship from known pedigree information.

Then you can use linear mixed model based association tests such as Fast-LMM score test, Fast-LMM LRT test and Grammar-gamma tests. An exemplar command is shown: 

    rvtests --inVcf input.vcf --pheno phenotype.ped --out output --kinship output.kinship --model famScore,famLRT,famGrammarGamma

## Meta-analysis tests

The meta-analysis models outputs association test results and genotype covariance matrix. These statistics can be used in rare variant association analysis.
We provide single variant score test and generate genotype covariance matrix. 
You can use command:
   
    rvtests --inVcf input.vcf --pheno phenotype.ped --covar example.covar --covar-name age,bmi --inverseNormal --useResidualAsPhenotype  --meta score,cov --out output

Here the `--covar` specify a covariate file, and `--covar-name` specify which covariates can used in the analysis. Covariate file format can be found [here](#Covariate file). `--inverseNormal --useResidualAsPhenotype` specifies trait transformation method. That means first fit a regression model of the phenotype on covariates (intercept automatically added), then the residuals are inverse normalized. Trait transformation details can be found [here](#Trait transformation).

# Input files

## Genotype file (VCF)

Rvtests supports VCF (Variant Call Format) files. Files in both plain txt format or gzipped format are supported. To use group-based rare variant tests, indexed the VCF files using [tabix](http://samtools.sourceforge.net/tabix.shtml) are required. 

Here are the commands to convert plain text format to bgzipped VCF format:

    (grep ^"#" $your_old_vcf; grep -v ^"#" $your_old_vcf | sed 's:^chr::ig' | sort -k1,1n -k2,2n) | bgzip -c > $your_vcf_file 
    tabix -f -p vcf $your_vcf_file
## Phenotype file

You can use `--mpheno $phenoypeColumnNumber` or `--pheno-name` to specify a given phenotype.

You can use `--covar` and `--covar-name` to specify covariates that will be used for single variant association analysis. This is an optional parameter. If you do not have covariate in the data, this option can be ignored. 

In this meta-analysis, we use inversed normal transformed residuals in the association analysis, which is achieved by using a combination of `--inverseNormal`  and `--useResidualAsPhenotype`. Specifically, we first fit the null model by regressing phenotype on covariates. The residuals are then inverse normal transformed (see Appendix A more detailed formulae for transformation). Transformed residuals will be used to obtain score statistics. 

In meta analysis, an exemplar command for using rvtest looks like the following:

# Models
	
Rvtests support various association models.

## Single variant tests

Single variant | Model(*)    |Traits(#) | Covariates | Related / unrelated | Description
:--------------|:---------:|:------:|:----------:|:-------------------:|:-----------
Score test     |  score    |B, Q  |     Y      |         U           | Only null model is used to performed the test
Wald  test     |  wald     |B, Q  |     Y      |         U           | Only fit alternative model, and effect size will be estimated
Exact test     |  exact    |B     |     N      |         U           | Fisher's test
Fam LRT        |  famLRT   |Q     |     Y      |         R, U        | Fast-LMM model
Fam Score      |  famScore |Q     |     Y      |         R, U        | Fast-LMM model style likelihood ratio test
Grammar-gamma  |famGrammarGamma| Q     |     Y      |         R, U        | Grammar-gamma method


(*) Model columns list the regconized names in rvtests. For example, use `--single score` will apply score test.

(#) In trait column, B and Q stand for binary, quantitiave trait.


## Burden tests

Burden tests | Model(*)    |Traits(#) | Covariates | Related / unrelated | Description
:--------------|:---------:|:------:|:----------:|:-------------------:|:-----------
CMC             |  cmc       |B, Q  |     N      |         U           | Collapsing and combine rare variants by Bingshan Li.
Zeggini         |  zeggini   |B, Q  |     N      |         U           | Aggregate counts of rare variants by Morris Zeggini.
Madsen-Browning |  mb        |B     |     N      |         U           | Up-weight rare variant using inverse frequency from controls by Madsen.
Fp              |  fp        |B     |     N      |         U           | Up-weight rare variant using inverse frequency from controls by Danyu Lin.
Exact CMC       |  exactCMC  |B     |     N      |         U           | Collapsing and combine rare variants, then pefore Fisher's exact test.
RareCover       |  rarecover |B     |     N      |         U           | Find optimal grouping unit for rare variant tests by Thomas Hoffman.
CMAT            |  cmat      |B     |     N      |         U           | Test non-coding variants by Matt Z.
CMC Wald        |  cmcWald   |B, Q  |     N      |         U           | Collapsing and combine rare variants, then pefore Wald test.


(*) Model columns list the regconized names in rvtests. For example, use `--burden cmc` will apply CMC test.

(#) In trait column, B and Q stand for binary, quantitiave trait.


## Variable threshold models

Single variant | Model(*)    |Traits(#) | Covariates | Related / unrelated | Description
:--------------|:---------:|:------:|:----------:|:-------------------:|:-----------
Variable threshold model     |  vt    |B, Q  |     N      |         U           | Every rare-variant frequency cutoffs are tests by Alkes Price.  
Variable threshold CMC     |  cmc     |B, Q  |     N      |         U           | This models is natiive so that it output CMC test statistics under all possible frequency cutoffs.

(*) Model columns list the regconized names in rvtests. For example, use `--vt price` will apply score test.

(#) In trait column, B and Q stand for binary, quantitiave trait.



## Kernel models

Kernel | Model(*)    |Traits(#) | Covariates | Related / unrelated | Description
:--------------|:---------:|:------:|:----------:|:-------------------:|:-----------
SKAT     |  skat    |B, Q  |     Y      |         U           | Sequencing kernel association test by Shawn Lee.
KBAC     |  kbac     |B  |     N      |         U           | Kernel-based adaptive clustering model by Dajiang Liu.


(*) Model columns list the regconized names in rvtests. For example, use `--kernel skat` will apply SKAT test.

(#) In trait column, B and Q stand for binary, quantitiave trait.


## Utility models


Rvtests has an usually option `--outputRaw`. When specify this, rvtests can output genotypes, phenotype, covariates(if any) and collapsed genotype to tabular files. These files can be imported into other software (e.g. R) for further analysis.


# Association test options

## Sample inclusion/exclusion

Rvtests can flexibly specify which sample(s) to include or exclude:

           --peopleIncludeID : give IDs of people that will be included in study
         --peopleIncludeFile : from given file, set IDs of people that will be included in study
           --peopleExcludeID : give IDs of people that will be included in study
         --peopleExcludeFile : from given file, set IDs of people that will be included in study

`--peopleIncludeID` and `--peopleExcludeID` are used to include/exclude samples from command line. 
For example, specify `--peopleIncludeID A,B,C` will include A, B and C sample from the VCF files if they exists.
`--peopleIncludeID` and `--peopleExcludeID` followed by a file name will include or exclude the IDs in the file.
So to include sample A, B and C, you can provide a file, `people.txt`, looks like:

    A
    B
    C

Then use `--peopleIncludeFile people.txt` to include them in the analysis.


## Variant site filters

It is common that different frequency cutoffs are applied in rare-variant analysis.
Therefore, rvtests specify frequency cutoffs.

Frequency Cutoff

                 --freqUpper : Specify upper frequency bound to be included in analysis
                 --freqLower : Specify lower frequency bound to be included in analysis

Similar to sample inclusion/exclusion options, you can specify a range of variants to be included by 
specifying `--rangeList` option. For example `--rangeList 1:100-200` will include the chromosome 1 position 100bp to 200bp region.
Alternatively, use a separate file, `range.txt`, and `--rangeFile range.txt` to speicify association tests range.

                 --rangeList : Specify some ranges to use, please use chr:begin-end format.
                 --rangeFile : Specify the file containing ranges, please use chr:begin-end format.
                  --siteFile : Specify the file containing sites to include, please use "chr pos" format.

It is supported to filter variant site by site depth, minor allele count or annotation (annotated VCF file is needed).

              --siteDepthMin : Specify minimum depth(inclusive) to be incluced in analysis
              --siteDepthMax : Specify maximum depth(inclusive) to be incluced in analysis
                --siteMACMin : Specify minimum Minor Allele Count(inclusive) to be incluced in analysis
                  --annoType : Specify annotation type that is follwed by ANNO= in the VCF INFO field, regular expression is allowed

*NOTE*: `--annoType Nonsynonymous` will only analyze nonsynonymous variants where they have `ANNO=Nonsynonymous` in the INFO field. 
VCF with annotatino information are called annotated VCF here. And to annotate 
a VCF file, you can use [ANNO](https://github.com/zhanxw/anno), a fast and accurate annotation software.

## Genotyep filters

Genotype with low depth or low quality can be filtered out by these options:

              --indvDepthMin : Specify minimum depth(inclusive) of a sample to be incluced in analysis
              --indvDepthMax : Specify maximum depth(inclusive) of a sample to be incluced in analysis
               --indvQualMin : Specify minimum depth(inclusive) of a sample to be incluced in analysis

When genotypes are filtered, they are marked as missing genotypes. 
Consequently, samples with missing genotype may or may not be included in the analysis.
That means samples with genotypes may be dropped (`--impute drop`) 
or may still be included (`--impute mean` or `--impute hwe`). 
By default, genotypes are imputed to its means.
See next section about how you like to handle missing genotypes.


## Handle missing genotypes and phenotypes

When genotypes are missing (e.g. genotype = "./.") or gentoypes are filtered out, 
there are three options to handle them: (1) impute to its mean(default option); (2) impute by HWE equilibrium; (3) remove from the model.
Use `--impute [mean|hwe|drop]` to specify which option to use.

When quantitative phenotypes are missing, for example, some samples have gneotype files, but not phenotypes, 
rvtests can impute missing phenotype to its mean. 

*NOTE:* Do not use `--imputePheno` for binary trait.

In summary, the following two options can be used:

               --impute : Specify either of mean, hwe, and drop
          --imputePheno : Impute phenotype to mean by those have genotypes but no
                          phenotpyes
                          
                          
## Specify groups (e.g burden unit)

Rare variants association tests are usually performed in gruops of variants. 
The natural grouping unit is gene. Rvtests can read gene definition file in `refFlat` format,
and perform association for each gene. Use `--geneFile` option to specify the gene file name.
For example, `--geneFile refFlat_hg19.txt.gz` will use `refFlat_hg19.txt.gz` as gene definition file,
and then perform association tests for every gene. Use `--gene` to specify a subset of genes to test.
For example, `--gene CFH` will only test CFH gene.

Alternative grouping unit can be specified as *set*. 
These *sets* are treated similar to gene.
You can thus use `--setFile` to define sets (similar to `--geneFile` option), 
and use `--set` to define a specific set (similar to `--gene` option). 
Additionally, use `--setList` can speicify a set to test from command line.

The format of a set file is: (1) set names; (2) ranges (e.g. chrom:begin-end);
For example, you have a set file, `example.set`, like this:

    set1 1:100-200,1:250-300
    set2 2:500-600
    
You can specify `--setFile example.set --set set2` to group variants 
within chromosome 2, position 500 to 600bp. 
If you want to test a particular region, for example, chromosome 2, position 500 to 550bp,
but do not want to make another file, you can use `--setList 2:500-600`.

In summary, options related to *Grouping Unit* are listed below: 

             --geneFile : specify a gene file (for burden tests)
                 --gene : specify which genes to test
              --setList : specify a list to test (for burden tests)
              --setFile : specify a list file (for burden tests, first two columns:
                          setName chr:beg-end)
                  --set : specify which set to test (1st column)


# Contact

Questions and requests can be sent to Xiaowei Zhan
([zhanxw@umich.edu](mailto:zhanxw@umich.edu "mailto:zhanxw@umich.edu"))
or Goncalo Abecasis
([goncalo@umich.edu](mailto:goncalo@umich.edu "mailto:goncalo@umich.edu"))

Rvtests is a collaborative effort by Youna Hu, Bingshan Li, Dajiang
Liu.



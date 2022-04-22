# Matthew Galbraith's GRCh38 GMKF CRAM files notes

CRAMs obtained from CAVATICA data delivery project
Appear to have been aligned using bwamem in ALT-aware mode to GRCh38/hg38

[Human genome reference builds - GRCh38 or hg38 - b37 - hg19](https://gatk.broadinstitute.org/hc/en-us/articles/360035890951-Human-genome-reference-builds-GRCh38-or-hg38-b37-hg19)

[How to Map reads to a reference with alternate contigs like GRCH38](https://gatk.broadinstitute.org/hc/en-us/articles/360037498992)

[How can I tell if a BAM was aligned with alt-handling?](https://gatk.broadinstitute.org/hc/en-us/articles/360037498992#3.1)


## create a clean environment

Always begin with a controlled environment for best case reproducibility of steps:

[conda](conda.io) is installed in the cavatica environment - the platform for the INCLUDE project. Direct login [cavatica.sbgenomics.com](cavatica.sbgenomics.com)

```bash
conda create -n picard -y
```

## Install a few tools

Install things to make work with - any packages and detailed instructions may be found on the [anaconda repository site](https://anaconda.org/anaconda/repo)

Install emacs
```bash
conda install -c conda-forge emacs -y
```

## Inspect the head of the file

Assuming the files are attached to the project - the files may be listed here:

```bash
ls -l /sbgenomics/project-files/
(picard) jovyan@570e064c3ef6:/sbgenomics/workspace$ ls -l /sbgenomics/project-files/total 666
-rw-r--r-- 1 1005 1002  91581 Nov 19 23:50 1KG_MHC_Alts_Decoys.bed
drwxr-xr-x 1 1005 1002   4096 Apr 21 17:31 HTP_CRAMs
-rw-r--r-- 1 1005 1002      0 Nov 18 18:34 output_test.txt
drwxr-xr-x 1 1005 1002   4096 Apr 21 17:31 References
-rw-r--r-- 1 1005 1002 581741 Nov 19 23:51 test2.interval_list
```

Downloaded and configured samtools -- think I will dockerize and containerize the thing!

```bash
wget https://github.com/samtools/samtools/releases/download/1.15.1/samtools-1.15.1.tar.bz2
bzip2 -d samtools-1.15.1.tar.bz2
tar xvf samtools-1.15.1.tar
cd samtools-1.15.1
cd /sbgenomics/workspace/samtools-1.15.1
./configure
./configure --without-curses
./configure --without-curses --disable-lzma
```


```bash
samtools view -H /sbgenomics/project-files/HTP_CRAMs/HTP0003A.cram
```

```bash
samtools view -H /sbgenomics/project-files/HTP_CRAMs/HTP0003A.cram | grep "@SQ" | wc -l
3366
```

```bash
samtools view -H /sbgenomics/project-files/HTP_CRAMs/HTP0003A.cram | grep "@SQ" | grep "HLA" | head
```

```bash
samtools view -H /sbgenomics/project-files/HTP_CRAMs/HTP0003A.cram | grep "@SQ" | grep "HLA" | wc -l
```

## Extract based upon a bed file?

Can extract specific regions from sam/bam/cram files using samtools view and a bed file:

```bash
samtools view -L test.bed HTP0003A.cram
```

However, without providing a reference via -T or –reference, it appears to try and download/cache the file(s) from http://www.ebi.ac.uk/ena/cram/md5/%s and is either very slow or can hang


How to get the appropriate reference file(s)?
GATK Resource Bundle https://gatk.broadinstitute.org/hc/en-us/articles/360035890811-Resource-bundle.%C2%A0

Files stored on Google Cloud:
https://console.cloud.google.com/storage/browser/genomics-public-data/resources/broad/hg38/v0;tab=objects?pli=1&prefix=&forceOnObjectsSortingFiltering=false

If you need it you can get it with:

```bash
wget gs://genomics-public-data/resources/broad/hg38/v0/Homo_sapiens_assembly38.fasta 
```

```bash
head /sbgenomics/project-files/References/Homo_sapiens_assembly38.fasta
```

```bash
cat Homo_sapiens_assembly38.fasta | grep ">" | head
```

```bash
cat Homo_sapiens_assembly38.fasta | grep ">" | wc -l
```

```bash
cat Homo_sapiens_assembly38.fasta | grep ">" | grep "HLA" | head
```

```bash
cat Homo_sapiens_assembly38.fasta | grep ">" | grep "HLA" | wc -l
```

Extracting reads aligned to specific regions from CRAMs
cat test.bed

```bash
samtools view -h --threads=3 -L test.bed -T ~/References/Homo_sapiens_assembly38.fasta HTP0003A.cram > test.out
```
(-h return output with header, required for samtools fastq)
(--threads=3 allows for 3 additional threads to be used; exact parameter name depends on samtools version; online docs for v1.13 lists –nthreads and does not work but man page lists --threads)

```bash
wc -l test.out
```

```bash
wc -l test2.bed
```

```bash
wc -l test2.out # this includes header lines
```

```bash
cat test2.out | samtools view | wc -l
```

```bash
cat test2.out | samtools view -f 1 | wc -l # read paired
```

```bash
cat test2.out | samtools view -f 4 | wc -l # unmapped
```

```bash
cat test2.out | samtools view -f 9 | wc -l # paired, mate unmapped
```

To convert paired-end to fastq, need to pass through collate:

```bash
cat test2.out | samtools collate -u -O - | samtools fastq -1 test2_paired_R1.fastq -2 test2_paired_R2.fastq -0 /dev/null -s test2_singletons.fastq -n
```

```bash
wc -l test2_paired_R1.fastq
```

3148312 / 4 = 787078

```bash
wc -l test2_paired_R2.fastq
```

3148312 / 4 = 787078

sum = 1574156

```bash
wc -l test2_singletons.fastq
```

79552 / 4 = 19888

sum 1574156 + 19888 = 1594044 != 1599436 reported as paired by view above!

These numbers do not add up – all reads are paired in the starting output = problem with collate?

```bash
cat test2.out | samtools sort -n -O sam - | samtools fastq -1 test3_paired_R1.fastq -2 test3_paired_R2.fastq -0 /dev/null -s test3_singletons.fastq -n
```

same problem
contains reads with paired flag set but names that do not match

OR contains reads with paired flag set but the mate fell outside the regions specified in the BED file = more sensical

## alternative approach with picard

Alternative approach that should keep read pairs intact: Picard FilterSamReads with FILTER= includePairedIntervals
https://broadinstitute.github.io/picard/command-line-overview.html#FilterSamReads

java jvm-args -jar picard.jar PicardToolName \
     OPTION1=value1 \
     OPTION2=value2

java -Xmx2g -jar picard.jar FilterSamReads \
     INPUT= \
     OUTPUT=
     FILTER=includePairedIntervals \
     INTERVAL_LIST= \
     SORT_ORDER=queryname

Need to convert BED file to Picard INTERVAL_LIST format ie 0-based vs 1-based + header

BedToIntervalList (requires a sequence dictionary + reference for CRAMs undocumented)

```bash
picard BedToIntervalList \
      I=/sbgenomics/project-files/1KG_MHC_Alts_Decoys.bed \
      O=/sbgenomics/project-files/1KG_MHC_Alts_Decoys.interval_list \
      SD=/sbgenomics/project-files/References/Homo_sapiens_assembly38.dict
```

```bash
picard FilterSamReads \
      REFERENCE_SEQUENCE=/sbgenomics/project-files/References/Homo_sapiens_assembly38.fasta \
      INPUT=/sbgenomics/project-files/HTP_CRAMs/HTP0003A.cram \
      OUTPUT=/sbgenomics/project-files/HTP_CRAMs/HTP0003A_picard_test2.cram \
      FILTER=includePairedIntervals \
      INTERVAL_LIST=test2.interval_list
```
(complains about coordinate sorting although CRAM header list SO:coordinate)

```bash
picard ValidateSamFile \
     I=/sbgenomics/project-files/HTP_CRAMs/HTP0003A.cram \
     MODE=SUMMARY REFERENCE_SEQUENCE=/sbgenomics/project-files/References/Homo_sapiens_assembly38.fasta
```

https://gatk.broadinstitute.org/hc/en-us/articles/360035891231-Errors-in-SAM-or-BAM-files-can-be-diagnosed-with-ValidateSamFile


Seems fine but will try re-sorting:
```bash
picard SortSam \
      I=/sbgenomics/project-files/HTP_CRAMs/HTP0003A.cram \
      O=/sbgenomics/project-files/HTP_CRAMs/HTP0003A_resorted.cram \
      SORT_ORDER=coordinate \
      REFERENCE_SEQUENCE=/sbgenomics/project-files/References/Homo_sapiens_assembly38.fasta \
      CREATE_INDEX=TRUE
```
Takes ~5+ hrs on Macbook:

```bash
picard FilterSamReads \
    REFERENCE_SEQUENCE=/sbgenomics/project-files/References/Homo_sapiens_assembly38.fasta \
    INPUT=/sbgenomics/project-files/HTP_CRAMs/HTP0003A_resorted.cram \
    OUTPUT=/sbgenomics/project-files/HTP_CRAMs/HTP0003A_test2.cram \
    FILTER=includePairedIntervals \
    INTERVAL_LIST=test2.interval_list \
    SORT_ORDER=queryname
```


Looks like it may work if SORT_ORDER=queryname is removed (but will have to collate before conversion to fastq)
Although manual clearly states this is for the output:


Confirmed to work without SORT_ORDER=queryname

```bash
picard FilterSamReads \
    REFERENCE_SEQUENCE=/sbgenomics/project-files/References/Homo_sapiens_assembly38.fasta \
    INPUT=/sbgenomics/project-files/HTP_CRAMs/HTP0003A_resorted.cram \
    OUTPUT=/sbgenomics/project-files/HTP_CRAMs/HTP0003A_test2.cram \
    FILTER=includePairedIntervals \
    INTERVAL_LIST=test2.interval_list
```

Filed a bug with Picard on Github:
https://github.com/broadinstitute/picard/issues/1734

# Final Command

```bash
picard FilterSamReads REFERENCE_SEQUENCE=/sbgenomics/project-files/References/Homo_sapiens_assembly38.fasta INPUT=/sbgenomics/project-files/HTP_CRAMs/HTP0003A.cram OUTPUT=HTP0003A_picard_test2.cram FILTER=includePairedIntervals INTERVAL_LIST=/sbgenomics/project-files/test2.interval_list MAX_RECORDS_IN_RAM=10000000
```


# RUNNING ON CAVATICA

Some tools available under “Public Apps”
Although Picard version here is old + app task GUI does not expose all command line params.
Public Apps > Search for tool>run app>copy to Project>creates task

Alternative is to run as “Interactive Analysis” > Data Cruncher > Create new analysis > Jupyter (may want to turn off suspend) + choose to allow internet access
Can install packages with Conda (gets wiped every time though)

Read Project files from: /sbgenomics/project-files/ (=read only)
Write files to: /sbgenomics/output-files/



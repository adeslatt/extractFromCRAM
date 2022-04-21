# Matthew Galbraith's GRCh38 GMKF CRAM files notes

CRAMs obtained from CAVATICA data delivery project
Appear to have been aligned using bwamem in ALT-aware mode to GRCh38/hg38

[Human genome reference builds - GRCh38 or hg38 - b37 - hg19](https://gatk.broadinstitute.org/hc/en-us/articles/360035890951-Human-genome-reference-builds-GRCh38-or-hg38-b37-hg19)

[How to Map reads to a reference with alternate contigs like GRCH38](https://gatk.broadinstitute.org/hc/en-us/articles/360037498992)

[How can I tell if a BAM was aligned with alt-handling?](https://gatk.broadinstitute.org/hc/en-us/articles/360037498992#3.1)


## Inspect the head of the file

```bash
samtools view -H HTP0003A.cram | head
```

## Inspect the SAM/BAM

Reference [samtools](https://github.com/samtools/samtools#readme) and Peter Robinson's [Computational Exome and Genome Analysis - Chapter 9](https://www.amazon.com/Computational-Analysis-Chapman-Mathematical-Biology/dp/1498775985)

```bash
samtools view -H HTP0003A.cram | grep "@SQ" | head
```

```bash
samtools view -H HTP0003A.cram | grep "@SQ" | wc -l
```


Including ~525 alternate contigs for HLA regions
samtools view -H HTP0003A.cram | grep "@SQ" | grep "HLA" | head


samtools view -H HTP0003A.cram | grep "@SQ" | grep "HLA" | wc -l




Can extract specific regions from sam/bam/cram files using samtools view and a bed file:
samtools view -L test.bed HTP0003A.cram
However, without providing a reference via -T or –reference, it appears to try and download/cache the file(s) from http://www.ebi.ac.uk/ena/cram/md5/%s and is either very slow or can hang


How to get the appropriate reference file(s)?
GATK Resource Bundle https://gatk.broadinstitute.org/hc/en-us/articles/360035890811-Resource-bundle.%C2%A0
Files stored on Google Cloud:
https://console.cloud.google.com/storage/browser/genomics-public-data/resources/broad/hg38/v0;tab=objects?pli=1&prefix=&forceOnObjectsSortingFiltering=false
gsutil cp gs://genomics-public-data/resources/broad/hg38/v0/Homo_sapiens_assembly38.fasta ./

head Homo_sapiens_assembly38.fasta


cat Homo_sapiens_assembly38.fasta | grep ">" | head


cat Homo_sapiens_assembly38.fasta | grep ">" | wc -l


cat Homo_sapiens_assembly38.fasta | grep ">" | grep "HLA" | head


cat Homo_sapiens_assembly38.fasta | grep ">" | grep "HLA" | wc -l


Extracting reads aligned to specific regions from CRAMs
cat test.bed


samtools view -h --threads=3 -L test.bed -T ~/References/Homo_sapiens_assembly38.fasta HTP0003A.cram > test.out

(-h return output with header, required for samtools fastq)
(--threads=3 allows for 3 additional threads to be used; exact parameter name depends on samtools version; online docs for v1.13 lists –nthreads and does not work but man page lists --threads)

wc -l test.out


wc -l test2.bed

wc -l test2.out # this includes header lines

cat test2.out | samtools view | wc -l

cat test2.out | samtools view -f 1 | wc -l # read paired

cat test2.out | samtools view -f 4 | wc -l # unmapped

cat test2.out | samtools view -f 9 | wc -l # paired, mate unmapped


To convert paired-end to fastq, need to pass through collate:
cat test2.out | samtools collate -u -O - | samtools fastq -1 test2_paired_R1.fastq -2 test2_paired_R2.fastq -0 /dev/null -s test2_singletons.fastq -n


wc -l test2_paired_R1.fastq
 # 3148312 / 4 = 787078
 wc -l test2_paired_R2.fastq
  # 3148312 / 4 = 787078
  # sum = 1574156
  wc -l test2_singletons.fastq
   # 79552 / 4 = 19888
   # sum 1574156 + 19888 = 1594044 != 1599436 reported as paired by view above!
   # These numbers do not add up – all reads are paired in the starting output = problem with collate?


cat test2.out | samtools sort -n -O sam - | samtools fastq -1 test3_paired_R1.fastq -2 test3_paired_R2.fastq -0 /dev/null -s test3_singletons.fastq -n

# = same problem
# contains reads with paired flag set but names that do not match

# OR contains reads with paired flag set but the mate fell outside the regions specified in the BED file = more sensical

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
picard BedToIntervalList \
      I=test2.bed \
            O=test2.interval_list \
	          SD=~/References/Homo_sapiens_assembly38.dict
		  picard FilterSamReads \
		      REFERENCE_SEQUENCE=~/References/Homo_sapiens_assembly38.fasta \
		          INPUT=HTP0003A.cram \
			      OUTPUT=HTP0003A_test2.cram \
			          FILTER=includePairedIntervals \
				      INTERVAL_LIST=test2.interval_list \
				          SORT_ORDER=queryname
					  (complains about coordinate sorting although CRAM header list SO:coordinate)

picard ValidateSamFile I=HTP0003A.cram MODE=SUMMARY REFERENCE_SEQUENCE=~/References/Homo_sapiens_assembly38.fasta


https://gatk.broadinstitute.org/hc/en-us/articles/360035891231-Errors-in-SAM-or-BAM-files-can-be-diagnosed-with-ValidateSamFile



Seems fine but will try re-sorting:
picard SortSam \
      I=HTP0003A.cram \
            O=HTP0003A_resorted.cram \
	          SORT_ORDER=coordinate \
		        REFERENCE_SEQUENCE=~/References/Homo_sapiens_assembly38.fasta \
			      CREATE_INDEX=TRUE
			      Takes ~5+ hrs on Macbook:


picard FilterSamReads \
    REFERENCE_SEQUENCE=~/References/Homo_sapiens_assembly38.fasta \
        INPUT=HTP0003A_resorted.cram \
	    OUTPUT=HTP0003A_test2.cram \
	        FILTER=includePairedIntervals \
		    INTERVAL_LIST=test2.interval_list \
		        SORT_ORDER=queryname
			Still complains about sorting!!



Looks like it may work if SORT_ORDER=queryname is removed (but will have to collate before conversion to fastq)
Although manual clearly states this is for the output:


Confirmed to work without SORT_ORDER=queryname


Filed a bug with Picard on Github:
https://github.com/broadinstitute/picard/issues/1734



# RUNNING ON CAVATICA
Some tools available under “Public Apps”
Although Picard version here is old + app task GUI does not expose all command line params.
Public Apps > Search for tool>run app>copy to Project>creates task

Alternative is to run as “Interactive Analysis” > Data Cruncher > Create new analysis > Jupyter (may want to turn off suspend) + choose to allow internet access
Can install packages with Conda (gets wiped every time though)

Read Project files from: /sbgenomics/project-files/ (=read only)
Write files to: /sbgenomics/output-files/

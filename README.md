# Bioinformatics
Sequencing was conducted by the Beijing Genomics Institute (BGI) using the DNBseq platform to produce 100bp paired-end reads at approximately ~5X coverage.
Samples were plated randomly with despect to Dune/Headland morphology (for  natural population samples) or gravitropic/agravitropic status (for advanced recombinant
samples). X lanes were used.

##### A note on differences in this pipeline between populations
The bioformatic processing of the datasets (natural populations and advancted recombinant populations) was conducted by different researchers at separate times. Extremely similar pipelines were used overall, with common programs used for all major steps, as is evidenced below. The few minor cleaning/processing steps where the piplines diverge or use different programs reflects only personal tool preferences.

## Quality filtering
We received forward and reverse files for each individual that had been cleaned by BGI to remove: barcode sequences, DNBseq adaptors, low quality reads (50% of quality scores <10), and reads containing >10% unidentified bases. 

Basic quality control reports were run for each file using ```FastQC``` [(Andrews, 2010)](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/).

```
fastqc ind1_1.fq.gz
```

A report for all individuals was compiled using ```MultiQC v1.8``` [(Ewels et al. 2016)](https://academic.oup.com/bioinformatics/article/32/19/3047/2196507) to screen for any quality-control anomalies.

```
multiqc directory/with/fastqc/outputs
```


## Alignment

The *Senecio lautus* reference genome was indexed using ```BWA v0.7.13``` on default parameters [(Li and Durbin, 2009)](https://pubmed.ncbi.nlm.nih.gov/19451168/).

```
bwa index reference.fasta
```

For each individual, reads were aligned to the reference genome and read groups added using the ```BWA-MEM v0.7.13``` algorithm [(Li and Durbin, 2009)](https://pubmed.ncbi.nlm.nih.gov/19451168/) under default parameters. The resulting BAM files were sorted using ```Samtools v1.3```[(Li et al. 2009)](https://pubmed.ncbi.nlm.nih.gov/19505943/).


```
bwa mem \
-M -R "@RG\tSM:ind1\tID:ind1\tLB:ind1\tPL:ILLUMINA\tPU:ind1" \
        reference.fasta \
        ind1_1.fq.gz \
        ind2_2.fq.gz |
samtools sort -T ind1 -o ind1_sorted.bam 
```

### Alignment statistics
Alignment statistics were calcualted for each individual using ```Samtools v1.3```flagstat function on default parameters.

```
samtools flagstat ind1_sorted.bam &> ind1_stats.txt
```

## Cleaning BAMs

### Basic cleaning
For the advanced recombinant population, sorted BAM files were cleaned using ```Picard v2.22``` [(Broad Institute, 2018)](http://broadinstitute.github.io/picard/) CleanSam to softclip reads extending beyond the reference genome, and set unmapped quality scores to 0.

```
java -Xmx2g -jar picard.jar CleanSam \
	INPUT=ind1_sorted.bam \
	OUTPUT=ind1_cln.sorted.bam
```

### Marking PCR duplicates

For natural population samples, ```samblaster v.0.1.24``` [(Faust and Hall, 2014)](https://academic.oup.com/bioinformatics/article/30/17/2503/2748175) was used on default parameters (using the option to work from a Samtools-sorted file) to mark PCR duplicates for removal.

```
samblaster -M ind1_sorted.bam > ind1_mdup.cln.sorted.bam
```

For natural population samples, ```Picard v2.22``` was used on default parameters (using the option to work from a sorted file) to mark PCR duplicates for removal.

```
java -XX:ConcGCThreads=1 -XX:ParallelGCThreads=1 -Xmx4g -jar picard.jar MarkDuplicates \
	INPUT=ind1_cln.sorted.bam \
	OUTPUT=ind1_mdup.cln.sorted.bam \
	REMOVE_DUPLICATES=false \
	ASSUME_SORTED=true \
	VALIDATION_STRINGENCY=SILENT \
	READ_NAME_REGEX=null \
	MAX_FILE_HANDLES_FOR_READ_ENDS_MAP=900 \
	METRICS_FILE=ind1_mdup.cln.sorted.metrics
```

### Indexing BAMs
Cleaned and sorted BAM files with PCR duplicates marked were then indexed using ```Samtools v1.3```.

```
samtools index ind1_mdup.cln.sorted.bam 
```

## Realigning around indels
Regions around indels were realigned using ```GenomeAnalysisToolKit (GATK) v3.8``` [(Van der Auwera and O'Connor, 2020](https://www.oreilly.com/library/view/genomics-in-the/9781491975183/).

First, a 'dictionary file' of the *S. lautus* reference genome was created using ```Picard v2.2``` for use with ```GATK v3.8```.

```
java -jar picard.jar CreateSequenceDictionary \
  -REFERENCE reference.fasta
```

Then, the *S. lautus* reference genome was indexed with ```Samtools v1.3``` for use with ```GATK v3.8```.

```
samtools faidx /90days/uqralls1/Reference/Senecio.contigs.fasta
```

Targets for realignment were identified using ```GATK v3.8``` RealignerTargetCreator.

```
java -jar GenomeAnalysisTK.jar \
        -T RealignerTargetCreator \
        -R reference.fasta \
        -I ind1_mdup.cln.sorted.bam \
        -nt 2 \
        -o ind1.intervals
```

The realignment was performed using ```GATK v3.8``` IndelRealigner.

```
java -jar /home/uqralls1/programs/GATK/GenomeAnalysisTK.jar \
        -T IndelRealigner \
        -R reference.fasta \
        -I ind1_mdup.cln.sorted.bam \
        -targetIntervals ind1.intervals \
        -o ind1_rln.mdup.cln.bam
```

## Calculating allele frequency
Using the low-coverage variant caller ANGSD v0.930 (Korneliussen et al. 2014) variable sites were called in regions from the auxin and shoot gravitropism gene set across all study populations, then allele frequency at these sites calculated jointly within each study population.



# File processing

Allele frequency files for all populations, both natural and recombinant, were combined in R (script available in files). Only variable sites that were successfully called in all populations were kept. 

Auxin and gravitropism gene names and functions from the original gene set were applied to each site in the combined file using a perl script (available in files).



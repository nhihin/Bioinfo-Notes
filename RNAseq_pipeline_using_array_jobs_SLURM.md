# RNA-seq pipeline using array jobs on SLURM scheduling system

## Summary

- The following pipeline is an example of how SLURM array jobs can be used to submit jobs separately for each sample for each step for an RNA-seq data processing pipeline involving:

1. Quality checking of raw data
2. Quality trimming + quality checking of trimmed data
3. Alignment to reference genome assembly
4. Quantification of gene expression counts
5. Generate MultiQC reports for steps 1-4. 

## Setup - RNA_CONFIG.sh file

- The following file defines the directories/variables that will be referred to throughout the pipeline. 
- All raw fastq files must be placed in the`$DATA` directory. Then copy the `RNASEQ_CONFIG.sh` file in the same directory, make sure the `$DATA` directory is defined properly (full path) and `$LIBRARYTYPE `is correct. Then type `source RNASEQ_CONFIG.sh` in the command line. 
- One of the sections in the file below will move these into `$RAWDATA/fastq` and assign a number to each sample, which allows us to take advantage of SLURM's array jobs. 

```
# INSTRUCTIONS:
# Put raw data in the $DATA directory. 
# Ensure paired-end files are named like : *_1.fastq.gz and *_2.fastq.gz
# or single-end files are naed like: *.fastq.gz.
# Next, edit the SLURM Headers of the  0_RAW.sh, 1_TRIM.sh, 2_ALIGN.sh, 3_GENES.sh, 
# and 2_TRANSCRIPTS.sh scripts, particularly the "#SBATCH --array=1-24" line 
# should be changed to 1-(how many fastq files there are). 

#~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~#
# Setup
#~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~#
THREADS=16 #Should be number of cores * 2
LIBRARYTYPE="single" #Either paired or single

#---------------------------------------------------------------------------------------#
# Data Directories 
#---------------------------------------------------------------------------------------#
# Main directories (via SAHMRI HPC)
DATA=(((REPLACE WITH DIRECTORY WHICH FASTQ FILES ARE IN)))  #All data here
LOCAL=/apps/bioinfo/bin #Local programs

# Data directories
RAWDATA=$DATA/0_rawData
TRIMDATA=$DATA/1_trimmedData
ALIGNDATA=$DATA/2_alignData
QUANTDATA=$DATA/3_quantData

#---------------------------------------------------------------------------------------#
# Create directories to organise data 	
#---------------------------------------------------------------------------------------#
# 0. Quality checking of raw data
mkdir -pv $RAWDATA/fastqc

# 1. Trim and filter raw data, then check quality of trimmed reads. 
mkdir -pv $TRIMDATA/fastq
mkdir -pv $TRIMDATA/fastqc
mkdir -pv $TRIMDATA/slurm_logs

# 2. Alignment to reference genome
mkdir -pv $ALIGNDATA
mkdir -pv $ALIGNDATA/log
mkdir -pv $ALIGNDATA/slurm_logs

# 3. Quantification of gene expression 
mkdir -pv $QUANTDATA
mkdir -pv $QUANTDATA/log
mkdir -pv $QUANTDATA/slurm_logs

#---------------------------------------------------------------------------------------#
# Rename data and move into $RAWDATA
#---------------------------------------------------------------------------------------#
# Move the raw .fastq.gz files into the $RAWDATA directory, and assign a 
# numerical ID number to help with running SLURM scripts in parallel using Job Arrays. 
# More info on Job Arrays: https://slurm.schedmd.com/job_array.html
#ID=1; for file in *1.fastq.gz; do mv ${file} 0_rawData/${ID}_${file}; ID=$(expr $ID + 1); done

cd $DATA
if [ ! -d "$RAWDATA/fastq" ]; then
	mkdir -pv $RAWDATA/fastq

	if [ "$LIBRARYTYPE" == "single" ]; then
		ID=1
		for read in *.fastq.gz
		do
			mv $read $RAWDATA/fastq/${ID}_${read}
			ID=$(expr $ID + 1)
		done
		echo "Moved the single-end .fastq.gz files in $DATA to $RAWDATA/fastq."
	fi

	if [ "$LIBRARYTYPE" == "paired" ]; then
		ID=1
		for firstread in *_1.fastq.gz
		do
			mv $firstread $RAWDATA/fastq/${ID}_${firstread}
			ID=$(expr $ID + 1)
		done

		ID=1
		for secondread in *_2.fastq.gz
		do
			mv $secondread $RAWDATA/fastq/${ID}_${secondread}
			ID=$(expr $ID + 1)
		done
		echo "Moved the paired-end .fastq.files in $DATA to $RAWDATA/fastq."
	fi
fi


#---------------------------------------------------------------------------------------#
#  References and Indexes 
#---------------------------------------------------------------------------------------#
# Here are the locations of all refs and indexes needed to run the pipeline. 
# In this pipeline we use the Mouse mm10 genome reference assembly. 

REFS=/homes/nhi.hin/REFS
# The following command was used to generate $STARIDX:
# /homes/nhi.hin/TOOLS/star-2.7.1a-0/bin/STAR --runThreadN 4 --runMode genomeGenerate --genomeDir mm10_star271 --genomeFastaFiles Mus_musculus.GRCm38.dna_rm.primary_assembly.fa --sjdbGTFfile Mus_musculus.GRCm38.102.gtf --sjdbOverhang 100
STARIDX=$REFS/mm10_star271
GTF=$REFS/Mus_musculus.GRCm38.102.gtf
FASTA=$REFS/Mus_musculus.GRCm38.dna_rm.primary_assembly.fa

#---------------------------------------------------------------------------------------#
# Programs 
#---------------------------------------------------------------------------------------#
AdapterRemoval=/apps/bioinfo/bin/AdapterRemoval
STAR=/homes/nhi.hin/TOOLS/star-2.7.1a-0/bin/STAR
fastqc=/homes/nhi.hin/TOOLS/FastQC/fastqc
featureCounts=/homes/nhi.hin/TOOLS/subread-2.0.1-Linux-x86_64/bin/featureCounts

```

## Step 1. Quality checking of raw data

`0_RAW.sh`

```
#!/bin/bash                           

#SBATCH --job-name=QC_rawData
#SBATCH --mail-user=nhi.hin@sahmri.com
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL
#SBATCH --output=0_RAW.log

# Resources allocation request parameters

#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem 16000M
#SBATCH --time=1:00:00   
#SBATCH --array=1-8

# Activate conda environment MULTIQC_env which contains the following installed: multiqc, fastqc
source activate MULTIQC_env
# Access file directory variables defined here:
source RNASEQ_CONFIG.sh

cd ${RAWDATA}/fastq

# Run FastQC for each read
if [ "$LIBRARYTYPE" == "single" ]; then
	READ=`ls ${SLURM_ARRAY_TASK_ID}*.fastq.gz`
	fastqc -t $THREADS -f fastq -o $RAWDATA/fastqc $READ
fi

if [ "$LIBRARYTYPE" == "paired" ]; then
	READ1=`ls ${SLURM_ARRAY_TASK_ID}_*_1.fastq.gz`
	READ2=`ls ${SLURM_ARRAY_TASK_ID}_*_2.fastq.gz`
	$fastqc -t $THREADS -f fastq -o ${RAWDATA}/fastqc ${READ1}
	$fastqc -t $THREADS -f fastq -o ${RAWDATA}/fastqc ${READ2}
fi


conda deactivate
```

## Step 2. Quality trimming + quality checking of trimmed data

`1_TRIM.sh`

```
#!/bin/bash
#SBATCH --job-name=Trim
#SBATCH --mail-user=nhi.hin@sahmri.com
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL
#SBATCH --output="1_TRIM_%a.log"
#SBATCH --err="1_TRIM_%a.err"

# Resources allocation request parameters

#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem 16000M
#SBATCH --time=1:00:00   
#SBATCH --array=1-8

# Activate conda environment ADAPTERRREMOVAL_env which contains the following installed: AdapterRemoval, fastqc, MultiQC
source activate ADAPTERREMOVAL_env

source RNASEQ_CONFIG.sh

cd $RAWDATA/fastq

if [ "$LIBRARYTYPE" == "single" ]; then
	READ=`ls ${SLURM_ARRAY_TASK_ID}*.fastq.gz`
	TRIM=${TRIMDATA}/fastq/${READ}

	AdapterRemoval --file1 $READ \
	--output1 $TRIM \
	--threads $THREADS \
	--gzip \
	--trimqualities \
	--trimns \
	--minquality 20 \
	--minlength 35

	fastqc -t $THREADS -f fastq -o ${TRIMDATA}/fastqc $TRIM
fi

if [ "$LIBRARYTYPE" == "paired" ]; then
	READ1=`ls ${SLURM_ARRAY_TASK_ID}*_1.fastq.gz`
	READ2=`ls ${SLURM_ARRAY_TASK_ID}*_2.fastq.gz`

	TRIM1=${TRIMDATA}/fastq/${READ1}
	TRIM2=${TRIMDATA}/fastq/${READ2}

	AdapterRemoval --file1 $READ1 \
	--file2 $READ2 \
	--output1 $TRIM1 \
	--output2 $TRIM2 \
	--threads $THREADS \
	--gzip \
	--trimqualities \
	--trimns \
	--minquality 20 \
	--minlength 35

	fastqc -f fastq -t $THREADS -o ${TRIMDATA}/fastqc $TRIM1
	fastqc -f fastq -t $THREADS -o ${TRIMDATA}/fastqc $TRIM2
fi

conda deactivate
```

## Step 3. Alignment of trimmed reads to reference genome assembly

`2_ALIGN.sh`

```
#!/bin/bash

#SBATCH --job-name=Align
#SBATCH --mail-user=nhi.hin@sahmri.com
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL
#SBATCH --err="Align_%a.err"
#SBATCH --output="Align_%a.out"

# Resources allocation request parameters
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem-per-cpu=8GB
#SBATCH --time=3:00:00          # Run time in hh:mm:
#SBATCH --array=1-8

source RNASEQ_CONFIG.sh


cd $TRIMDATA/fastq

READ=`ls ${SLURM_ARRAY_TASK_ID}_*.fastq.gz`
BAM1=${READ%%fastq.gz}

# Run STAR to align trimmed fastq files to the reference genome assembly.
# For details on how the index STARIDX was created, see RNASEQ_CONFIG.sh
$STAR \
    --runThreadN $THREADS \
    --genomeDir $STARIDX \
    --readFilesIn $READ \
    --readFilesCommand gunzip -c \
    --outFileNamePrefix ${ALIGNDATA}/${BAM1} \
    --outSAMtype BAM SortedByCoordinate

cd $ALIGNDATA

# Move log files into a separate directory
mv *out log
mv *tab log

# Create the index for each alignment .bam file
# This ensures that the bam files can be viewed in IGV Viewer. 
# samtools local copy available on SAHMRI HPC: /apps/bioinfo/custom_local/miniconda3/bin/samtools
BAM2=`ls ${SLURM_ARRAY_TASK_ID}_*.bam`
samtools index $BAM2
```

## Step 4. Quantification of gene expression

`3_QUANT.sh`

```
#!/bin/bash

#SBATCH --job-name=Quantify
#SBATCH --mail-user=nhi.hin@sahmri.com
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL
#SBATCH --err="Quantify_%a.err"
#SBATCH --output="Quantify_%a.out"

# Resources allocation request parameters
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem-per-cpu=8GB
#SBATCH --time=3:00:00          # Run time in hh:mm:

# Run FeatureCounts to quantify gene expression counts for each aligned .bam file

source activate FEATURECOUNTS_env
source RNASEQ_CONFIG.sh

cd $ALIGNDATA

if [ "$LIBRARYTYPE" == "single" ]; then
sampleList=`find $ALIGNDATA -name "*.bam" | tr '\n' ' '`
featureCounts -Q 10 -s 0 -T ${THREADS} -a $GTF -o $QUANTDATA/geneCounts.out ${sampleList}
cut -f1,7- $QUANTDATA/geneCounts.out | sed 1d > $QUANTDATA/geneCounts.txt
fi

if [ "$LIBRARYTYPE" == "paired" ]; then
sampleList=`find $ALIGNDATA -name "*.bam" | tr '\n' ' '`
featureCounts -Q 10 -s 0 -T ${THREADS} -p -a $GTF -o $QUANTDATA/geneCounts.out ${sampleList}
cut -f1,7- $QUANTDATA/geneCounts.out | sed 1d > $QUANTDATA/geneCounts.txt
fi

# Move log files to separate directory.
mv $QUANTDATA/geneCounts.out $QUANTDATA/log
mv $QUANTDATA/geneCounts.out.summary $QUANTDATA/log

conda deactivate
```

## Step 5. MultiQC reports

`4_MULTIQC.sh`

```
#!/bin/bash

#SBATCH --job-name=MultiQC
#SBATCH --mail-user=nhi.hin@sahmri.com
#SBATCH --mail-type=BEGIN
#SBATCH --mail-type=END
#SBATCH --mail-type=FAIL
#SBATCH --err="MultiQC_%a.err"
#SBATCH --output="MultiQC_%a.out"

# Resources allocation request parameters
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem-per-cpu=8GB
#SBATCH --time=0:30:00          # Run time in hh:mm:

# MultiQC summarises all quality checking reports into a single 
# multiqc_report.html file which is easily viewable in a web browser

# Activate the conda environment containing an install of MultiQC:
source activate MULTIQC_env
# Access paths/variables defined in the following file:
source RNASEQ_CONFIG.sh

# Generate multiqc report for raw fastqc's
# Report available at $RAWDATA/fastqc/fastqc_report.html
cd $RAWDATA/fastqc
multiqc .

# Generate multiqc report for trimmed fastqc's
# Report available at $TRIMDATA/fastqc/fastqc_report.html
cd $TRIMDATA/fastqc
multiqc .

# Generate multiqc report for mapped/aligned reads
# Report available at $ALIGNDATA/logs/fastqc_report.html
cd $ALIGNDATA/logs
multiqc .

# Generate multiqc report for quantification/gene counts
# Report available at $QUANTDATA/logs/fastqc_report.html
cd $QUANTDATA/logs
multiqc .

conda deactivate
```

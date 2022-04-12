# RNA velocity analysis in scRNA-seq data using Kallisto and bustools

## Prerequisites

- Create a new conda environment with Python 3 and use `pip` to install the `kb` wrapper into it, see [kb documentation](https://www.kallistobus.tools/tutorials/kb_velocity_index/python/kb_velocity_index/#install-kb). `kb` is a wrapper around `kallisto` and `bustools` commands and it simplifies the RNA velocity workflow. Unfortunately I have had some errors using the full `kb` workflow (`kb index` + `kb_count`), so in this page we will just be using it to generate the index. For counting, we will do the steps separately using a combination of `kallisto` and `bustools`.

```
conda create --name KB_env python=3
conda activate KB_env
pip install git+https://github.com/pachterlab/kb_python@count-kite
conda deactivate
```

- Create another conda environment with `kallisto` and `bustools` installed. 

```
conda create --name BUSTOOLS_env -c bioconda bustools kallisto
```

## Step 1 - Generate index

- A genome .fa file and gene models .gtf file for the organism of interest needs to be downloaded first e.g. from ENSEMBL. To maintain consistency with the genome and gene models used in cell ranger, I am using the mm10 (mouse) or GRCh38 (human) ones by cell ranger which can be downloaded [here](https://support.10xgenomics.com/single-cell-gene-expression/software/downloads/latest). 
- To generate an index suitable for RNA velocity, the following command can be used. 

```
REFDIR=/homes/nhi.hin/REFS/refdata-gex-mm10-2020-A
source activate KB_env

kb ref -i index.idx -g t2g.txt -f1 cdna.fa -f2 intron.fa -c1 cdna_t2c.txt -c2 intron_t2c.txt --workflow lamanno -n 8 --overwrite \
$REFDIR/fasta/genome.fa \
$REFDIR/genes/genes.gtf
conda deactivate
```

The second step is to make a combined index that contains both spliced and unspliced transcripts as follows. See `dsull`'s comment from this [biostars thread](https://www.biostars.org/p/468180/). 

```
source activate KALLISTOBUSTOOLS_env
cat cdna.fa intron.fa > cdna_intron.fa
kallisto index -i index_cDNA_introns.idx -k 31 cdna_intron.fa
```

- This command will create an index suitable for use with `kallisto` + `bustools` called `index_cDNA_introns.idx`. It also splits the genome .fa file into the cDNA and intron .fa files. It will also create two transcript-to-capture lists `cdna_t2c.txt` and `intron_t2c.txt` both of which will also be used. 

## Step 2 - Generate count matrices 

- The steps to generate count matrices using `kallisto` and `bustools` are:

1. Run `kallisto bus`
2. Run `bustools correct`
3. Run `bustools sort`
4. Run `bustools capture`
5. Run `bustools count`

- These steps have been adapted from the tutorial [here](https://bustools.github.io/BUS_notebooks_R/velocity.html#directly_using_kallisto_and_bustools). 

- My scripts to run these steps are shown below. 

**Running kallisto bus**

- The kallisto bus command is quite straightforward, taking the kallisto index generated in the previous step, path to the output directory, the 10X technology being used (here it is `10xv3`), and list of paired-end fastq files in order. 

```
# Where the fastq files are stored (after cellranger mkfastq)
FASTADIR=/homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY

# Where the reference is stored, including indexes, etc generated in previous step
REFDIR=/homes/nhi.hin/REFS/refdata-gex-mm10-2020-A

# Where the results of this will be stored, create this dir beforehand
RESULTSDIR=/homes/nhi.hin/WORKING/Project1/kallistoResults


source activate BUSTOOLS_env

kallisto bus -i $REFDIR/index_cDNA_introns.idx \
-o $RESULTSDIR -x 10xv3 -t8 \
/homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/AOPEmAT2/AOPEmAT2_S1_L001_R1_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/AOPEmAT2/AOPEmAT2_S1_L001_R2_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/AOPEmPBS/AOPEmPBS_S5_L001_R1_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/AOPEmPBS/AOPEmPBS_S5_L001_R2_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/AOPEpAT2/AOPEpAT2_S2_L001_R1_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/AOPEpAT2/AOPEpAT2_S2_L001_R2_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/AOPEpPBS/AOPEpPBS_S6_L001_R1_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/AOPEpPBS/AOPEpPBS_S6_L001_R2_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/HEPEmAT2/HEPEmAT2_S3_L001_R1_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/HEPEmAT2/HEPEmAT2_S3_L001_R2_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/HEPEmPBS/HEPEmPBS_S7_L001_R1_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/HEPEmPBS/HEPEmPBS_S7_L001_R2_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/HEPEpAT2/HEPEpAT2_S4_L001_R1_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/HEPEpAT2/HEPEpAT2_S4_L001_R2_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/HEPEpPBS/HEPEpPBS_S8_L001_R1_001.fastq.gz /homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY/HEPEpPBS/HEPEpPBS_S8_L001_R2_001.fastq.gz
```

- This will create the following files in the output directory: `matrix.ec` and `output.bus`.

**Running bustools correct**

- Before bustools correct can be run, a whitelist needs to be generated of all valid barcodes from the kit, using the output.bus file from the previous step. This can be generated as follows.

```
conda activate BUSTOOLS_env
bustools whitelist -o kallistoResults/10xv3_whitelist.txt kallistoResults/output.bus
```

- Now bustools correct checks the whitelist and can error-correct the bus file depending on the contents of the whitelist. 

```
# Where the fastq files are stored (after cellranger mkfastq)
FASTADIR=/homes/nhi.hin/WORKING/Project1/Project1/outs/fastq_path/H3TJYDRXY

# Where the reference is stored, including indexes, etc generated in previous step
REFDIR=/homes/nhi.hin/REFS/refdata-gex-mm10-2020-A

# Where the results of this will be stored, create this dir beforehand
RESULTSDIR=/homes/nhi.hin/WORKING/Project1/kallistoResults

# Path to where the generated whitelist is
WHITELIST=/homes/nhi.hin/WORKING/Peter_Velocity_HeartAorta/kallistoResults/10xv3_whitelist.txt

source activate BUSTOOLS_env

cd $RESULTSDIR
bustools correct -w $WHITELIST -o output.correct.bus output.bus 
```

## Relevant Links

- https://bustools.github.io/BUS_notebooks_R/velocity.html#directly_using_kallisto_and_bustools
- https://www.kallistobus.tools/tutorials/kb_velocity/python/kb_velocity/

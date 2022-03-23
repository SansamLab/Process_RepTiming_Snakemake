# Process_RepTiming_Snakemake

## Project Description:

## Table of contents:
* [Description of individual steps in pipeline](https://github.com/SansamLab/Process_HiC_SnakeMake/edit/main/README.md#description-of-individual-steps-in-pipeline)
  * [1.  trim_reads_with_fastp](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#1--run_bwa_mem)
  * [2.  align_reads_with_bwamem](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#2--make_pairs_with_pairtools_parse)
  * [3.  mark_duplicates_with_picard](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#3--sort_pairs_with_pairtools_sort)
  * [4.  quality_filter_with_bamtools](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#4--mark_duplicates_with_pairtools_dedup)
  * [5.  blacklist_filter_with_bedtools](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#5--filter_pairs)
  * [6.  count_reads_in_RT_windows](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#6--add_frag2pairs)
* [Step-by-step instructions on running Snakemake pipeline:](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#step-by-step-instructions-on-running-snakemake-pipeline)
  * [1.  Load slurm and miniconda](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#1--load-slurm-and-miniconda)
  * [2.  Clone repository](https://github.com/SansamLab/Process_HiC_SnakeMake#2--clone-repository)
  * [3.  Start the conda environment](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#3--start-the-conda-environment)
    * [3A.  FIRST TIME ONLY:  Setup conda environment](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#3a--first-time-only--setup-conda-environment)
    * [3B.  Activate conda environment](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#3b--activate-conda-environment)
  * [4.  Modify the job-specific configuration files.](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#4--modify-the-job-specific-coniguration-files)
    * [4A.  Modify the config/config.yml file](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#4a--modify-the-configconfigyml-file)
    * [4B.  Modify the config/samples.csv file](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#4b--modify-the-configsamplescsv-file)
    * [4C.  IF SLURM RESOURCE CHANGES ARE NEEDED. Modify the config/cluster_config.yml file](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#4c--if-slurm-resource-changes-are-needed-modify-the-configcluster_configyml-file)
  * [5.  Do a dry run](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#4--do-a-dry-run)
  * [6.  Make a DAG diagram](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#5--make-a-dag-diagram)
  * [7.  Run on cluster with slurm](https://github.com/SansamLab/Process_HiC_SnakeMake/blob/main/README.md#6--run-on-cluster-with-slurm)

## Description of individual steps in pipeline:
![DAG of Pipeline](dag.svg)

### 1.  trim_reads_with_fastp
```bash
fastp \
  -i {input.fq1} \
  -I {input.fq2} \
  -o {output.trimmed1} \
  -O {output.trimmed2} \
  -h {output.fastp_report} \
  --json {output.fastp_json} \
  -R "{wildcards.sample}" \
  -w {params.threads}
```
### 2.  run_bwa_mem
```bash
# align pair of .fastq files to the genome and convert the .sam output to .bam
bwa mem \
  -M \
  -t {params.threads} \
  {params.genome} \
  {input.R1} {input.R2} | \
samtools sort \
  -@ {params.threads} > {output.bam}
    
samtools index \
  -@ {params.threads} \
    {output.bam} > {output.bai}
```
### 3.  mark_duplicates_with_picard
```bash
JAVA_MEM_OPTS={params.memory}

RAM=$(echo $JAVA_MEM_OPTS | rev | cut -c2- | rev)

picard MarkDuplicates \
  COMPRESSION_LEVEL=9 \
  VALIDATION_STRINGENCY=LENIENT \
  MAX_RECORDS_IN_RAM=$((200000*RAM)) \
  CREATE_INDEX=true \
  I={input.bam} \
  O={output.marked_bam} \
  M={output.marked_metrics}

samtools index {output.marked_bam}
```

### 4.  quality_filter_with_bamtools
```bash
bamtools filter \
  -in {input.marked_bam} \
  -forceCompression \
  -mapQuality '{params.mapQ}' \
  -isDuplicate false \
  -isFailedQC false \
  -isMapped true \
  -isMateMapped true \
  -isPaired true \
  -isPrimaryAlignment true \
  -isProperPair true \
  -out {output.filtered_bam}

samtools index {output.filtered_bam}
```

### 5.  blacklist_filter_with_bedtools
```bash
bedtools intersect \
  -v \
  -split \
  -abam {input.filtered_bam} \
  -b {params.HiCount_BED_FILENAME} > {input.filtered_bam}_FILTERED1.BAM

bedtools intersect \
  -v \
  -split \
  -f 0.3 \
  -abam {input.filtered_bam}_FILTERED1.BAM \
  -b {params.Sat_BED_FILENAME} > {output.doubleFiltered_bam}

rm {input.filtered_bam}_FILTERED1.BAM

samtools index {output.doubleFiltered_bam}
```
### 6.  count_reads_in_RT_windows
```bash
Rscript \
  workflow/scripts/CountReadsOverBed_ver02.R \
    {params.RT_windows} \
    {input.doubleFiltered_bam} \
    {output.counts}          
```

## Step-by-step instructions on running Snakemake pipeline:

### 1.  Load slurm and miniconda
Note. The commands to do this will be different on your machine. These commands are specific to an HPC using slurm with these modules installed.

```bash
ml slurm
ml miniconda
```
### 2.  Clone repository
```bash
git clone https://github.com/SansamLab/Process_RepTiming_Snakemake.git
# rename folder with project name
mv Process_RepTiming_Snakemake/ My_RT_Project_Folder/
# change directory into root of your project folder
cd My_RT_Project_Folder
```
### 3.  Start the conda environment
### 3A.  FIRST TIME ONLY:  Setup conda environment
```bash
# -f is the location of the environment .yml file. 
## The relative path assumes that you are in the root directory of this repository.
# -p is the path where you want to install this environment
conda env create -f workflow/envs/RTEnv.yml -p /s/sansam-lab/RT_Environment 
```

### 3B.  Activate conda environment
```bash
conda activate /s/sansam-lab/RT_Environment
```

### 4.  Modify the job-specific configuration files.
#### 4A.  Modify the config/config.yml file

You must enter the following:
* samples_table:
  * path and name of the .csv file that lists sample names and paths to all fastq files
  * example:  "config/testSamples.csv"
* fastp_cpus:
  * number of processors that fastp will use to trim your sequencing reads
  * example:  8
* bwa_cpus:
  * number of processors that bwa mem will use to align reads to genome
  * example:  12
* picard_memory:
  * amount of memory allocated to picard for marking duplicates
  * example:  "24G"
* bwa_genome:
  * location of bwa indexed genome for the alignment
  * example:  "/Volumes/hts_core/Shared/zebrafish/danRer11/noAlts/danRer11_noAlts.fa"
* bamtools_filter_mapQ:
  * mapQ score cutoff for quality filtering reads
  * example:  ">10"
* bedtools_intersect_blacklist:
  * .bed file with blacklisted genomic regions for filtering out reads
  * example:  "resources/Satellites.bed"
    * for zebrafish GRCz11
    * file has regions with certain repeat types that we find are associated with anomalously high read counts
* bedtools_intersect_greylist:
  * .bed file with greylisted genomic regions for filtering out reads
  * example:  "resources/HiCountWindows.bed"
    * for zebrafish GRCz11
    * file has regions that have anomalously high read counts in multiple Tab5 whole genome sequencing runs
* RT_windows:
  * .bed file with regions used for calculating replication timing values
  * example:  "resources/RTWindows_danRer11_noAlts_ver01.bed"
    * for zebrafish GRCz11

#### 4B.  Modify the config/samples.csv file

The samples.csv file in the config folder has paths to the test fastq files. You must replace those paths with those for your own fastq files. The first column of each row is the sample name. This name will be used for all output files.

#### 4C.  IF SLURM RESOURCE CHANGES ARE NEEDED. Modify the config/cluster_config.yml file

CPU and memory requests for each rule in the pipeline are detailed in this file. If you are using SLURM, you may need to alter this file to fit your needs/system.

### 5.  Do a dry run.
A dry run produces a text output showing exactly what commands will be executed. Look this over carefully before submitting the full job. It is normal to see warnings about changes made to the code, input, and params.
```bash
snakemake -npr
```

### 6.  Make a DAG diagram.
```bash
snakemake --dag | dot -Tpdf > dag.pdf
```

### 7.  Run on cluster with slurm.
This snakemake pipeline could be executed without slurm, but if an hpc with slurm is used, the following will start the pipeline with the parameters defined in the config/cluster_config.yml file.
```bash
sbatch --wrap="\
snakemake \
-R \
-j 999 \
--cluster-config config/cluster_config.yml \
--cluster '\
sbatch \
--constraint=westmere \
-A {cluster.account} \
-p {cluster.partition} \
--cpus-per-task {cluster.cpus-per-task}  \
--mem {cluster.mem} \
--output {cluster.output}'"
```

### 8.  Check results, and when finished, exit environment.
The results will be saved to the "results" folder. Look over log files generated in either the logs/ or logs/snakelogs folders (depending on whether slurm was used).
```bash
conda deactivate
```

## References:

Goloborodko, A., Nezar Abdennur, Venev, S., Hbbrandao, & Gfudenberg. (2019). mirnylab/pairtools v0.3.0 (v0.3.0) [Computer software]. Zenodo. https://doi.org/10.5281/ZENODO.2649383

Abdennur, N., Goloborodko, A., Imakaev, M., Kerpedjiev, P., Fudenberg, G., Oullette, S., Lee, S., Strobelt, H., Gehlenborg, N., & Mirny, L. (2021). open2c/cooler: v0.8.11 (v0.8.11) [Computer software]. Zenodo. https://doi.org/10.5281/ZENODO.4655850

Lee, S., Bakker, C. R., Vitzthum, C., Alver, B. H., & Park, P. J. (2022). Pairs and Pairix: a file format and a tool for efficient storage and retrieval for Hi-C read pairs. In C. Alkan (Ed.), Bioinformatics (Vol. 38, Issue 6, pp. 1729–1731). Oxford University Press (OUP). https://doi.org/10.1093/bioinformatics/btab870

Durand, N. C., Shamim, M. S., Machol, I., Rao, S. S. P., Huntley, M. H., Lander, E. S., & Aiden, E. L. (2016). Juicer provides a one-click system for analyzing loop-resolution hi-c experiments. Cell Systems, 3(1), 95–98. https://doi.org/10.1016/j.cels.2016.07.002

Danecek, P., Bonfield, J. K., Liddle, J., Marshall, J., Ohan, V., Pollard, M. O., Whitwham, A., Keane, T., McCarthy, S. A., Davies, R. M., & Li, H. (2021). Twelve years of samtools and bcftools. GigaScience, 10(2), giab008. https://doi.org/10.1093/gigascience/giab008

Li, H. (2013). Aligning sequence reads, clone sequences and assembly contigs with BWA-MEM. ArXiv:1303.3997 [q-Bio]. http://arxiv.org/abs/1303.3997

Köster, J., & Rahmann, S. (2012). Snakemake—A scalable bioinformatics workflow engine. Bioinformatics (Oxford, England), 28(19), 2520–2522. https://doi.org/10.1093/bioinformatics/bts480

Anaconda Software Distribution. (2020). Anaconda Documentation. Anaconda Inc. Retrieved from https://docs.anaconda.com/

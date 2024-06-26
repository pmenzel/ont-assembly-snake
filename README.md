# ont-assembly-snake

A Snakemake wrapper for easily creating *de novo* bacterial isolate genome assemblies from Oxford Nanopore (ONT) sequencing data, and optionally Illumina data,
using any combination of read filtering, assembly, long and short read polishing, and reference-based polishing.

The workflow is published in [Snakemake workflows for long-read bacterial genome assembly and evaluation](https://gigabytejournal.com/articles/116) in _GigaByte_.

## Included programs

| read filtering | assembly | long read polishing | short read polishing | reference-based polishing |
| --- | --- | --- | --- | --- |
| [Filtlong](https://github.com/rrwick/Filtlong)<br/> [Rasusa](https://github.com/mbhall88/rasusa) | [Flye](https://github.com/fenderglass/Flye)<br/> [raven](https://github.com/lbcb-sci/raven)<br/> [miniasm](https://github.com/lh3/miniasm)<br/> [Unicycler](https://github.com/rrwick/Unicycler)<br/> [Canu](https://github.com/marbl/canu)  | [racon](https://github.com/lbcb-sci/racon)<br/> [medaka](https://github.com/nanoporetech/medaka) | [pilon](https://github.com/broadinstitute/pilon/wiki)<br/> [Polypolish](https://github.com/rrwick/Polypolish)<br/> [POLCA](https://github.com/alekseyzimin/masurca#polca) | [Homopolish](https://github.com/ythuang0522/homopolish)<br/> [proovframe](https://github.com/thackl/proovframe) | 


## Quick start
```bash
# Install
git clone https://github.com/pmenzel/ont-assembly-snake.git
conda config --add channels bioconda
conda env create -n ont-assembly-snake --file ont-assembly-snake/env/conda-main.yaml
conda activate ont-assembly-snake

# Prepare ONT reads, one file per sample
mkdir fastq-ont
cp /path/to/my/data/my_sample/ont_reads.fastq.gz fastq-ont/mysample.fastq.gz

# optionally: add Illumina paired-end reads
mkdir fastq-illumina
cp /path/to/my/data/my_sample/illumina_reads_R1.fastq.gz fastq-illumina/mysample_R1.fastq.gz
cp /path/to/my/data/my_sample/illumina_reads_R2.fastq.gz fastq-illumina/mysample_R2.fastq.gz

# Declare desired combination of read filtering, assembly and polishing
mkdir assemblies
mkdir assemblies/mysample_flye+medaka
mkdir assemblies/mysample+filtlongMB500_flye+racon2+medaka
mkdir assemblies/mysample_raven2+medaka+pilon
[...]

# Run workflow
snakemake -s ont-assembly-snake/Snakefile --use-conda --cores 20
```

## Setup
Clone repository, for example into the existing folder `/opt/software/`:
```
git clone https://github.com/pmenzel/ont-assembly-snake.git /opt/software/ont-assembly-snake
```
Install [conda](https://docs.conda.io/en/latest/miniconda.html) and create a new environment called `ont-assembly-snake`:
```
conda config --add channels bioconda
conda env create -n ont-assembly-snake --file /opt/software/ont-assembly-snake/env/conda-main.yaml
```
Activate the environment:
```
conda activate ont-assembly-snake
```

## Usage
First, prepare a folder called `fastq-ont/` containing the ONT sequencing reads as
one `.fastq` or `.fastq.gz` file per sample, e.g. `fastq-ont/sample1.fastq.gz`.

Unicycler and some polishing tools additionally use paired-end Illumina reads, which need to be placed in the folder `fastq-illumina/` using `_R[12].fastq` or `_R[12].fastq.gz` suffixes, e.g. `fastq-illumina/sample1_R1.fastq.gz` and `fastq-illumina/sample1_R2.fastq.gz`.

Next, create a folder `assemblies` and in there, create empty folders specifying
the desired combinations of read filtering, assembly, and polishing steps by using specific keywords for each program, see below.

The first part of a folder name is a sample name, which must match the filenames in `fastq-ont/` and, optionally, `fastq-illumina/`.

The next part can be a keyword for read filtering with Filtlong, see below, which is separated from the sample name by `+`.

Then follows, separated by an underscore, a keyword for the assembler.  
NB: This also means that sample names must not contain underscores.

After the keyword for the assembler follow the keywords for one ore more polishing steps, all separated by `+`.

As an alternative to create subfolders within `assemblies/`, it is also possible to use a YAML file to list the desired assemblies, e.g. in a file called `samples.yaml`:
```
assemblies:
  - mysample_flye+medaka
  - mysample+filtlongMB500_flye+racon2+medaka
  - mysample_raven2+medaka+pilon
```
and add the argument `--configfile samples.yaml` to the Snakemake command line.

Finally, after making the desired subfolders in `assemblies/` or making the config file, run the workflow, e.g. with 20 threads:
```
snakemake -s /opt/software/ont-assembly-snake/Snakefile --use-conda --cores 20
```

### Workflow output

The assemblies created in each step are contained in the files `output.fa` (FASTA format) in each of the previously created folders within `assemblies/`.
Additionally, a symlink named `<FOLDER_NAME>.fa` links to the final output file of each step, i.e. `assemblies/<FOLDER_NAME>/output.fa`, also see the example below.  
The log files of running the individual Snakemake rules are located in `assemblies/<FOLDER_NAME>/*log.txt`.

## Test dataset
A test dataset containing a pair of ONT and Illumina sequencing data from the same bacterial isolate is available in the repository [ont-assembly-snake-testdata](https://github.com/pmenzel/ont-assembly-snake-testdata). See the instructions therein on how to download the dataset and run the ont-assembly-snake and [score-assemblies](https://github.com/pmenzel/score-assemblies) workflows.

## Included Programs

### Filtlong
The ONT reads can be filtered by length and quality using [Filtlong](https://github.com/rrwick/Filtlong) prior to the assembly.

The available keywords are:

**`filtlong`**:  
This will filter the ONT reads in `fastq-ont/mysample.fastq` and keep only
reads longer than 1000 bases; using the Filtlong option `--min_length`. The filtered read set is written to
`fastq-ont/mysample+filtlong.fastq`. The length can be changed using the
Snakemake configuration option `filtlong_min_read_length`.

**`filtlongPC<p>`**  
This will filter the reads to only include the top `p` percent of megabases from reads with highest average quality using
the Filtlong option `--keep_percent`.
Further, reads are filtered by their length as above. The output is written to `fastq-ont/mysample+filtlongPC<p>.fastq`.

**`filtlongMB<m>`**  
This will filter the reads to only include reads with highest average quality up to a total length of `m` megabases.
Further, reads are filtered by their length. The output is written to `fastq-ont/mysample+filtlongMB<m>.fastq`.

**`filtlongMB<m>,<q>,<l>`**  
This will filter the reads to only include reads up to a total length of `m` megabases, which are filtered by length
and quality, where `q` and `l` set the priority for each using the Filtlong options `--mean_q_weight` and `--length_weight`, respectively.
See also the [section in the Filtlong docs](https://github.com/rrwick/Filtlong#length-priority).
Further, reads are filtered by their length as above. The output is written to `fastq-ont/mysample+filtlongMB<m>,<q>,<l>.fastq`.

**`filtlongMB<m>,<q>,<l>,<n>`**  
As above, but the the minimum read length is explicitly specified by `n` and not by the global option `filtlong_min_read_length`:
The output is written to `fastq-ont/mysample+filtlongMB<m>,<q>,<l>,<n>.fastq`.

When using any of the Filtlong keywords in a folder name, they must be followed by an underscore, followed by the keyword for the assembler.

### Rasusa
The ONT reads can be randomly subsampled prior to the assembly.

The available keywords are:

**`rasusaMB<m>`**  
This will subsample the ONT reads to a total of `m` megabases.  
The output is written to `fastq-ont/mysample+rasusaMB<m>.fastq`.

When using any the Rasusa keyword in a folder name, it must be followed by an underscore, followed by the keyword for the assembler.

### Flye

Following keywords can be used to run the assembly with Flye:

**`flye`**  
Default assembly, which includes one round of internal polishing the assembly with the ONT reads.

**`flyeX`**  
Assembly with `X` rounds of internal polishing. Setting `X` to 0 disables polishing altogether.

**`flyehq`**  
Assembly for high-quality ONT reads using Flye option `--nano-hq` for ONT Guppy5+ in SUP mode, with one round of internal polishing.

**`flyehqX`**  
High-quality assembly, with `X` rounds of internal polishing. Setting `X` to 0 disables polishing altogether.


### Raven

Following keywords can be used to run the assembly with raven:

**`raven`**  
Default assembly, which includes two rounds of internal polishing with racon using the ONT reads.

**`ravenX`**  
Assembly with `X` rounds of internal polishing with racon. Setting `X` to 0 disables polishing altogether.

### Miniasm

Following keywords can be used to run the assembly with miniasm:

**`miniasm`**  
Default assembly. Miniasm does not do any polishing by itself.

### Unicycler

**`unicycler`**  
Unicycler does a hybrid assembly, i.e., both ONT and Illumina reads must be present in `fastq-ont` and `fastq-illumina`, respectively.

### Canu

**`canu`**  
The Canu assembler requires to know the genome size (in Megabases) beforehand, use Snakemake option: `--config genome_size=5.2` (e.g. for 5.2 Mb)  
NB: Canu can take a long time to complete the assembly, up to several hours!

### racon
Following keywords can be used to polish an assembly using ONT reads:

**`racon`**  
Polishing the assembly once.

**`raconX`**  
Run racon polishing iteratively `X` times.

### medaka

**`medaka`**  
Medaka polishes the assembly using the ONT reads, but also requires the name of
the Medaka model to be used, which depends on the flow cell and basecalling that were used for creating the reads.

The model name can either be set globally for all samples using the Snakemake configuration option `medaka_model`,
or by supplying a tab-separated file with two columns that maps sample names to medaka models using the Snakemake configuration option `map_medaka_model`.

Options are specified using snake make's `--config` parameter, for example:

```
snakemake /opt/software/ont-assembly-snake/Snakefile --cores 20 --config map_medaka_model=map_medaka.tsv
```
where `map_medaka.tsv` contains, for example, the two columns:
```
sample1     r941_min_high_g330
sample2     r941_min_high_g351
```

If no model is specified, Medaka will use the `r941_min_hac_g507` model by default.

### pilon

**`pilon`**  
Pilon polishes an assembly using Illumina reads, which must be located in the `fastq-illumina` folder.

### Polypolish

**`polypolish`**  
Polypolish polishes an assembly using Illumina reads, which must be located in the `fastq-illumina` folder.

### POLCA

**`polca`**  
POLCA polishes an assembly using Illumina reads, which must be located in the `fastq-illumina` folder.


### Homopolish

**`homopolish`**  
Homopolish does reference-based polishing based on one ore more provided reference genomes in FASTA format located in 
`references/NAME1.fa`, `references/NAME2.fa`, etc., where `NAME1` and `NAME2` can be any string.
Snakemake will create output files `...+homopolish/output_NAME1.fa`, `...+homopolish/output_NAME1.fa`, etc., containing the polished assemblies.

When using Homopolish, it must be the last keyword in the folder name.

### proovframe

**`proovframe`**  
Proovframe does reference-based polishing based on one ore more provided reference proteomes in FASTA format containing the amino acid sequences located in 
`references-protein/NAME1.faa`, `references-protein/NAME2.faa`, etc., where `NAME1` and `NAME2` can be any string.
Snakemake will create output files `...+proovframe/output_NAME1.fa`, `...+proovframe/output_NAME1.fa`, etc., containing the polished assemblies.

When using proovframe, it must be the last keyword in the folder name.

## Example
This example contains two samples with ONT sequencing reads and Illumina reads
for sample 2 only.  

For sample 1, the assembly should be done with Flye (including the default single round of
polishing), followed by polishing the assembly with racon (twice), medaka, and eventually Homopolish, which will use the E. coli genome in the file `references/Ecoli.faa`.  
In another assembly, we also want to filter the ONT reads of sample 1 to only include the highest quality reads up to a total of 500Mb
using Filtlong and apply the same assembly and polishing protocol.

Sample 2 should be assembled by raven including two internal polishing rounds,
followed by medaka and pilon polishing using the Illumina reads, and finally reference-based polishing with E. coli proteins using proovframe, which requires the file `references-protein/Ecoli.faa`.

We therefore create the folders and files as follows:
```
.
├── assemblies
│   ├── sample1+filtlongMB500_flye+racon2+medaka+homopolish
│   ├── sample1_flye+racon2+medaka+homopolish
│   ├── sample2_raven2+medaka+pilon+proovframe
├── fastq-illumina
│   ├── sample2_R1.fastq
│   └── sample2_R2.fastq
├── fastq-ont
│   ├── sample1.fastq
│   └── sample2.fastq
├── references
│   └── Ecoli.fa
└── references-protein
    └── Ecoli.faa
```

We also want to set the minimum read length threshold for Filtlong to 500nt and use the medaka model `r941_min_high_g351` for both samples.

Therefore, we run the workflow with:
```
snakemake -s /opt/software/ont-assembly-snake/Snakefile --use-conda --cores 20 --config medaka_model=r941_min_high_g351 filtlong_min_read_length=500
```

Snakemake will recursively handle the dependencies for each assembly,
and create folders for all intermediate steps automatically.
Additionally, a symlink is created for each output assembly in the `assemblies/` folder.

For the above example, the folders will look like this after running the workflow:
```
.
├── assemblies
│   ├── sample1+filtlongMB500_flye
│   ├── sample1+filtlongMB500_flye.fa -> sample1+filtlongMB500_flye/output.fa
│   ├── sample1+filtlongMB500_flye+racon2
│   ├── sample1+filtlongMB500_flye+racon2.fa -> sample1+filtlongMB500_flye+racon2/output.fa
│   ├── sample1+filtlongMB500_flye+racon2+medaka
│   ├── sample1+filtlongMB500_flye+racon2+medaka.fa -> sample1+filtlongMB500_flye+racon2+medaka/output.fa
│   ├── sample1+filtlongMB500_flye+racon2+medaka+homopolish
│   ├── sample1+filtlongMB500_flye+racon2+medaka+homopolishEcoli.fa -> sample1+filtlongMB500_flye+racon2+medaka+homopolish/output_Ecoli.fa
│   ├── sample1_flye
│   ├── sample1_flye.fa -> sample1_flye/output.fa
│   ├── sample1_flye+racon2
│   ├── sample1_flye+racon2.fa -> sample1_flye+racon2/output.fa
│   ├── sample1_flye+racon2+medaka
│   ├── sample1_flye+racon2+medaka.fa -> sample1_flye+racon2+medaka/output.fa
│   ├── sample1_flye+racon2+medaka+homopolish
│   ├── sample1_flye+racon2+medaka+homopolishEcoli.fa -> sample1_flye+racon2+medaka+homopolish/output_Ecoli.fa
│   ├── sample2_raven2
│   ├── sample2_raven2.fa -> sample2_raven2/output.fa
│   ├── sample2_raven2+medaka
│   ├── sample2_raven2+medaka.fa -> sample2_raven2+medaka/output.fa
│   ├── sample2_raven2+medaka+pilon
│   ├── sample2_raven2+medaka+pilon.fa -> sample2_raven2+medaka+pilon/output.fa
│   ├── sample2_raven2+medaka+pilon+proovframe
│   └── sample2_raven2+medaka+pilon+proovframeEcoli.fa -> sample2_raven2+medaka+pilon+proovframe/output_Ecoli.fa
├── fastq-illumina
│   ├── sample2_R1.fastq
│   └── sample2_R2.fastq
├── fastq-ont
│   ├── sample1.fastq
│   ├── sample1+filtlongMB500.fastq
│   └── sample2.fastq
├── references
│   └── Ecoli.fa
└── references-protein
    └── Ecoli.faa
```
(Not shown is the content of each subfolder in `assemblies/` and some auxiliary files.)

## score-assemblies

The assemblies generated by this workflow can be evaluated using the [score-assemblies](https://github.com/pmenzel/score-assemblies) workflow.
One can either download the score-assemblies repository separately and just run it in the same folder after ont-assembly-snake is done, see the [README](https://github.com/pmenzel/score-assemblies/blob/master/README.md) or
add the Snakemake command line parameter `--config run_score_assemblies=1` to the Snakemake call of ont-assembly-snake.



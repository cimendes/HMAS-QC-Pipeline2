# HMAS-QC-Pipeline2 (sample based)

An alternative pipeline for doing initial quality control of highly-multiplexed amplicon sequencing data. **This sample based approach takes demultiplexed raw reads for each sample, and generates output file in a separate folder for each sample**

 [the HMAS2 QC Pipeline](https://github.com/ncezid-biome/HMAS-QC-Pipeline2/tree/main)

## TOC
* [Description](#description)
* [Requirements](#requirements)
* [INSTALL](#install)
* [USAGE](#usage)
* [Workflow](#workflow)  
* [miscellaneous](#miscellaneous)  

## Description

This is a pipeline that performs quality control analysis on highly-multiplexed amplicon sequencing (HMAS) data.
The pipeline is implemented in nextflow with some python and shell scripts. The input of the pipeline is demultiplexed fastq files (gzipped or not) for each sample. If you have multiplexed fastq files, you can run bcl2fastq for demultiplexing. 
It provides 3 main outputs of interest (**for each sample**): 

1. A **fasta** file containing the high-quality representative unique sequences after cleaning
2. A **count_table** file containing the abundance information of the above fasta file   
~~**note:** _this count_table file is converted to a python pickle file (.pkl) for fast loading_~~  
**note:** _this count_table is becoming obsolete because we now embedded abundance information  in the seq_ID in the above fasta file. For example (size=551 is the abundance value for this particular sequence):_    
`>M03235:107:000000000-KPP6Y:1:1101:19825:4748=OG0001064primerGroup7=isolateD-3-M3235-23-014;size=551`  
3. A csv report file containing the statistics (mean read depth, successful primer counts, etc.) of the reads for this sample

For more information and to see visualizations describing the workflow, see [this folder containing visual documents](https://github.com/ncezid-biome/HMAS-QC-Pipeline2/tree/main/documents).

This pipeline has been designed and tested under Linux CentOS and Ubuntu platforms.  It has not been tested under Windows.

## Requirements

1. Python 3 or higher. Download python [here](https://www.python.org/downloads/). 


2. PEAR installed and/or on your path. Download PEAR [here](https://www.h-its.org/downloads/pear-academic/).


3. Cutadapt installed and/or on your path. Find cutadapt installation guide [here](https://cutadapt.readthedocs.io/en/stable/installation.html).

4. VSEARCH installed and/or on your path. Find more vsearch installtion guide [here](https://github.com/torognes/vsearch).

5. nextflow installed and/or on your path. Find more nextflow installation guide [here](https://www.nextflow.io/docs/latest/getstarted.html).

## INSTALL


1. Copy the Github repository to a folder  
`git clone https://github.com/ncezid-biome/HMAS-QC-Pipeline2.git` 




## USAGE

1. make sure your paired demultiplexed fastq files (for each sample) are under the **/HMAS-QC-Pipeline2/data** folder. And they have a `*_R{1,2}*.fastq.gz` pattern


2. create a  **output** folder under **/HMAS-QC-Pipeline2**   
_note_: the output folder contains all the subfolders (one for each sample). And each subfolder holds the _final.unique.fasta_, _final.count_table_, _samplename.csv_ and a few other intermediary files


3. If you haven't installed those required packages and if have access to Scicomp, you can module load all of them. Otherwise, you can create a conda env with our provided yaml file. For that, you will run the following:   
    1. `conda env create -n hmas -f bin/hmas.yaml` (if you have mamba installed, use `mamba env create` instead for speed)   
    2.  `conda activate hmas`  
>**Note**: We didn't use singularity container at this time because:  
> 1. I can't find some docker images (i.e. PEAR)
> 2. nextflow only allows 1 container image per process (and we need more than 1 for some processes)   
4. Run the following:  
`nextflow run hmas2.nf`    
**note:** if you have your input reads and/or your output folder in other than your current working directory, you can override those set in the nextflow script and provide them at the command line, for example:  
 `nextflow run hmas2.nf --reads YOUR_READS --outdir YOUR_OUTPUT`

   

## Workflow 

```mermaid
graph TD
    p0(Channel.fromFilePairs)
    p10[cutadapt]
    p11[concat_reads]
    p12[pair_merging]
    p13[quality_filtering]
    p14[dereplication]
    p15[denoising]
    p16([join])
    p17[search_exact]
    p18[make_count_table]
    p19((final count_table))
    p20((final unique.fasta))
    p21(combined_report)
    p22([collect])
    p23((combined_report.csv))

    p0 -->|paired_reads| p10
    p10 --> |sample.*.1.fastq, sample.*.2.fastq| p11
    p11 --> |sample.1.fastq, sample.2.fastq| p12
    p12 --> |sample.fastq| p13
    p13 --> |sample.fasta| p14
    p14 --> |unique.fasta| p15
    p15 --> |final.unique.fasta| p16
    p16 --> p17
    p17 --> |output.match.txt| p18
    p18 --> |final_count_table| p19
    p18 --> |report.csv| p22 --> p21 --> p23
    p15 --> p20

    %% Change the positioning
    %% Group the final output nodes
    subgraph Final Output 
        p19
        p20
        p23
    end
    %% Group the preprocessing nodes
    subgraph    preprocessing
        p0
        p10
        p11
        p12
    end
    %% Group the processing nodes
    subgraph          
        p13
        p14
        p15
        p16
        p17
        p18
        p21
        p22
    end

```  
**note:** the final.count_table and final.unique.fasta files are per sample based, while combined_report.csv is for the combined report for all the samples in the study.  

## Miscellaneous 

1.  **difference in sequence abundance distribution** (between previous mothur version and current hmas2 version)   
    - with mothur version, there is a significant peak at the very low end of sequence abundance. Please check out the visualization [this folder containing visual documents](https://github.com/ncezid-biome/HMAS-QC-Pipeline2/tree/sample_base/documents).    
    - And this difference comes from the additional **unoise_alpha** parameter in the denoising step of the hmas2 pipeline. 
    - > **alpha**: determines the threshold level of dissimilarity between frequent and infrequent reads for exclusion of infrequent reads.  
    (Denoising exploits the observation that a low-abundance sequence that is very similar to a 
high-abundance sequence is likely to be an error.)
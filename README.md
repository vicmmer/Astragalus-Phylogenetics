# Astragalus-Phylogenetics
Hello future Grillo Lab members, below I will document every step of the pipeline I used/created to process my Astragalus data. 
Most of the dependencies I used were already installed on the Smithsonian server I used to analyze my data, therefore I will not be listing the steps to download and install these dependencies although, where possible, I will link the website to those tools where download information must be found. 

## Downloading data 
Project code: PRJNA1108222
Download link: https://www.ncbi.nlm.nih.gov/sra/PRJNA1108222

## Step One: Trimming Reads
Dependencies required: Trimmomatic 
The following code was ran within a job file onto a server. It might be need to be modified for future users depending on whether they do it within a job file, on a server or locally. 

```
for seq in *_1.fastq.gz
do
echo $seq
seq2=${seq/_1\.fastq\.gz/_2\.fastq\.gz}
seq_tfp=${seq/_1\.fastq\.gz/_1_trimmed\.fastq\.gz}
seq_tfu=${seq/_1\.fastq\.gz/_1_trimmed_unpaired\.fastq\.gz}
seq_trp=${seq/_1\.fastq\.gz/_2_trimmed\.fastq\.gz}
seq_tru=${seq/_1\.fastq\.gz/_2_trimmed_unpaired\.fastq\.gz}
runtrimmomatic PE -threads 16 -phred33 $seq $seq2 $seq_tfp $seq_tfu $seq_trp $seq_tru ILLUMINACLIP:TruSeq3-PE.fa:2:30:10 LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:36  
done
```


## Step Two: Looking at sequence quality
Dependencies required: fastqc and multiqc. 
First, fastqc is run to get the quality of each sequence separately where the general command is: with $name being the name of each sample and _1 & _2 are the forward and reverse trimmed files 
```
fastqc $name*_1_trimmed_noAdapters.fastq
fastqc $name*_2_trimmed.noAdapters.fastq
```
Second, multiqc is run and we create a csv file with all the information. Multiqc will take as input the files generated from fastqc and create a csv file as well as html file with all of the sequence quality information. The command I used is: 


## Step Three:  Analyzing nuclear data
Depencencies required: HybPiper, raxml, Astral, figtree
### 
multiqc .  --export csv
```

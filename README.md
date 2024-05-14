# Astragalus-Phylogenetics
Hello future Grillo Lab members, below I will document every step of the pipeline I used/created to process my Astragalus data. 
Most of the dependencies I used were already installed on the Smithsonian server I used to analyze my data, therefore I will not be listing the steps to download and install these dependencies although, where possible, I will link the website to those tools where download information must be found. 

## Downloading data 
Project code: PRJNA1108222
Download link: https://www.ncbi.nlm.nih.gov/sra/PRJNA1108222

## Trimming Reads
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

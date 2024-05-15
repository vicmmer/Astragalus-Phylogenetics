# Astragalus-Phylogenetics
Hello future Grillo Lab members, below I will document every step of the pipeline I used/created to process my Astragalus data. 
Most of the dependencies I used were already installed on the Smithsonian server I used to analyze my data, therefore I will not be listing the steps to download and install these dependencies although, where possible, I will link the website to those tools where download information must be found. 

## Downloading data 
Project code: PRJNA1108222
Download link: https://www.ncbi.nlm.nih.gov/sra/PRJNA1108222

## Step One: Trimming Reads
Dependencies required: Trimmomatic (https://github.com/usadellab/Trimmomatic)
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
Dependencies required: fastqc (https://github.com/s-andrews/FastQC) and multiqc (https://github.com/MultiQC/MultiQC)
First, fastqc is run to get the quality of each sequence separately where the general command is: with $name being the name of each sample and _1 & _2 are the forward and reverse trimmed files 
```
fastqc $name*_1_trimmed_noAdapters.fastq
fastqc $name*_2_trimmed.noAdapters.fastq
```
Second, multiqc is run and we create a csv file with all the information. Multiqc will take as input the files generated from fastqc and create a csv file as well as html file with all of the sequence quality information. The command I used is: 
```
multiqc .  --export csv
```

## Step Three:  Analyzing nuclear data
Depencencies required: HybPiper (https://github.com/mossmatters/HybPiper/wiki), raxml (https://github.com/amkozlov/raxml-ng/wiki), Astral (https://github.com/smirarab/ASTRAL), figtree (https://tree.bio.ed.ac.uk/software/figtree/)
The first thing to do is installing Hybpiper and preparing files. More information on how to use Hybpiper and try with sample data can be found on: (https://github.com/mossmatters/HybPiper/wiki/Tutorial)
I recommend creating a new directory with the following : 
1. Trimmed fastq files (one forward one reverse for each sample). Example format: name*_2_trimmed_noAdapters.fastq and name*_2_trimmed_noAdapters.fastq
2. A namelist.txt file that you can create with the following command:
```
ls *_trimmed_noAdapters.fastq | sed 's/\(_[12]_trimmed_noAdapters\.fastq\)//' | sort -u > namelist.txt
```
3. A targetfile. My target Astragalus_targefile.fasta is available to download from this repository. 

### a) First hypiper command: nuclear assembly. 
The following code is for the hybpiper assembly: 
```
while read name; 
do hybpiper assemble -t_dna Astragalus_targetfile.fasta -r $name_1_trimmed.fastq $name_2_trimmed.fastq  --prefix $name --bwa;
done < namelist.txt
```
### b) Second hyhbpiper command: creating a stats summary 
By running the following command, you will generate a stats table with information such as number of reads, reads mapped, total bases recovered, etc: 
```
hybpiper stats -t_dna Astragalus_targetfile.fasta gene namelist.txt
```

### c) Third hybpiper command : visualizing results 
The following command takes the stats summary from b) and creates a visual summary/heatmap showing the gene recovery for each gene within each sample
```
hybpiper recovery_heatmap seq_lengths.tsv
```

### d) Fourth hybpiper command: retrieving sequences 
Parameters can be changed in the following command based on whether the user wants to retrieve gene or supercontig (might include more information about the flanking regions). Generally, people retrieve the genes 
```
hybpiper retrieve_sequences dna -t_dna Astragalus_targetfile.fasta --sample_names namelist.txt
```
### e) Fifth hybpiper command: retrieving paralogs 

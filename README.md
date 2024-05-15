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
Analysing paralogs is important. Good practice is removing them from further analysis, which I did. I retrieved and recognized 145 paralogs using the following command: 
```
hybpiper paralog_retriever namelist.txt -t_dna Astragalus_targetfile.fasta
```
At this point, the hybpiper step is done, what comes next is to trim the gene files, and create gene trees for each of the genes (minus the paralogous genes). For this, I created an additional folder that contains the genes that are NOT found in the paralogs_above_threshold_report.txt file. Within this folder I will perform the following steps: 

### Multiple sequence alignment with MAFFT 
After filtering out the genes that were flagged paralogous, we need to align these recovered FNAs from each gene using the following command: 
```
mafft --auto $1.FNA > $1.aligned.FNA
```
### Trim gene trees with TrimAL 
TrimAL (https://vicfero.github.io/trimal/) is another command line based tool that removes ambiguous, poorly aligned regions of the gene that might introduce noise and decrease the reliability of phylogenetic analysis. 
I trimmed every single one of the recovered and aligned gene files from mafft using the following bash script: 
```
#!/bin/bash

# Directory where your .aligned.fasta files are located
input_dir="/mnt/c/Users/omatg/OneDrive/Desktop/trimal/Astragalus" #change to your directory

output_dir="/mnt/c/Users/omatg/OneDrive/Desktop/trimal/Astragalus/trimal_output" #output directory 

# Iterate over .aligned.fasta files in the directory
for file in "$input_dir"/*.aligned.fasta; do
    if [ -e "$file" ]; then
        # Extract the gene name from the filename
        gene_name=$(basename "$file" .aligned.fasta)

        # Construct the paths for output files in the specified directory
        output_fasta="${output_dir}/${gene_name}.aligned.trimmed.fasta"
        output_html="${output_dir}/${gene_name}.aligned.trimmed.html"

        # Run the trimal command and direct output to the specified paths
        trimal -in "$file" -out "$output_fasta" -htmlout "$output_html" -automated1

        echo "Processed $gene_name"
    fi
done
```

### Create gene trees with raxml: 
After aligning, and trimming, we are ready to make gene trees. The code below is what I had on my raxml_all.job file which i executed with the following command

```
name=$1

raxmlHPC -f a -x 12345 -p 12345 -# 1000 -m GTRGAMMA -s ${name}.aligned.trimmed.fasta -n ${name}_output_tree
#Paramatere explanation:
#-f a : tells RAxML to conduct a rapid bootstrap analysis and search for the best scoring ML tree in one run
#-x 12345: sets the random number seed for bootstra analysis and ensures that bootstrap sampling is reproducible
#-p 12345: sets another random seed that allows the program to start tree inference from a different initial tree for each new run.
# -# 1000: specified boostrap replicates
#-m GTRGAMMA: selects model of nucleotido or amino acid substitution and the rate heterogeneity model 
```
The job file above was executed directly on the command line with the following command: 
```
while read name; do   qsub -o $name.log raxml_all.job $name; done < genenamelist.txt
#where genenamelist.txt is a text list containing all the genes that were kept after the paralog filtering
```
### Concatanate all species trees into one file: 
Astral in the next step, will analyze all the gene trees and create a species tree using that data. but first we need to concatante these files into one using: 
```
cat *output_tree > concatenated_trees.tre
```
### Concatanate trees with Astral 
Astral will generate a species tree. Use following code: 
```
java -jar astral.5.7.8.jar -i concatenated_trees.tre -o astragalus.astral_consensus.tre
```


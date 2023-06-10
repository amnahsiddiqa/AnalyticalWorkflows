
## Checking Strandedness using RSeQC

I will take a step back and mention how I created GENOME index using STAR and arrive at
creating bam files.


Step1: 

Download the genome fasta and gtf/gff files. It is important to note that the chromosome names of the genome (FASTA)
file and the GTF file needs to be identical. Thus, Ensembl and Gencode GTF files should not be mixed (unless the 
chromosome GTF names have been fixed

```
wget https://ftp.ensembl.org/pub/release-109/fasta/homo_sapiens/dna/Homo_sapiens.GRCh38.dna.primary_assembly.fa.gz
wget https://ftp.ensembl.org/pub/release-109/gtf/homo_sapiens/Homo_sapiens.GRCh38.109.gtf.gz
#unzip
gunzip Homo_sapiens.GRCh38.109.gtf.gz
gunzip Homo_sapiens.GRCh38.dna.primary_assembly.fa.gz
```

Step2: 
Create  genome Index using STAR
```

#!/bin/bash
#SBATCH --job-name=stargenomeindex
#SBATCH --mail-type=END
#SBATCH -p compute
#SBATCH -q batch
#SBATCH -t 72:00:00
#SBATCH --mem=500000
#SBATCH -o stargenomeindex.%j.out # STDOUT
#SBATCH -e stargenomeindex.%j.err # STDERR



STAR --runMode genomeGenerate \
     --genomeDir GenomeIndex \
     --genomeFastaFiles Homo_sapiens.GRCh38.dna.primary_assembly.fa \
     --sjdbGTFfile Homo_sapiens.GRCh38.109.gtf \
     --runThreadN 60 \
     --sjdbOverhang 149# this should be number of bp -1 ; see the manual for details and alexdobin/STAR#931 (comment)
```

 Step3: 

 Align the reads using the index we just created. 

 ```
 #!/bin/bash
 #SBATCH --job-name=Trimgalore
 #SBATCH --mail-type=END
 #SBATCH -p compute
 #SBATCH -q batch
 #SBATCH -t 24:23:00
 #SBATCH --mem=50000
 #SBATCH -o fastqc.%j.out # STDOUT
 #SBATCH -e fastqc.%j.err # STDERR
 #SBATCH --cpus-per-task=16  # Number of cores

 #source activate rnaseq  # Assuming "rnaseq" is the name of your Conda environment
 module load singularity 

 genomedir="/pathto/star_reference/GenomeIndex"
 I_PATH="pathtotrimmedfastqfiles/results_trimmed"
 output_dir="/pathtobamfiles/alignment_on_trimmedfiles_results"


 csv_file="/listoffilesincsv/sorted_fastq_list_trimmedvalidated.csv"
 start_file="$1"
 end_file="$2"



 while IFS= read -r line; do
     if [[ $line == *_R1_001_val_1.fq.gz ]]; then
         i="${line%_R1_001_val_1.fq.gz}"
         echo "Processing files: ${i}_R1_001_val_1.fq.gz, ${i}_R2_001_val_2.fq.gz"
         
         pairedend1="${i}_R1_001_val_1.fq.gz"
         pairedend2="${i}_R2_001_val_2.fq.gz"
         
         # Debug: Check if the files exist and have read permission
         ls -l "$I_PATH/$pairedend1"
         ls -l "$I_PATH/$pairedend2"

         echo "Aligning ${i}..."
         STAR --runThreadN 8 \
             --genomeDir $genomedir/ \
             --readFilesIn $I_PATH/$pairedend1 $I_PATH/$pairedend2 \
             --readFilesCommand zcat \
             --outSAMtype BAM SortedByCoordinate \
             --quantMode TranscriptomeSAM GeneCounts \ 
             --outSAMunmapped Within \
             --twopassMode Basic \
             --outFilterMultimapNmax 1 \
             --runMode alignReads \
             --outFileNamePrefix $output_dir/${i}_
         echo "Alignment for ${i} is done."
     fi
 done < <(sed -n "${start_file},${end_file}p" "$csv_file")

 ``` 
Note: 
-   --quantMode TranscriptomeSAM GeneCounts: could use raw gene count after assessing strandedness from here too ; see STAR manuual https://github.com/alexdobin/STAR/blob/master/doc/STARmanual.pdf section 8 page 18 ;
-  --outSAMtype BAM SortedByCoordinate  can keep it unsorted too; this sort is by location 

## Making a bed file for RSeQC

- To run RseqC we need a bed file and bam file; bam files we have already created ; below is bed file creation ; 
- conda install -c bioconda ucsc-genepredtobed
- conda install -c bioconda ucsc-gtftogenepred


```

(base) siddia@MLG-JGM353 ~ % conda create --name forstrandedness         
Collecting package metadata (current_repodata.json): done
Solving environment: done


==> WARNING: A newer version of conda exists. <==
  current version: 23.1.0
  latest version: 23.5.0

Please update conda by running

    $ conda update -n base -c defaults conda

Or to minimize the number of packages updated during conda update use

     conda install conda=23.5.0



## Package Plan ##

  environment location: /Users/siddia/opt/anaconda3/envs/forstrandedness



Proceed ([y]/n)? y

Preparing transaction: done
Verifying transaction: done
Executing transaction: done
#
# To activate this environment, use
#
#     $ conda activate forstrandedness
#
# To deactivate an active environment, use
#
#     $ conda deactivate

(base) siddia@MLG-JGM353 ~ % conda activate forstrandedness

(forstrandedness) siddia@MLG-JGM353 ~ % conda install -c bioconda ucsc-genepredtobed
Collecting package metadata (current_repodata.json): done
Solving environment: failed with initial frozen solve. Retrying with flexible solve.
Solving environment: failed with repodata from current_repodata.json, will retry with next repodata source.
Collecting package metadata (repodata.json): done
Solving environment: done


==> WARNING: A newer version of conda exists. <==
  current version: 23.1.0
  latest version: 23.5.0

Please update conda by running

    $ conda update -n base -c defaults conda

Or to minimize the number of packages updated during conda update use

     conda install conda=23.5.0



## Package Plan ##

  environment location: /Users/siddia/opt/anaconda3/envs/forstrandedness

  added / updated specs:
    - ucsc-genepredtobed


The following packages will be downloaded:

    package                    |            build
    ---------------------------|-----------------
    ucsc-genepredtobed-366     |       h1341992_0         2.1 MB  bioconda
    ------------------------------------------------------------
                                           Total:         2.1 MB

The following NEW packages will be INSTALLED:

  ucsc-genepredtobed bioconda/osx-64::ucsc-genepredtobed-366-h1341992_0 
  zlib               pkgs/main/osx-64::zlib-1.2.13-h4dc903c_0 


Proceed ([y]/n)? y


Downloading and Extracting Packages
                                                                                                                      
Preparing transaction: done
Verifying transaction: done
Executing transaction: done
(forstrandedness) siddia@MLG-JGM353 ~ % conda install -c bioconda ucsc-gtftogenepred
Collecting package metadata (current_repodata.json): done
Solving environment: failed with initial frozen solve. Retrying with flexible solve.
Solving environment: failed with repodata from current_repodata.json, will retry with next repodata source.
Collecting package metadata (repodata.json): done
Solving environment: done


==> WARNING: A newer version of conda exists. <==
  current version: 23.1.0
  latest version: 23.5.0

Please update conda by running

    $ conda update -n base -c defaults conda

Or to minimize the number of packages updated during conda update use

     conda install conda=23.5.0



## Package Plan ##

  environment location: /Users/siddia/opt/anaconda3/envs/forstrandedness

  added / updated specs:
    - ucsc-gtftogenepred


The following packages will be downloaded:

    package                    |            build
    ---------------------------|-----------------
    ucsc-gtftogenepred-366     |       h1341992_0         2.1 MB  bioconda
    ------------------------------------------------------------
                                           Total:         2.1 MB

The following NEW packages will be INSTALLED:

  ucsc-gtftogenepred bioconda/osx-64::ucsc-gtftogenepred-366-h1341992_0 


Proceed ([y]/n)? y


Downloading and Extracting Packages
                                                                                                                      
Preparing transaction: done
Verifying transaction: done
Executing transaction: done

(forstrandedness) siddia@MLG-JGM353 ~ % pwd
/Users/siddia
(forstrandedness) siddia@MLG-JGM353 ~ % cd /Users/siddia/Documents/RNAseq/2023/create_bedfilefromgtf/June082023 
(forstrandedness) siddia@MLG-JGM353 June082023 % wget https://ftp.ensembl.org/pub/release-109/gtf/homo_sapiens/Homo_sapiens.GRCh38.109.gtf.gz                  
  
--2023-06-10 12:51:15--  https://ftp.ensembl.org/pub/release-109/gtf/homo_sapiens/Homo_sapiens.GRCh38.109.gtf.gz
Resolving ftp.ensembl.org (ftp.ensembl.org)... 193.62.193.139
Connecting to ftp.ensembl.org (ftp.ensembl.org)|193.62.193.139|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 54258835 (52M) [application/x-gzip]
Saving to: 'Homo_sapiens.GRCh38.109.gtf.gz'

Homo_sapiens.GRCh38.109.gtf.g 100%[===============================================>]  51.75M   568KB/s    in 86s     

2023-06-10 12:52:41 (617 KB/s) - 'Homo_sapiens.GRCh38.109.gtf.gz' saved [54258835/54258835]

(forstrandedness) siddia@MLG-JGM353 June082023 % gzip -cd Homo_sapiens.GRCh38.109.gtf.gz |\
  gtfToGenePred /dev/stdin /dev/stdout |\
  genePredToBed /dev/stdin /dev/stdout |\
  head
1	1471764	1497848	ENST00000673477	0	+	1471884	1495817	0	16	325,77,102,60,70,166,70,156,57,126,125,52,71,168,109,2364,	0,5509,6879,7284,9102,10373,10780,13251,14017,14345,14779,16098,17439,18492,18798,23720,
1	1478025	1497848	ENST00000472194	0	+	1497848	1497848	0	14	720,60,70,166,70,156,57,126,125,52,71,168,109,2364,	0,1023,2841,4112,4519,6990,7756,8084,8518,9837,11178,12231,12537,17459,
1	1479048	1482662	ENST00000378736	0	+	1482662	1482662	0	4	60,70,166,118,	0,1818,3089,3496,
1	1483484	1496202	ENST00000485748	0	+	1496202	1496202	0	10	1687,57,126,125,52,71,120,168,109,718,	0,2297,2625,3059,4378,5719,6207,6772,7078,12000,
1	1484568	1496201	ENST00000474481	0	+	1496201	1496201	0	5	603,454,125,1468,717,	0,1213,1975,4635,10916,
1	1471783	1496201	ENST00000308647	0	+	1471884	1486666	0	14	306,77,42,38,70,156,57,126,125,52,71,168,109,717,	0,5490,9083,10482,10761,13232,13998,14326,14760,16079,17420,18473,18779,23701,
1	182695	184174	ENST00000624431	0	+	184174	184174	0	5	51,85,78,162,194,	0,436,798,1044,1285,
1	2581559	2584533	ENST00000424215	0	+	2584533	2584533	0	3	91,126,409,	0,1810,2565,
1	3069167	3434342	ENST00000511072	0	+	3069259	3433689	0	16	129,350,51,138,103,208,148,154,1417,88,170,78,170,175,237,666,	0,116957,174919,315981,327323,333623,335571,336327,342216,345392,348660,349499,356413,356883,361704,364509,
1	3069182	3186591	ENST00000607632	0	+	3186591	3186591	0	2	114,467,	0,116942,
(forstrandedness) siddia@MLG-JGM353 June082023 % gzip -cd Homo_sapiens.GRCh38.109.gtf.gz | \
  gtfToGenePred /dev/stdin /dev/stdout | \
  genePredToBed /dev/stdin /dev/stdout > Homo_sapiens.GRCh38.109.bed

(forstrandedness) siddia@MLG-JGM353 June082023 % wget https://ftp.ensembl.org/pub/release-108/gtf/homo_sapiens/Homo_sapiens.GRCh38.108.gtf.gz                  
  
--2023-06-10 12:54:40--  https://ftp.ensembl.org/pub/release-108/gtf/homo_sapiens/Homo_sapiens.GRCh38.108.gtf.gz
Resolving ftp.ensembl.org (ftp.ensembl.org)... 193.62.193.139
Connecting to ftp.ensembl.org (ftp.ensembl.org)|193.62.193.139|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 54107597 (52M) [application/x-gzip]
Saving to: 'Homo_sapiens.GRCh38.108.gtf.gz'

Homo_sapiens.GRCh38.108.gtf.gz           100%[===============================================================================>]  51.60M   640KB/s    in 83s     

2023-06-10 12:56:04 (633 KB/s) - 'Homo_sapiens.GRCh38.108.gtf.gz' saved [54107597/54107597]

(forstrandedness) siddia@MLG-JGM353 June082023 % gzip -cd Homo_sapiens.GRCh38.108.gtf.gz | \                                                
  gtfToGenePred /dev/stdin /dev/stdout | \
  genePredToBed /dev/stdin /dev/stdout > Homo_sapiens.GRCh38.108.bed

(forstrandedness) siddia@MLG-JGM353 June082023 % 

```

## Use Infer.experiment.py  to check strandedness 

- Install RseQC ; see their website https://rseqc.sourceforge.net; pip or conda was fine ; 
```infer_experiment.py -r Homo_sapiens.GRCh38.108.bed -i 90075_day0_R_3_C8_GT23-06957_CGTTAGAA-GACCTGAA_S248_L004_Aligned.sortedByCoord.out.bam
GACCTGAA_S248_L004_Aligned.sortedByCoord.out.bam'
Reading reference gene model Homo_sapiens.GRCh38.108.bed ... Done
Loading SAM/BAM file ... Total 200000 usable reads were sampled

This is PairEnd Data
Fraction of reads failed to determine: 0.3369
Fraction of reads explained by "1++,1--,2+-,2-+": 0.0396
Fraction of reads explained by "1+-,1-+,2++,2--": 0.6235
(base) [siddia@sumner-log1 NEWSTARREFRENCEGENOME]$
```


It means you have a standard (dUTP-based) strand-specific library. 
If you want to use featureCounts, you'll want the -s 2 setting. 
For HTSeq-count it's --strand reverse.


Notes:
- MultiQC has module RseQC for easier use too taht i would prefere to use. (https://github.com/ewels/MultiQC/blob/master/docs/modules/rseqc.md)

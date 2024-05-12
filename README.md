# applied-microbiome-analysis-course

## **1. Installing QIIME2**
   
```
conda install wget      
wget https://data.qiime2.org/distro/core/qiime2-2023.2-py38-osx-conda.yml
conda env create -n qiime2-2023.2 --file qiime2-2023.2-py38-osx-conda.yml
conda activate qiime2-2023.2 
```
## **2. Variable identification**
```
input_reads=/Users/your/path/to/folder/qiime_test
manifest=/Users/your/path/to/folder/qiime_test/manifest.tmp

output_qza=/Users/your/path/to/folder/qiime_formatted/seq.qza
classifier=/Users/your/path/to/folder/silva138_AB_V4_classifier.qza
```
File **manifest.tmp** should look like this:
```
sample-id     	absolute-filepath
SRR21066812	/Users/your/path/to/fastq/qiime_test/SRR28803612.fastq
SRR21066823	/Users/your/path/to/fastq/qiime_test/SRR28803626.fastq
SRR21066834	/Users/your/path/to/fastq/qiime_test/SRR28803646.fastq
SRR21066845	/Users/your/path/to/fastq/qiime_test/SRR28803656.fastq
SRR21066856	/Users/your/path/to/fastq/qiime_test/SRR28803668.fastq
```
## **3. Creating seq.qza**

The command is used to convert input data files into Qiime2 artifacts, which are compressed folders that contain the data and metadata about the data. This process is necessary because Qiime2 requires all data to be in the form of artifacts in order to track and log the data through the system.

```
qiime tools import --type 'SampleData[SequencesWithQuality]' --input-path $manifest --output-path $output_qza --input-format SingleEndFastqManifestPhred33V2
```
## **4. Crop and clusterize with DADA2 or DEBLUR**

**DEBLUR:**
Positive mode - keeps only sequences similar to a reference database (by default known 16S sequences). SortMeRNA is used, and any sequence with an e-value <= 10 is retained.
```
# DEBLUR
qiime deblur denoise-16S --i-demultiplexed-seqs $output_qza --p-trim-length 249 --p-sample-stats --o-representative-sequences "./denoise_out/deblur-rep-seqs.qza" --o-table "./denoise_out/deblur-table.qza" --o-stats "./denoise_out/deblur-stats.qza"
```
```
# DADA2
qiime dada2 denoise-single --i-demultiplexed-seqs $output_qza --p-trunc-len 249 --p-trunc-q 10 --o-table "denoise_out/feature_table.qza" --o-representative-sequences "denoise_out/rep_seqs.qza" --o-denoising-stats "denoise_out/stats.qza"
```
Then we can get .qzv file and use [website](https://view.qiime2.org/) to visualize it.
```
qiime feature-table summarize --i-table ./denoise_out/deblur-table.qza --o-visualization ./denoise_out/deblur-table.qzv
```
Or we can get .fasta file of representative sequences and .csv statistics file:
```
# Variable identification
feature_table="/Users/your/path/to/folder/denoise_out/deblur-table.qza"
rep_seqs_qza="/Users/your/path/to/folder/denoise_out/deblur-rep-seqs.qza"

# EXPORT REPRESENTATIVE SEQUENCES AS .FASTA
qiime tools export --input-path $rep_seqs_qza --output-path "denoise_out/"
rep_seqs="denoise_out/dna-sequences.fasta"

# EXPORT DENOISING STATS
qiime tools export --input-path "denoise_out/deblur-stats.qza" --output-path "denoise_out/"
deblur_stats="denoise_out/stats.csv"
```
Thats how .csv statistics file looks like:

| Sample ID | Reads Raw | Unique Reads Derep | Reads Derep | Unique Reads Deblur | Reads Deblur | Unique Reads Hit Artifact | Reads Hit Artifact | Unique Reads Chimeric | Reads Chimeric | Unique Reads Hit Reference | Reads Hit Reference | Unique Reads Missed Reference | Reads Missed Reference |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| SRR21066812 | 103110 | 5401 | 62793 | 559 | 40134 | 148 | 302 | 101 | 215 | 272 | 39550 | 0 | 0 |
| SRR21066823 | 79182 | 4461 | 48778 | 625 | 33066 | 64 | 128 | 152 | 294 | 273 | 32376 | 0 | 0 |
| SRR21066834 | 88282 | 3660 | 28384 | 454 | 14627 | 87 | 180 | 67 | 105 | 217 | 14123 | 0 | 0 |
| SRR21066845 | 86269 | 5091 | 52309 | 595 | 32426 | 52 | 108 | 146 | 341 | 266 | 31743 | 0 | 0 |
| SRR21066856 | 80441 | 4562 | 50004 | 578 | 31581 | 58 | 119 | 92 | 157 | 275 | 31020 | 0 | 0 |

## **5. Classify representative sequences**

We will use --p-confidence {0.94}
```
# CLASSIFY REPRESENTATIVE SEQUENCES WITH --p-confidence {0.94}
mkdir taxonomy
qiime feature-classifier classify-sklearn --i-classifier $classifier --i-reads $rep_seqs_qza --o-classification "taxonomy/taxonomy.qza" --p-confidence 0.94
```
```
# EXPORT OUTPUT TABLE
qiime tools export --input-path "taxonomy/taxonomy.qza" --output-path "taxonomy"
```
Thats how taxonomy.tsv file looks like (the part of it):

| Feature ID | Taxon | Confidence |
| --- | --- | --- |
| 36254343960732d5f49267d8a8eaaafe | d__Bacteria; p__Firmicutes; c__Clostridia; o__Lachnospirales; f__Lachnospiraceae | 0.9999999748431286 |
| 028a44ca991af7d75a61868614bf424f | d__Bacteria; p__Firmicutes; c__Clostridia; o__Lachnospirales; f__Lachnospiraceae; g__Blautia | 0.9851119662521896 |
| 85a3d09075b38d10f128fb93c2405e30 | d__Bacteria; p__Bacteroidota; c__Bacteroidia; o__Bacteroidales; f__Prevotellaceae; g__Prevotellaceae_Ga6A1_group; s__uncultured_bacterium | 0.9959633605480437 |
| 538ac848d26ce29136cf56fc6439dda3 | d__Bacteria; p__Firmicutes; c__Clostridia; o__Lachnospirales; f__Lachnospiraceae; g__Fusicatenibacter | 0.9997030935376512 |

## **6. Rarefy and alpha-diversity**

First we rarefy and leave 10000 values for each sample. Then we count the alpha diversity.

**Alpha diversity** in the microbiota is a measure of the diversity of the microbiota within a single sample or specimen. It reflects the variation in microbiota composition within a single sample, that is, the number and diversity of microorganisms present in a particular sample.

> To calculate this, we use the chao1 index.

```
#RAREFY and ALPHA
mkdir rarefied
qiime feature-table rarefy --i-table "denoise_out/deblur-table.qza" --p-sampling-depth 10000 --o-rarefied-table "rarefied/otus_rar_10K.qza"
qiime diversity alpha --i-table "rarefied/otus_rar_10K.qza" --p-metric "chao1" --o-alpha-diversity "rarefied/alpha_chao.qza"
qiime tools export --input-path "rarefied/alpha_chao.qza" --output-path "rarefied/alpha_chao.tsv" --output-format "AlphaDiversityFormat"
```

Make readable taxa tables
```
mkdir otus
qiime tools export --input-path "rarefied/otus_rar_10K.qza" --output-path "otus"

Exported rarefied/otus_rar_10K.qza as BIOMV210DirFmt to directory otus
```

Consider representation to **species level (7)**:
```
qiime taxa collapse --i-table "rarefied/otus_rar_10K.qza" --i-taxonomy "taxonomy/taxonomy.qza" --p-level 7 --o-collapsed-table "otus/collapse_7.qza"
qiime tools export --input-path "otus/collapse_7.qza" --output-path "otus/summarized_taxa"
biom convert -i "otus/summarized_taxa/feature-table.biom" -o "otus/summarized_taxa/otu_table_L7.txt" --to-tsv
```
| OTU ID | SRR21066834 | SRR21066845 | SRR21066823 | SRR21066856 | SRR21066812 |
| --- | --- | --- | --- | --- | --- |
| d__Bacteria;p__Firmicutes;c__Clostridia;o__Lachnospirales;f__Lachnospiraceae;__;__ | 1284.0 | 303.0 | 622.0 | 624.0 | 650.0 |
| d__Bacteria;p__Firmicutes;c__Clostridia;o__Lachnospirales;f__Lachnospiraceae;g__Blautia;__ | 1199.0 | 447.0 | 766.0 | 721.0 | 801.0 |
| d__Bacteria;p__Bacteroidota;c__Bacteroidia;o__Bacteroidales;f__Prevotellaceae;g__Prevotellaceae_Ga6A1_group;s__uncultured_bacterium | 457.0 | 303.0 | 467.0 | 579.0 | 534.0 |
| d__Bacteria;p__Firmicutes;c__Clostridia;o__Lachnospirales;f__Lachnospiraceae;g__Fusicatenibacter;__ | 443.0 | 130.0 | 197.0 | 240.0 | 238.0 |

Then we can get .qzv file and use [website](https://view.qiime2.org/) to visualize taxonomy.qza file.

```
qiime taxa barplot --i-table rarefied/otus_rar_10K.qza --i-taxonomy taxonomy/taxonomy.qza --o-visualization taxonomy/taxa-bar-plots.qzv 
```
<img width="966" alt="Снимок экрана 2024-05-12 в 14 53 25" src="https://github.com/natsyam/applied-microbiome-analysis-course/assets/146081042/c1b7ac3c-c2a2-43db-9126-53629eb66888">

sources:
https://telatin.github.io/microbiome-bioinformatics/Metabarcoding-deblur/

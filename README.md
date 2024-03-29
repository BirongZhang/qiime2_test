# data processing ########################################################
```
#Importing data:Multiplexed single-end FASTQ with barcodes in sequence

qiime tools import \
  --type MultiplexedSingleEndBarcodeInSequence \
  --input-path PGM20161121.fastq.gz \
  --output-path multiplexed-seqs-3.qza


#Demultiplex the reads

qiime cutadapt demux-single \
  --i-seqs multiplexed-seqs-3.qza \
  --m-barcodes-file human_oral_3_m_map.txt \
  --m-barcodes-column BarcodeSequence \
  --p-error-rate 0 \
  --o-per-sample-sequences demultiplexed-seqs-3.qza \
  --o-untrimmed-sequences untrimmed-3.qza \
  --verbose

#visualizating trimmed-seqs.qza

qiime demux summarize \
  --i-data demultiplexed-seqs-3.qza \
  --o-visualization demultiplexed-seqs-3.qzv


#Trim adapters+primers from demulitiplexed reads

qiime cutadapt trim-single \
  --i-demultiplexed-sequences demultiplexed-seqs-3.qza \
  --p-front GATGTGCCAGCMGCCGCGGTAA \
  --p-error-rate 0 \
  --p-minimum-length 200 \
  --o-trimmed-sequences trimmed-seqs-3.qza \
  --verbose


#visualizating trimmed-seqs.qza

qiime demux summarize \
  --i-data trimmed-seqs-3.qza \
  --o-visualization trimmed-seqs-3.qzv


#Sequence quality control and feature table construction

nohup qiime dada2 denoise-single \
  --p-n-threads 16 \
  --i-demultiplexed-seqs trimmed-seqs-3.qza \
  --p-trim-left 0 \
  --p-trunc-len 233 \
  --o-representative-sequences rep-seqs-3.qza \
  --o-table table-3.qza \
  --o-denoising-stats stats-3.qza

#visualizating stats-3.qza

qiime metadata tabulate \
  --m-input-file stats-3.qza \
  --o-visualization stats-3.qzv

# merge data ########################################################
#merge feature-table

qiime feature-table merge \
  --i-tables table-1.qza \
  --i-tables table-2.qza \
  --i-tables table-3.qza \
  --o-merged-table table.qza

#merge-seqs feature-table

qiime feature-table merge-seqs \
  --i-data rep-seqs-1.qza \
  --i-data rep-seqs-2.qza \
  --i-data rep-seqs-3.qza \
  --o-merged-data rep-seqs.qza


#feature-table summarize

qiime feature-table summarize \
  --i-table table.qza \
  --o-visualization table.qzv \
  --m-sample-metadata-file merge_map.txt

#Representative sequences

qiime feature-table tabulate-seqs \
--i-data rep-seqs.qza \
--o-visualization rep_seqs.qzv
```


# data filtering ########################################################
```
#Filter features that only appear in a single sample

qiime feature-table filter-features \
--i-table table.qza \
--p-min-samples 2 \
--o-filtered-table table1.qza

qiime feature-table summarize \
  --i-table table1.qza \
  --o-visualization table1.qzv 


#Filter features by reads that appears 10 times or fewer across all samples

qiime feature-table filter-features \
--i-table table1.qza \
--p-min-frequency 10 \
--o-filtered-table table2.qza

qiime feature-table summarize \
  --i-table table2.qza \
  --o-visualization table2.qzv 

 
  
# Taxonomic analysis ########################################################
########### Taxonomic analysis---SILVA_132_QIIME_release#########
mkdir taxonomy
cd taxonomy

#assign taxonomy to features

mkdir SILVA
cd SILVA

#Get SILVA database

wget https://www.arb-silva.de/fileadmin/silva_databases/qiime/Silva_132_release.zip
unzip Silva_132_release.zip
rm Silva_132_release.zip


#Read database into QIIME2

cd SILVA_132_QIIME_release

#The sequences

qiime tools import \
--type 'FeatureData[Sequence]' \
--input-path rep_set/rep_set_16S_only/99/silva_132_99_16S.fna \
--output-path 99_otus_16S.qza

#The taxonomy strings

qiime tools import \
--type 'FeatureData[Taxonomy]' \
--input-format HeaderlessTSVTaxonomyFormat \
--input-path taxonomy/16S_only/99/majority_taxonomy_7_levels \
--output-path 99_otus_16S_taxonomy.qza

#Extract the V3/V4 region from the reference database
#--p-f-primer the primer on the left, 515F
#--p-r-primer the primer on the right, 806R
#--p-trunc-len  this value must be equal or less than the length of your trimed data

qiime feature-classifier extract-reads \
--i-sequences 99_otus_16S.qza \
--p-f-primer GTGCCAGCMGCCGCGGTAA \
--p-r-primer GGACTACHVGGGTWTCTAAT \
--p-trunc-len 233 \
--p-min-length 200 \
--p-max-length 250 \
--o-reads ref_seqs_16S_V3-V4-233.qza 

#this produces the QIIME2 artifact ref_seqs_16S_V3-V4.qza; the V3/V4 reference sequences.
#Train the classifier on this region

qiime feature-classifier fit-classifier-naive-bayes \
--i-reference-reads ref_seqs_16S_V3-V4-233.qza \
--i-reference-taxonomy 99_otus_16S_taxonomy.qza \
--o-classifier classifier_16S_V3-V4-233.qza 

#This step took a couple of hours. This produces the QIIME2 artifact classified_rep_seqs.qza.

cd taxonomy_silva132

#Classify rep seqs

qiime feature-classifier classify-sklearn \
--i-classifier taxonomy/classifier_16S_V3-V4-233.qza \
--i-reads merge_data/rep-seqs.qza \
--o-classification taxonomy/classified_rep_seqs-233.qza

#Tabulate the features, their taxonomy and the confidence of taxonomy assignment

qiime metadata tabulate \
--m-input-file classified_rep_seqs-233.qza \
--o-visualization classified_rep_seqs-233.qzv

#classified_rep_seqs barplot---table2

qiime taxa barplot \
  --i-table merge_data/table2.qza \
  --i-taxonomy taxonomy/classified_rep_seqs-233.qza \
  --m-metadata-file merge_data/merge_map.txt \
  --o-visualization taxonomy/taxabarplots_table2.qzv


############# Taxonomic analysis---gg-13-8-99-515-806-nb-classifier ############
qiime feature-classifier classify-sklearn \
  --i-classifier gg-13-8-99-515-806-nb-classifier.qza \
  --i-reads rep-seqs.qza \
  --o-classification taxonomy.qza


############# Taxonomic analysis---silva-132-99-515-806-nb-classifier ############

qiime feature-classifier classify-sklearn \
  --i-classifier silva-132-99-515-806-nb-classifier.qza \
  --i-reads rep-seqs.qza \
  --o-classification taxonomy.qza

#classified_rep_seqs barplot

qiime metadata tabulate \
  --m-input-file taxonomy.qza \
  --o-visualization taxonomy.qzv

qiime taxa barplot \
  --i-table table.qza \
  --i-taxonomy taxonomy.qza \
  --m-metadata-file merge_map.txt \
  --o-visualization taxa-bar-plots.qzv
```

# Collapse table at different level #####################################################
```
mkdir collapse

#Collapse table at phylum level

qiime taxa collapse \
--i-table table2.qza \
--i-taxonomy classified_rep_seqs-233.qza \
--p-level 2 \
--o-collapsed-table collapse/feature_table_level2.qza

#Collapse table at class level

qiime taxa collapse \
--i-table table2.qza \
--i-taxonomy classified_rep_seqs-233.qza \
--p-level 3 \
--o-collapsed-table collapse/feature_table_level3.qza

#Collapse table at order level

qiime taxa collapse \
--i-table table2.qza \
--i-taxonomy classified_rep_seqs-233.qza \
--p-level 4 \
--o-collapsed-table collapse/feature_table_level4.qza

#Collapse table at family level

qiime taxa collapse \
--i-table table2.qza \
--i-taxonomy classified_rep_seqs-233.qza \
--p-level 5 \
--o-collapsed-table collapse/feature_table_level5.qza

#Collapse table at genus level

qiime taxa collapse \
--i-table table2.qza \
--i-taxonomy classified_rep_seqs-233.qza \
--p-level 6 \
--o-collapsed-table collapse/feature_table_level6.qza
 
 
qiime tools extract \
--input-path collapse/feature_table_level6.qza \
--output-path collapse/feature_table_level6


biom convert \
-i collapse/feature_table_level6/4e6797a1-cb88-472e-a906-798e8ff496ab/data/feature-table.biom \
-o lcollapse/feature_table_level6.tsv \
--to-tsv
```

# diversity analysis ##################################################
```
#Generate a tree for phylogenetic diversity analyses

qiime phylogeny align-to-tree-mafft-fasttree \
  --p-n-threads 16 \
  --i-sequences rep-seqs.qza \
  --o-alignment aligned-rep-seqs.qza \
  --o-masked-alignment masked-aligned-rep-seqs.qza \
  --o-tree unrooted-tree.qza \
  --o-rooted-tree rooted-tree.qza
  
#Alpha and beta diversity analysis---table2

qiime diversity core-metrics-phylogenetic \
  --i-phylogeny rooted-tree.qza \
  --i-table table2.qza \
  --p-sampling-depth 10738 \
  --m-metadata-file merge_map.txt \
  --output-dir core-metrics-results-table2

#Alpha diversity analysis---faith

qiime diversity alpha-group-significance \
  --i-alpha-diversity core-metrics-results-table2/faith_pd_vector.qza \
  --m-metadata-file merge_map.txt \
  --o-visualization core-metrics-results-table2/faith-pd-group-significance.qzv

#Alpha diversity analysis---eveness

qiime diversity alpha-group-significance \
  --i-alpha-diversity core-metrics-results-table2/evenness_vector.qza \
  --m-metadata-file merge_map.txt \
  --o-visualization core-metrics-results-table2/evenness-group-significance.qzv


#Alpha diversity analysis---shannon

qiime diversity alpha-group-significance \
  --i-alpha-diversity core-metrics-results-table2/shannon_vector.qza \
  --m-metadata-file merge_map.txt \
  --o-visualization core-metrics-results-table2/shannon_vector.qzv
  
#Alpha diversity analysis---observed_otus

qiime diversity alpha-group-significance \
  --i-alpha-diversity core-metrics-results-table2/observed_otus_vector.qza \
  --m-metadata-file merge_map.txt \
  --o-visualization core-metrics-results-table2/observed_otus_vector.qzv




#Beta diversity analysis---unweighted_unifrac_distance_matrix

qiime diversity beta-group-significance \
  --i-distance-matrix core-metrics-results-table2/unweighted_unifrac_distance_matrix.qza \
  --m-metadata-file merge_map.txt \
  --m-metadata-column Treatment \
  --o-visualization core-metrics-results-table2/unweighted-unifrac-Treatment-significance.qzv \
  --p-pairwise

#Beta diversity analysis---jaccard_distance_matrix

qiime diversity beta-group-significance \
  --i-distance-matrix jaccard_distance_matrix.qza \
  --m-metadata-file merge_map.txt \
  --m-metadata-column Treatment \
  --o-visualization jaccard-Treatment-significance.qzv \
  --p-pairwise
  
#Beta diversity analysis---bray_curtis_distance_matrix

qiime diversity beta-group-significance \
  --i-distance-matrix bray_curtis_distance_matrix.qza \
  --m-metadata-file merge_map.txt \
  --m-metadata-column Treatment \
  --o-visualization bray_curtis-Treatment_significance.qzv \
  --p-pairwise

#Beta diversity analysis---weighted_unifrac_distance_matrix

qiime diversity beta-group-significance \
  --i-distance-matrix core-metrics-results-table2/weighted_unifrac_distance_matrix.qza \
  --m-metadata-file merge_map.txt \
  --m-metadata-column Treatment \
  --o-visualization core-metrics-results-table2/weighted-unifrac-Treatment-significance.qzv \
  --p-pairwise



#emperor tool: explore principal coordinates (PCoA) plots

qiime emperor plot \
  --i-pcoa core-metrics-results-table2/unweighted_unifrac_pcoa_results.qza \
  --m-metadata-file merge_map.txt \
  --o-visualization core-metrics-results-table2/unweighted-unifrac_pcoa_results.qzv


qiime emperor plot \
  --i-pcoa core-metrics-results-table2/bray_curtis_pcoa_results.qza \
  --m-metadata-file merge_map.txt \
  --o-visualization core-metrics-results-table2/bray-curtis_pcoa_results.qzv




#Alpha rarefaction plotting---table2

qiime diversity alpha-rarefaction \
  --i-table table2.qza \
  --i-phylogeny rooted-tree.qza \
  --p-max-depth 51000\
  --m-metadata-file merge_map.txt \
  --o-visualization alpha-rarefaction-table2-51000.qzv
```


# composition abundance analysis ########################################################
```
# ANCOM ##########
mkdir ANCOM

#ANCOM for original otu_table---table2

qiime composition add-pseudocount \
  --i-table table2.qza \
  --o-composition-table ANCOM/comp-table2.qza

#Run ANCOM

qiime composition ancom \
  --i-table ANCOM/comp-table2.qza \
  --m-metadata-file merge_map.txt \
  --m-metadata-column Treatment \
  --o-visualization ANCOM/ancom-Treatment-table2.qzv


#Differential abundance testing with ANCOM --- Treatment="ND"~Sex

qiime feature-table filter-samples \
  --i-table table2.qza \
  --m-metadata-file merge_map.txt \
  --p-where "[Treatment]='ND'" \
  --o-filtered-table ANCOM/ND-table.qza

qiime composition add-pseudocount \
  --i-table ANCOM/ND-table.qza \
  --o-composition-table ANCOM/comp-ND-table.qza

nohup qiime composition ancom \
  --i-table ANCOM/comp-ND-table.qza \
  --m-metadata-file merge_map.txt  \
  --m-metadata-column Description \
  --o-visualization ANCOM/ancom-ND-Description.qzv


#Differential abundance testing with ANCOM --- Treatment="D"~Sex

qiime feature-table filter-samples \
  --i-table table2.qza \
  --m-metadata-file merge_map.txt \
  --p-where "[Treatment]='D'" \
  --o-filtered-table ANCOM/D-table.qza

qiime composition add-pseudocount \
  --i-table ANCOM/D-table.qza \
  --o-composition-table ANCOM/comp-D-table.qza

qiime composition ancom \
  --i-table ANCOM/comp-D-table.qza \
  --m-metadata-file merge_map.txt  \
  --m-metadata-column Description \
  --o-visualization ANCOM/ancom-D-Description.qzv

#ANCOM for collapse table 

qiime composition add-pseudocount \
--i-table collapse/feature_table_level6.qza \
--o-composition-table ANNCOM/ANCOM_ready_table_level6.qza

qiime composition ancom \
  --i-table ANCOM/ANCOM_ready_table_level6 \
  --m-metadata-file merge_map.txt  \
  --m-metadata-column Treatment \
  --o-visualization ANCOM/ancom-Treatment-level6.qzv



# gneiss ##########
mkdir gneiss

#Also check remaining read counts

qiime feature-table summarize \
--i-table table2.qza \
--o-visualization gneiss/feature_table.qzv

#Correlation-clustering

qiime gneiss correlation-clustering \
  --i-table table2.qza \
  --o-clustering gneiss/hierarchy.qza

#computes the log ratios between groups at each node in the tree.

qiime gneiss ilr-hierarchical \
  --i-table table2.qza \
  --i-tree gneiss/hierarchy.qza \
  --o-balances gneiss/balances.qza

#run linear regression on the balances

qiime gneiss ols-regression \
  --p-formula "Treatment" \
  --i-table gneiss/balances.qza \
  --i-tree gneiss/hierarchy.qza \
  --m-metadata-file merge_map.txt \
  --o-visualization gneiss/regression_summary_Treatment.qzv

#run linear regression on the balances---different factors
qiime gneiss ols-regression \
  --p-formula "Diabetes+Age+HLA_Risk_Group" \
  --i-table gneiss2/balances.qza \
  --i-tree gneiss2/hierarchy.qza \
  --m-metadata-file new_metadata.txt \
  --o-visualization gneiss2/regression_summary_threefactors.qzv

qiime gneiss ols-regression \
  --p-formula "Description" \
  --i-table gneiss/balances.qza \
  --i-tree gneiss/hierarchy.qza \
  --m-metadata-file merge_map.txt \
  --o-visualization gneiss/regression_summary_Description.qzv

#visualize these balances on a heatmap

qiime gneiss dendrogram-heatmap \
  --i-table table2.qza \
  --i-tree gneiss/hierarchy.qza \
  --m-metadata-file merge_map.txt \
  --m-metadata-column Treatment \
  --p-color-map seismic \
  --o-visualization gneiss/heatmap.qzv

#plot a boxplot and identify taxa that could be explaining the differences between the D and ND groups.

qiime gneiss balance-taxonomy \
  --i-table table2.qza \
  --i-tree gneiss/hierarchy.qza \
  --i-taxonomy classified_rep_seqs-233.qza \
  --p-taxa-level 2 \
  --p-balance-name 'y0' \
  --m-metadata-file merge_map.txt \
  --m-metadata-column Treatment \
  --o-visualization gneiss/y0_taxa_summary_lv2.qzv

nohup qiime gneiss balance-taxonomy \
  --i-table table2.qza \
  --i-tree gneiss/hierarchy.qza \
  --i-taxonomy classified_rep_seqs-233.qza \
  --p-taxa-level 6 \
  --p-balance-name 'y0' \
  --m-metadata-file merge_map.txt \
  --m-metadata-column Treatment \
   --o-visualization gneiss/y0_taxa_summary_lv6.qzv
```

# data extraction ########################################################
```
qiime tools extract \
--input-path rooted_tree.qza \
--output-path rooted_tree

qiime tools extract \
--input-path classified_rep_seqs-233.qza \
--output-path classified_rep_seqs-233

qiime tools extract \
--input-path table.qza \
--output-path table
```

# data conversion  ###############################
```
biom convert \
-i table2/9c40c08e-45c1-405c-bc41-27c3e05025c5/data/feature-table.biom \
-o table2/out_table.tsv \
--to-tsv

biom convert -h

# small tips  ########################################################
#Count the number of sequences in a FASTQ file

grep -n filename.fastq

#Count the total number of rows in a file that contain primers

grep -c ‘primer/barcode/adapter/’ filename.fastq

#count the total number of rows in a file

wc -l filename.fastq

#compress file

gzip filename
gzip -c filename

#unzip file

gzip -d filename
gunzip filename

#check .fastq file

less -S -N filename.fastq
zless -S -N filename.fastq.gz

cat -n filename.fastq
zcat -n filename.fastq.gz
zcat filename.fastq.gz | head -n 10

head -n filename.fastq
tail -n filename.fastq




biom add-metadata \
-i exported/feature-table.biom \
-o table-with-taxonomy.biom \
--observation-metadata-fp exported/taxonomy.tsv \
--observation-header taxonomy \
--sc-separated taxonomy

biom convert \
-i table-with-taxonomy.biom \
-o table-with-taxonomy.tsv \
--to-tsv \
--header-key taxonomy


biom add-metadata \
-i feature-table.biom \
-o table- with -taxonomy.biom \
–-observation-metadata-fp biom-taxonomy.tsv \
--sc-separated taxonomy
```

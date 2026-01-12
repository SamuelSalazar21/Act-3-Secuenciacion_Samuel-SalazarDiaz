COMANDOS

A conyinuación presento la serie comandos realizados para llevar a cabo toda la actividad.

1. Descargar metadatos y secuencias

```
# metadatos
wget -O "sample-metadata.tsv" https://data.qiime2.org/2023.9/tutorials/atacama-soils/sample_metadata.tsv

# secuencias
curl -sL "https://data.qiime2.org/2023.9/tutorials/atacama-soils/10p/forward.fastq.gz" > "emp-paired-end-sequences/forward.fastq.gz"
curl -sL "https://data.qiime2.org/2023.9/tutorials/atacama-soils/10p/reverse.fastq.gz" > "emp-paired-end-sequences/reverse.fastq.gz"
curl -sL "https://data.qiime2.org/2023.9/tutorials/atacama-soils/10p/barcodes.fastq.gz" > "emp-paired-end-sequences/barcodes.fastq.gz"
```

2. Crear el artefacto de qiime2 con las secuencias

```
qiime tools import --type EMPPairedEndSequences --input-path emp-paired-end-sequences --output-path emp-paired-end-sequences.qza
```

3. Demultiplexar las reads

```
qiime demux emp-paired --m-barcodes-file sample-metadata.tsv --m-barcodes-column barcode-sequence --p-rev-comp-mapping-barcodes --i-seqs emp-paired-end-sequences.qza --o-per-sample-sequences demux-full.qza --o-error-correction-details demux-details.qza
```

4. Crear una submuestra con el 30%

```
# creacion
qiime demux subsample-paired --i-sequences demux-full.qza --p-fraction 0.3 --o-subsampled-sequences demux-subsample.qza

# visualizacion
qiime demux summarize --i-data demux-subsample.qza --o-visualization demux-subsample.qzv
```

5. Filtrar muestras que tienen menos de 100 reads

```
# crear carpeta de salida
mkdir demux-subsample

# informacion de la submuestra
qiime tools export --input-path demux-subsample.qzv --output-path demux-subsample

# filtrar muestras
qiime demux filter-samples --i-demux demux-subsample.qza --m-metadata-file demux-subsample/per-sample-fastq-counts.tsv --p-where 'CAST([forward sequence count] AS INT) > 100' --o-filtered-demux demux.qza
```

6. Revisar la calidad y realizar denoising

```
# DADA2
qiime dada2 denoise-paired --i-demultiplexed-seqs demux.qza --p-trim-left-f 13 --p-trim-left-r 13 --p-trunc-len-f 150 --p-trunc-len-r 150 --o-table table.qza --o-representative-sequences rep-seqs.qza --o-denoising-stats denoising-stats.qza

# generar resumenes
qiime feature-table summarize --i-table table.qza --o-visualization table.qzv --m-sample-metadata-file sample-metadata.tsv

qiime feature-table tabulate-seqs --i-data rep-seqs.qza --o-visualization rep-seqs.qzv

qiime metadata tabulate --m-input-file denoising-stats.qza --o-visualization denoising-stats.qzv
```

7. Generar un árbol para el análisis de diversidad filogenética (MAFF y FastTree)

```
# construccion del arbol
qiime phylogeny align-to-tree-mafft-fasttree --i-sequences rep-seqs.qza --o-alignment aligned-rep-seqs.qza --o-masked-alignment masked-aligned-rep-seqs.qza --o-tree unrooted-tree.qza --o-rooted-tree rooted-tree.qza

# exportar en formato newick
qiime tools export --input-path rooted-tree.qza --output-path visualizacion/
```

8. Cálculo de diversidad alfa y beta

```
# curva de rarefaccion
qiime diversity alpha-rarefaction --i-table table.qza --i-phylogeny rooted-tree.qza --p-max-depth 2113 --m-metadata-file sample-metadata.tsv --o-visualization alpha-rarefaction.qzv

# metricas estandar
qiime diversity core-metrics-phylogenetic --i-phylogeny rooted-tree.qza --i-table table.qza --p-sampling-depth 927 --m-metadata-file sample-metadata.tsv --output-dir core-metrics-results

# diversidad alfa
qiime diversity alpha-group-significance --i-alpha-diversity core-metrics-results/observed_features_vector.qza --m-metadata-file sample-metadata.tsv --o-visualization core-metrics-results/observed_features-group-significance.qzv

qiime diversity alpha-group-significance --i-alpha-diversity core-metrics-results/faith_pd_vector.qza --m-metadata-file sample-metadata.tsv --o-visualization core-metrics-results/faith_pd-group-significance.qzv

qiime diversity alpha-group-significance --i-alpha-diversity core-metrics-results/shannon_vector.qza --m-metadata-file sample-metadata.tsv --o-visualization core-metrics-results/shannon-group-significance.qzv

qiime diversity alpha-group-significance --i-alpha-diversity core-metrics-results/evenness_vector.qza --m-metadata-file sample-metadata.tsv --o-visualization core-metrics-results/evenness-group-significance.qzv

# diversidad beta PERMANOVA

### transect-name
qiime diversity beta-group-significance --i-distance-matrix core-metrics- https://open.spotify.com/track/0l0vcZMU7AOeQmUIREoI2Uresults/unweighted_unifrac_distance_matrix.qza --m-metadata-file sample-metadata.tsv --m-metadata-column transect-name --o-visualization core-metrics-results/unweighted-unifrac-transect-name-significance.qzv --p-pairwise

qiime diversity beta-group-significance --i-distance-matrix core-metrics-results/weighted_unifrac_distance_matrix.qza --m-metadata-file sample-metadata.tsv --m-metadata-column transect-name --o-visualization core-metrics-results/weighted-unifrac-transect-name-significance.qzv --p-pairwise

### vegetation
qiime diversity beta-group-significance --i-distance-matrix core-metrics-results/unweighted_unifrac_distance_matrix.qza --m-metadata-file sample-metadata.tsv --m-metadata-column vegetation --o-visualization core-metrics-results/unweighted-unifrac-vegetation-significance.qzv --p-pairwise

qiime diversity beta-group-significance --i-distance-matrix core-metrics-results/weighted_unifrac_distance_matrix.qza --m-metadata-file sample-metadata.tsv --m-metadata-column vegetation --o-visualization core-metrics-results/weighted-unifrac-vegetation-significance.qzv --p-pairwise

### extract-group-no
qiime diversity beta-group-significance --i-distance-matrix core-metrics-results/unweighted_unifrac_distance_matrix.qza --m-metadata-file sample-metadata.tsv --m-metadata-column extract-group-no --o-visualization core-metrics-results/unweighted-unifrac-extract-group-no-significance.qzv --p-pairwise

qiime diversity beta-group-significance --i-distance-matrix core-metrics-results/weighted_unifrac_distance_matrix.qza --m-metadata-file sample-metadata.tsv --m-metadata-column extract-group-no --o-visualization core-metrics-results/weighted-unifrac-extract-group-no-significance.qzv --p-pairwise

# diversidad beta PCoA 
qiime emperor plot --i-pcoa core-metrics-results/weighted_unifrac_pcoa_results.qza --m-metadata-file sample-metadata.tsv --p-custom-axes elevation ph --o-visualization core-metrics-results/pcoa/weighted-unifrac-emperor-elevation-ph.qzv
```

9. Análisis taxonómico

```
# descargar base de datos greengenes
curl -sL "https://data.qiime2.org/2023.9/common/gg-13-8-99-515-806-nb-classifier.qza" > "gg-13-8-99-515-806-nb-classifier.qza"

# clasificacion taxonomica
qiime feature-classifier classify-sklearn --i-classifier gg-13-8-99-515-806-nb-classifier.qza --i-reads rep-seqs.qza --o-classification taxonomy.qza

# crear tabla con taxonomia
qiime metadata tabulate --m-input-file taxonomy.qza --o-visualization taxonomy.qzv

# grafico de abundacia taxonomica
qiime taxa barplot --i-table table.qza --i-taxonomy taxonomy.qza --m-metadata-file sample-metadata.tsv --o-visualization taxa-bar-plots.qzv
```

10. Análisis diferencial de abundancia microbiana

```
# preparar tabla
qiime composition add-pseudocount --i-table table.qza --o-composition-table comp-table.qza

# AMCO
qiime composition ancom --i-table comp-table.qza --m-metadata-file sample-metadata.tsv --m-metadata-column extract-group-no --o-visualization ancom-extract-group-no.qzv

qiime composition ancom --i-table comp-table.qza --m-metadata-file sample-metadata.tsv --m-metadata-column vegetation --o-visualization ancom-vegetation.qzv

qiime composition ancom --i-table comp-table.qza --m-metadata-file sample-metadata.tsv --m-metadata-column site-name --o-visualization ancom-site-name.qzv

# taxonomia nivel 6
qiime taxa collapse --i-table table.qza --i-taxonomy taxonomy.qza --p-level 6 --o-collapsed-table table-l6.qza

qiime composition add-pseudocount --i-table table-l6.qza --o-composition-table comp-table-l6.qza

qiime composition ancom --i-table comp-table-l6.qza --m-metadata-file sample-metadata.tsv --m-metadata-column extract-group-no --o-visualization l6-ancom-extract-group-no.qzv
```














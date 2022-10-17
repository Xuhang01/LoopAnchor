# LoopAnchor

Variable interactions between enhancer and target gene have been used in explaining cell type-specific transcriptional regulation scheme, but the mechanisms underlying the precise wiring between genes and the enhancers from varied distances remain unclear. As the main insulator-related transcription factor (TF) discovered in vertebrates, CTCF assists cohesin to form chromatin loops, which is believed to be the fundamental biophysical basis for distal gene regulation.
Here we present LoopAnchor to provide a two-step in silico strategy to predict cohesin-meidated chromatin loops. 
In the first step, we developed a Deep Learning model `DeepAnchor` to predict the genomic/epigenomic partterns surrounding insulation related CTCF binding sites (CBSs). Generally, `DeepAnchor` uses cohesin ChIA-PET data, CTCF ChIP-seq data, and CBSs as inputs, to train a classifier to distinguish insulation-related CBSs from others. Then we use the model to calculate anchor score ranging from [0, 1] for all CBSs. CBSs with larger score is more likely to be positive CTCF insulator. 
In the second step, we introduced anchor score into an existing method, loop extrusion model, to make predictions for cohesin-meidated loops. 
Besides, we have collected CTCF ChIP-seq data from many cell types and use LoopAnchor to predict loops. All predicted loops for 168 selected biosamples and all 764 biosamples can be visualized and compared as separated tracks at UCSC Track Data Hubs (https://genome.ucsc.edu/cgi-bin/hgHubConnect) by entering customized hub URLs https://raw.githubusercontent.com/mulinlab/LoopAnchor/master/hubs_landscape.txt or https://raw.githubusercontent.com/mulinlab/LoopAnchor/master/hubs_all.txt, respectively.

Because LoopAnchor is a two-step strategy, it allows users to start from any steps they want. For example, if users want to test DeepAnchor model, they can download the feature data, which is pretty large, to train the model. If users only want to get the cohensin-meidated loops, they can either start from the second step, where they only need to prepare their own CTCF ChIP-seq data to make predictions, or download the loops from landscape if the cell type is already included in landscape. 



## DeepAnchor
### Prepare data
To implement `DeepAnchor`, user should prepare four types of data:
1. Obtain CBSs by motif scanning.
2. Download base-wise genomic/epigenomic features from [CADD v1.3](https://cadd.gs.washington.edu/download). Here we also provide preprocessed CADD features for CTCF binding sites (please see below).
3. CTCF ChIP-seq data for specific cell type (for example GM12878).
4. Cohesin ChIA-PET data for the same cell type. 


### Generate training data
#### 1. Extract feature values
We totally use 44 genomic/epigenomic features from CADD database and 4 DNA sequence features (`A/T/G/C`). For each CBS and each feature, feature values between ±500bp are extracted. Therefore, the size of feature matrix for each CBS is $48 x 1000$. Feature matrix is min-max normalized to make the value between [0, 1] that will facilitate downstream analyses. However, this step is very time consuming. So we provide the preprocessed feature matrix data ([`cadd feature matrix`](http://www.mulinlab.org/LoopAnchor/cadd_feature.npz) and [`dna feature matrix`](http://www.mulinlab.org/LoopAnchor/dna_feature.npz)).

#### 2. Generate positive/negative (P/N) CBSs
We need create a `work_dir` with the following structures and put five data files into it.
```
work_dir
    └── raw                   
        ├── CTCF_peak.bed.gz              
        ├── loop.bedpe              
        ├── CTCF_motif.tsv
        ├── dna_feature.npz      
        └── cadd_feature.npz
```
Here are the detailed explainations of these files and the meanings of columns:

* `CTCF_peak.bed.gz`: the ChIP-seq narrowPeak file for CTCF with 10 columns (chrom, chromStart, chromEnd, name, score, strand, signalValue, pValue, qValue, summit). The definition of narrowPeak can be referred from [`ENCODE narrowPeak: Narrow (or Point-Source) Peaks format`](https://genome.ucsc.edu/FAQ/FAQformat.html#format12). Here is a brief explaination:
  * chrom, chromStart, chromEnd, strand:
  * name, score, signalValue:
  * pValue:
  * qValue:
  * summit:
* `loop.bedpe`: the ChIA-PET loop file for Cohesin (RAD21) from the same cell type with 10 columns (chr1, start1, end1, chr2, start2, end2, name, score, strand1, strand2). The detailed explaination of bedpe format can be referred from [BEDPE format](https://bedtools.readthedocs.io/en/latest/content/general-usage.html). The meanings of these columns are:
  * chr1, start1, end1, strand1:
  * chr2, start2, end2, strand2: 
  * name, score:
* `CTCF_motif.tsv`: the position of all CTCF binding sites. This file is non-cell-type specific and can be found in the `data` folder ([data/CTCF_motif.tsv](data/CTCF_motif.tsv)).
* `dna_feature.npz`: the one-hot representation of DNA sequence for CTCF binding sites. This file is non-cell-type specific. We have prepared this file which can be downloaded directly:  [`dna feature matrix`](http://www.mulinlab.org/LoopAnchor/dna_feature.npz).
* `cadd_feature.npz`: the CADD features of DNA sequence for CTCF binding sites. This file is non-cell-type specific. We have prepared this file which can be downloaded directly: [`cadd feature matrix`](http://www.mulinlab.org/LoopAnchor/cadd_feature.npz). 

To generate P/N datasets, user can simply run the following command:
```properties
# cd LoopAnchor # check the working directory
python DeepAnchor_input.py work_dir
```


It will generate an output folder named `DeepAnchor` in `work_dir`, that is `work_dir/DeepAnchor`:
```
work_dir
    └── DeepAnchor  
        ├── total_anchors.bed        # all anchors extracted from loop.bedpe
        ├── CTCF_peaks.bed           # intersect CTCF_peak.bed.gz and CTCF_motif.tsv
        ├── marked motif.tsv         # mark CBSs with CTCF ChIP-seq peaks          
        ├── train.npz                # train set (chr1-16)
        ├── valid.npz                # valid set (chr17-18)
        ├── test.npz                 # test set (chr19-X)                 
        └── total.npz                # feature data for all CBSs
```


### Run DeepAnchor
`DeepAnchor` trains a classifier to distinguish insulator-related CBSs from others. After training, the model will be used to predict DeepAnchor score to show the possibility that each CBS belong to insulator-related CBSs. 

To train the classifier model, run command:
```properties
python DeepAnchor.py work_dir train
```
This will generate a DeepAnchor model which will be saved as `DeepAnchor.model` in `work_dir` (`work_dir/DeepAnchor.model`).

To predict the DeepAnchor score for all CBSs, run command:

```properties
python DeepAnchor.py work_dir predict
```

This will generate a file `scored_motif.tsv` that contains all CBSs and their DeepAnchor score. This file **must be copied to** `data` folder for downstream analyses: `cp work_dir/scored_motif.tsv data/`.

The file `scored_motif.tsv` contains 6 columns (chrom, start, end, strand, motif_score, anchor_score):

* chrom, start, end, strand:
* motif_score: the score from motif scanning
* anchor_score:  the score predicted by DeepAnchor model


## LoopAnchor

To use `LoopAnchor` for loop prediction, you should prepare input data arranged as follow:

```
work_dir
    └── raw                   
        └── CTCF_peak.bed.gz
```

Then, run command:
```properties
python run_LoopAnchor_denovo.py work_dir
```
In `work_dir/LoopAnchor` folder, you can find the result `LoopAnchor_pred.bedpe` which contains all the loops predicted by `LoopAnchor`. Result is arranged in [bedpe format](https://bedtools.readthedocs.io/en/latest/content/general-usage.html) and the last column named `score` is the predicted loop intensity.

Here is a complete example. The data can be found in `data` folder, but you still need to download some files as shown before.
```properties
python DeepAnchor_input.py ./data/GM12878
python DeepAnchor.py ./data/GM12878 train
python DeepAnchor.py ./data/GM12878 predict
python run_LoopAnchor_denovo.py ./data/K562
```


## Landscape availability

We have collected 764 available CTCF ChIP-seq data from ENCODE, CistromDB, and ChIP-Atlas, and used `LoopAnchor` to predict CTCF-anchored loops. The results are available at UCSC Track Data Hubs (https://genome.ucsc.edu/cgi-bin/hgHubConnect) by entering customized hub URLs https://raw.githubusercontent.com/mulinlab/LoopAnchor/master/hubs_landscape.txt or https://raw.githubusercontent.com/mulinlab/LoopAnchor/master/hubs_all.txt, respectively.

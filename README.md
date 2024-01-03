# 🍳 Metagenomic-DeepFRI [![Stars](https://img.shields.io/github/stars/bioinf-MCB/Metagenomic-DeepFRI.svg?style=social&maxAge=3600&label=Star)](https://github.com/bioinf-MCB/Metagenomic-DeepFRI/stargazers)
*A pipeline for annotation of genes with [DeepFRI](https://github.com/flatironinstitute/DeepFRI), a deep learning model for functional protein annotation with [Gene Ontology (GO) terms](https://geneontology.org/docs/go-annotations/). It incorporates [FoldComp](https://github.com/steineggerlab/foldcomp) databases of predicted protein structures for fast annotation of metagenomic gene catalogues.*

## 🔍 Overview
Proteins perform most of the work of living cells. Amino acid sequence and structural features of proteins determine a wide range of functions: from binding specificity and conferring mechanical stability, to catalysis of biochemical reactions, transport, and signal transduction.
DeepFRI is a neural network designed to predict protein function within the framework of the Gene Ontology (GO). The exponential growth in the number of available protein sequences, driven by advancements in low-cost sequencing technologies and computational methods (e.g., gene prediction), has resulted in a pressing need for efficient software to facilitate the annotation of protein databases.
Metagenomic-DeepFRI addresses such need, building upon efficient libraries. It incorporates novel databases of predicted structures (AlphaFold, ESM-Fold, MIP, etc.) and improves runtimes of DeepFRI by [2-12 times](https://github.com/bioinf-mcb/Metagenomic-DeepFRI/blob/main/weight_convert/onnx_vs_tf2.png)!

### 📋 Pipeline stages

1. Search proteins similar to query in a `FoldComp` database with `MMSeqs2`.
2. Find the best alignment among `MMSeqs2` hits using `PyOpal`.
3. Align target protein contact map to query protein with unknown structure.
4. Run `DeepFRI` with structure if it was found in database, otherwise run `DeepFRI` with sequence only.

### 🛠️ Built With

* [MMseqs2](https://github.com/soedinglab/MMseqs2)
* [pyOpal](https://github.com/althonos/pyOpal)
* [DeepFRI](https://github.com/flatironinstitute/DeepFRI)
* [FoldComp](https://github.com/steineggerlab/foldcomp)
* [ONNX](https://github.com/onnx/onnx)

## 🔧 Installation

1. Clone repo locally
```{code-block} bash
git clone https://github.com/bioinf-mcb/Metagenomic-DeepFRI
cd Metagenomic-DeepFRI
```
2. Setup conda environment
```{code-block} bash
conda env create --name deepfri --file environment.yml
conda activate deepfri
```
3. Install `mDeepFRI`
```{code-block} bash
pip install .
```

## 💡 Usage
### 1. Prepare structural database
Download the database from the [website](https://foldcomp.steineggerlab.workers.dev/). The app was tested with `afdb_swissprot_v4`. You can use different databases, but be mindful that computation time might increase exponentially with the size of the database and the format of protein names might differ and the app will crash.
### 2. Download models
Two versions of models available:
- `v1.0` - is the original version from DeepFRI publication.
- `v1.1` - is a version finetuned on AlphaFold models and Gene Ontology Uniprot annotations.
To download models run command:
```
mDeepFRI get-models --output path/to/weights/folder -v {1.0 or 1.1}
```

### 3. Predict protein function & capture log
```
mDeepFRI predict-function -i /path/to/protein/sequences -d /path/to/foldcomp/database/ -w /path/to/deepfri/weights/folder -o /output_path 2> log.txt
```

The `logging` module writes output into `stderr`, so use `2>` to redirect it to the file.
Other available parameters can be found upon command `mDeepFRI --help`.

## ✅ Results
The output folder will contain:
1. `{database_name}.search_results.tsv`
2. `metadata_skipped_ids_due_to_length.json` - too long or too short queries (DeepFRI is designed to predict the function of proteins in the range of 60-1000 aa).
3. `query.mmseqsDB` + index from MMSeqs2 search.
4. `results.tsv` - a final output from the DeepFRI model.

### Example output (`results.tsv`)
|  Protein     | GO_term/EC_numer | Score | Annotation                                   | Neural_net | DeepFRI_mode | DB_hit        | DB_name        |Identity |
|--------------|------------------|-------|----------------------------------------------|------------|--------------|---------------|----------------|------------|
| MIP_00215364 | GO:0016798       | 0.218 | hydrolase activity, acting on glycosyl bonds | gcn        | mf           | MIP_00215364  | mip_rosetta_hq |0.933      |
| 1GVH_1 | GO:0009055       | 0.217 | electron transfer activity	                 | gnn        | mf           | 	AF-P24232-F1-model_v4 | afdb_swissprot_v4 | 1.0  |
| unaligned | 3.2.1.-          | 0.215 | 3.2.1.-                        | cnn        | ec           | nan | nan | nan

This is an example of protein annotation with the AlphaFold database.
- Protein - the name of the protein from the FASTA file.
- GO_term/EC_numer - predicted GO term or EC number (dependent on mode)
- Score - DeepFRI score, translates to model confidence in prediction. Details in [publication](https://www.nature.com/articles/s41467-021-23303-9).
- Annotation - annotation from ontology
- Neural_net - type of neural network used for prediction (gcn = Graph Convolutional Network; cnn = Convolutional Neural Network). GCN (Graph Convolutional Network) is employed when structural information is available in the database, allowing for generally more confident predictions.
- DeepFRI_mode:
   ```
   mf = molecular_function
   bp = biological_process
   cc = cellular_component
   ec = enzyme_commission
   ```

## ⚙️Features
### 1. Prediction modes
The GO ontology contains three subontologies, defined by their root nodes:
- Molecular Function (MF)
- Biological Process (BP)
- Cellular Component (CC)
- Additionally, Metagenomic-DeepFRI v1.0 is able to predict Enzyme Comission number (EC).
By default, the tool makes predictions in all 4 categories. To select only a few pass the parameter `-p` or `--processing-modes` few times, i.e.:
```
mDeepFRI predict-function -i /path/to/protein/sequences -d /path/to/foldcomp/database/ -w /path/to/deepfri/weights/folder -o /output_path -p mf -p bp
```

### 2. Hierarchical database search
Different databases have a different level of evidence. For example, PDB structures are real experimental structures, and AlphaFold predictions are more accurate than ESMFold predictions. We provide an opporunity to search multiple databases in a hierarchical manner. For example, if you want to search AlphaFold database first, and then ESMFold, you can pass the parameter `-d` or `--databases` few times, i.e.:
```
mDeepFRI predict-function -i /path/to/protein/sequences -d /path/to/alphafold/database/ -d /path/to/another/esmcomp/database/ -w /path/to/deepfri/weights/folder -o /output_path
```

### 3. Temporary files
The first run of `mDeepFRI` with the database will create temporary files, needed for the pipeline. If you don't want to keep them for the next run add
flag `--remove-intermediate`.

### 4. CPU / GPU utilization
If argument `threads` is provided, the app will parallelize certain steps (alignment, contact map alignment, functional annotation).
GPU is often used to speed up neural networks. Metagenomic-DeepFRI takes care of this and, if CUDA is installed on your machine, `mDeepFRI` will automatically use it for prediction. If not, the model will use CPUs.
**Technical tip:** Single instance of DeepFRI on GPU requires 2GB VRAM. Every currently available GPU with CUDA support should be able to run the model.

## 🔖 Citations
Metagenomic-DeepFRI is a scientific software. If you use it in an academic work, please cite the papers behind it:
- Gligorijević et al. "Structure-based protein function prediction using graph convolutional networks" Nat. Comms. (2021). https://doi.org/10.1038/s41467-021-23303-9
- Steinegger & Söding "MMseqs2 enables sensitive protein sequence searching for the analysis of massive data sets" Nat. Biotechnol. (2017) https://doi.org/10.1038/nbt.3988
- Kim, Midrita & Steinegger "Foldcomp: a library and format for compressing and indexing large protein structure sets" Bioinformatics (2023) https://doi.org/10.1093/bioinformatics/btad153
- Maranga et al. "Comprehensive Functional Annotation of Metagenomes and Microbial Genomes Using a Deep Learning-Based Method" mSystems (2023) https://doi.org/10.1128/msystems.01178-22

## 💭 Feedback

### ⚠️ Issue Tracker

Found a bug ? Have an enhancement request ? Head over to the [GitHub issue
tracker](https://github.com/bioinf-mcb/Metagenomic-DeepFRI/issues) if you need to report
or ask something. If you are filing in on a bug, please include as much
information as you can about the issue, and try to recreate the same bug
in a simple, easily reproducible situation.

### 🏗️ Contributing

Contributions are more than welcome! See
[`CONTRIBUTING.md`](https://github.com/bioinf-mcb/Metagenomic-DeepFRI/blob/main/CONTRIBUTING.md)
for more details.

## 📋 Changelog

This project adheres to [Semantic Versioning](http://semver.org/spec/v2.0.0.html)
and provides a [changelog](https://github.com/bioinf-mcb/Metagenomic-DeepFRI/blob/main/CONTRIBUTING.md)
in the [Keep a Changelog](http://keepachangelog.com/en/1.0.0/) format.


## ⚖️ License

This library is provided under the [The 3-Clause BSD License](https://opensource.org/license/bsd-3-clause/).

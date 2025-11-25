
# EvOlf: Evolutionary-Guided GPCR-Ligand Interaction Prediction

[](https://opensource.org/licenses/MIT)

**EvOlf** is a state-of-the-art deep learning framework designed to predict ligand-GPCR interactions with high accuracy. By integrating data from both odorant (ORs) and non-odorant GPCRs across 24 species, EvOlf leverages evolutionary insights to deorphanize receptors and predict novel bioactive ligands.

This repository contains the [**Nextflow pipeline**](https://www.nextflow.io/) implementation of EvOlf. It automates the complex process of molecular featurization, protein embedding generation, and deep learning inference, allowing researchers to run large-scale interaction screens on local machines, HPC clusters, or the cloud.

 **Explore the Data and try EvOlf on the Webserver:** [EvOlf Web](https://www.google.com/search?q=https://evolf.iiitd.edu.in)

-----

## Table of Contents

  - [Pipeline Architecture](#pipeline-architecture)
  - [Installation & Prerequisites](#installation--prerequisites)
  - [Input Format](#input-format)
  - [Usage](#usage)
      - [Single-File Mode](#1-single-file-mode)
      - [Batch Mode](#2-batch-mode-high-throughput)
  - [Output Explanation](#output-explanation)
  - [Configuration & Tuning](#configuration--tuning)
  - [Troubleshooting](#troubleshooting)
  - [Credits & Citations](#credits--citations)

-----

## Pipeline Architecture

The EvOlf pipeline is a modular, multi-stage workflow built on Nextflow. It parallelizes the heavy lifting of feature generation across multiple containers.

### 1\. Input Standardization (`PREPARE_INPUT`)

  - Validates input CSV formats.
  - Standardizes SMILES strings and checks for invalid characters.
  - Maps user-provided IDs or generates auto-incrementing IDs (`L1`, `R1`, `LR1`) to track interactions throughout the pipeline.

### 2\. Multi-Modal Featurization

EvOlf uses a "consensus" approach, combining multiple representations of both ligands and receptors to capture chemical and biological nuances.

**Ligand Featurizers (Small Molecules):**

  * **ChemBERTa:** A transformer model pre-trained on 77M SMILES strings for learning molecular substructures.
  * **Signaturizer:** Infers bioactivity signatures (MoA) based on the Chemical Checker.
  * **Mol2Vec:** An unsupervised method (Word2Vec-based) to learn vector representations of molecular substructures.
  * **Mordred:** Calculates over 1,600 2D and 3D physicochemical descriptors.
  * **Graph2Vec:** Represents molecules as graphs to capture topological structures.

**Receptor Featurizers (Proteins):**

  * **ProtBERT:** A massive protein language model (BERT-based) trained on UniRef100.
  * **ProtT5:** A T5-based encoder for capturing long-range dependencies in protein sequences.
  * **ProtR:** Calculates physicochemical properties (hydrophobicity, charge, etc.) from amino acid sequences.
  * **MathFeaturizer:** A custom module encoding sequences using mathematical descriptors (chaos game representation, etc.).

### 3\. Deep Learning Inference (`EVOLF_PREDICTION`)

  - **Feature Compilation:** Aggregates the diverse feature sets into a unified tensor.
  - **Prediction:** Passes the tensors through the trained EvOlf deep neural network.
  - **Output:** Returns a probability score (`0.0` to `1.0`) indicating the likelihood of interaction.

-----

## Installation & Prerequisites

You do not need to install Python libraries or R packages manually. The only requirements are Nextflow and a container engine.

1.  **Install [Nextflow](https://www.nextflow.io/)** (Requires Java 11+):

    ```bash
    conda create -n nf-env bioconda::nextflow
    conda activate nf-env
    ```

2.  **Install a Container Engine:**

      * [**Docker Desktop / Engine:**](https://docker.com/) Best for local development and workstations.
      * **[Apptainer](https://apptainer.org/) (formerly [Singularity](https://sylabs.io/singularity/)):** **Required** for HPC clusters and shared servers.

3.  **Clone this Repository:**

    ```bash
    git clone https://github.com/the-ahuja-lab/evolf-pipeline.git
    cd evolf-pipeline
    ```

-----

## Input Format

The pipeline expects a **CSV file** with at least two columns: the Ligand SMILES and the Receptor Amino Acid Sequence.

### Recommended Format (With IDs)

Providing your own IDs is the safest way to track your results.

```csv
ID,Ligand_ID,SMILES,Receptor_ID,Receptor_Sequence
Pair_1,Lig_A,CCO,Rec_1,MGA...
Pair_2,Lig_B,CCN,Rec_1,MGA...
```

### Minimal Format (No IDs)

EvOlf can auto-generate IDs for you.

```csv
SMILES,Sequence
CCO,MGA...
CCN,MGA...
```

> **⚠️ Crucial Tip:** Convert your SMILES to **Canonical Format** before running the pipeline\!
>
> ```bash
> obabel -ismi input.smi -ocan -O canonical.smi
> ```
>
> Non-canonical SMILES can lead to inconsistent feature generation.

-----

## Usage

### 1\. Single-File Mode

Use this for running a single dataset.

```bash
nextflow run main.nf \
    --inputFile "data/my_experiment.csv" \
    --ligandSmiles "SMILES" \
    --receptorSequence "Sequence" \
    --outdir "./results/experiment_1" \
    -profile docker,gpu
```

### 2\. Batch Mode (High-Throughput)

Use this to process multiple datasets in parallel. Create a manifest CSV file:

**`batch_manifest.csv`**:

```csv
inputFile,ligandSmiles,receptorSequence,ligandID,receptorID,lrID
/data/project_A.csv,SMILES,Seq,L_ID,R_ID,Pair_ID
/data/project_B.csv,cano_smiles,aa_seq,lig_name,prot_name,int_id
```

**Run Command:**

```bash
nextflow run main.nf \
    --batchFile "batch_manifest.csv" \
    --outdir "./results/batch_run" \
    -profile docker,gpu
```

-----

## Output Explanation

The pipeline creates a separate folder for each input file in your output directory.

```
results/
├── my_experiment/
│   ├── my_experiment_Prediction_Output.csv    <-- FINAL RESULTS
│   ├── my_experiment_Input_ID_Information.csv <-- ID Mapping Key
│   └── pipeline_info/                           
│       ├── execution_report.html              <-- Resource usage (RAM/CPU)
│       └── execution_timeline.html            <-- Gantt chart of jobs
```

### The Prediction File (`Prediction_Output.csv`)

| Column | Description |
| :--- | :--- |
| `ID` | The unique Pair ID (User-provided or auto-generated `LR1`). |
| `Predicted Label` | **1** (Interaction) or **0** (No Interaction). Threshold is 0.5. |
| `P1` | The raw **probability score** (0.0 to 1.0). Higher = stronger interaction confidence. |

-----

## Configuration & Tuning

### Hardware Profiles (`-profile`)

  * **`docker`**: Uses Docker. Runs as the current user (`-u $(id -u)`) to prevent root-owned file issues.
  * **`apptainer` / `singularity`**: Uses `.sif` images. Automatically mounts your project directory.
  * **`gpu`**: **Highly Recommended.** Enables CUDA for ChemBERTa, ProtBERT, ProtT5, and the Prediction model. Without this, the pipeline will run on CPU (which is much slower).

### Model Caching

The first time you run EvOlf, it will download \~15GB of model weights (Hugging Face transformers).

  * **Location:** These are stored in `~/.evolf_cache` in your home directory.
  * **Benefit:** They are downloaded **only once**. All subsequent runs (even in different project folders) will use this centralized cache.

-----

## Troubleshooting

**Q: `Permission denied: '/.cache'`**

  * **Fix:** This usually happens if the cache directory isn't set correctly. Ensure you haven't manually modified the `modules/` scripts. The pipeline is configured to use `~/.evolf_cache` automatically.

**Q: `CUDA out of memory`**

  * **Fix:** Deep learning models are memory-hungry. The pipeline automatically limits GPU jobs (`maxForks = 1`) to prevent crashing, but if you still see this, try reducing the batch size in your input file or running on a GPU with more VRAM (16GB+ recommended).

**Q: `docker: invalid reference format`**

  * **Fix:** You are likely using an older version of Nextflow or Docker. Ensure you are using `-profile docker` and not trying to build the image yourself. The pipeline pulls pre-built images from Docker Hub.

-----

## Credits & Citations

The EvOlf Pipeline was developed by the **Ahuja Lab** at IIIT-Delhi.

### Principal Investigator

  * **Dr. Gaurav Ahuja** ([@the-ahuja-lab](https://github.com/the-ahuja-lab))

### Lead Developers

  * **Adnan Raza** ([@woosflex](https://github.com/woosflex))
  * **Syed Yasser** ([@yasservision24](https://github.com/yasservision24))
  * **Pranjal Sharma** ([@PRANJAL2208](https://github.com/PRANJAL2208))
  * **Saveena Solanki** ([@SaveenaSolanki](https://github.com/SaveenaSolanki))
  * **Ayushi Mittal** ([@Aayushi006](https://github.com/Aayushi006))

### Related Repositories

  * **Pipeline (This Repo):** [evolf-pipeline](https://github.com/the-ahuja-lab/evolf-pipeline)
  * **Source Code & Development:** [evolf-pipeline-source](https://www.google.com/search?q=https://github.com/the-ahuja-lab/evolf-pipeline-source)

-----
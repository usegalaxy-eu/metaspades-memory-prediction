# MetaSPAdes Memory Prediction from k-mer statistics, biome metadata, and input file size to reduce retry-driven failures and wasted GB·h in production workflows.

This project aims to improve resource efficiency in metagenomic assembly by replacing static memory heuristics with data-driven predictors that reduce failures, retries, and over-allocation. The workflow:
1. builds k-mer, file-size, and biome features from MetaSPAdes jobs,
2. evaluates multiple machine-learning models with both regression and retry-aware cost metrics, and 
3. compares them against production heuristics under realistic workload distributions.

## Inputs

This analysis used 40,231 metagenome assembly jobs processed by EMBL-EBI between X and Y with the MGnify Genomes Generation pipeline (vX). Records included primary accession, sample biome, peak memory usage (GB), assembler, and assembler version (`data/mgnify_assembly_stats.csv` file). To reduce confounding technical variation, the dataset was restricted to MetaSPAdes v3.15.3, resulting in 9,102 jobs. From these, 300 jobs were randomly selected for model development, with 30 samples per major biome

## Process

### Requirements

- Python 3.10+
- Jupyter Notebook / JupyterLab
- Jellyfish (v2.3.0) for k-mer counting
- Core Python packages:
   - numpy
   - pandas
   - matplotlib
   - scikit-learn
   - scipy
   - joblib
   - adjustText

Install dependencies in your environment:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Feature generation

1. Map primary accessions to SRA accessions through the EMBL-EBI API by running [add_SSR_to_assembly_stats.ipynb](bin/add_SSR_to_assembly_stats.ipynb) notebook
2. Use Galaxy to download raw read, processed with Jellyfish (v2.3.0) using k=10 (default parameters otherwise) and compute k-mer statistical features with kmer2stats (v1.0.1) ([Galaxy History](https://usegalaxy.eu/u/paulzierep/h/kmer-counting-subset-3-15-3-metaspades-v2))
   1. Upload SRR ID file to Galaxy
   2. Run [SRR to kmer workflow](https://usegalaxy.eu/u/paulzierep/w/srr-to-kmer-statistics)
3. Download [kmer statistics](results/updated_mgnify_assemblies_stats_v3.15.3_metaspades_kmer10_stats.csv) into `results` folder
4. Add file size to kmer stats using [`bin/add_file_size_to_kmer_stats.ipynb` notebook](bin/add_file_size_to_kmer_stats.ipynb)

### Evaluate models 

Using scikit-learn (v1.8.0), three model families were evaluated:

1. Random Forest Regressor
2. Linear Regression
3. Custom Gaussian Process regressor with integrated preprocessing, optional log-transform of the target, predictive uncertainty estimates, and quantile-based allocations (0.9 and 0.99)

Feature sets included combinations of k-mer statistics, input file size, and biome. Two production heuristics were also evaluated:

1. Galaxy memory heuristic (step-based, file-size-driven)
2. MGnify heuristic (biome mapping-table based)

In total, 18 predictors were compared using out-of-fold predictions from five-fold shuffled cross-validation on the 300-job dataset.

Predictive performance was measured with RMSE and R², then translated into operational cost via a retry-aware memory-waste function.

Both densities were estimated with 60 log-spaced histogram bins over the combined memory range (~1-300 GB). For each model, 4,000 Monte Carlo iterations were run, each sampling 1,000 jobs with replacement according to $w_i$. Results are reported as mean and 95% interval (2.5-97.5 percentiles) of wasted GB·h per 1,000 jobs.


7. **Run evaluation notebook** using [`evaluation_metrics.ipynb` notebook](bin/evaluation_metrics.ipynb) which uses the following scripts:
   1. [`bin/gaussian_process.py`](bin/gaussian_process.py) to: 
      - Loads feature/label tables and prepares features.
      - Trains a Gaussian Process regressor for memory prediction.
      - Produces predictive means and safety quantiles (for safer allocations).
      - Includes helpers for cross-validation, model persistence, and inference.
   2. [`bin/evaluation_metrics.py`](bin/evaluation_metrics.py) to:
      - Implements retry policy simulation and job cost accounting.
      - Computes allocation failure, over-allocation, waste, and total wall-time metrics.
      - Provides per-job and batch-level metric utilities.
   3. [`bin/evaluate_memory_allocation_methods.py`](bin/evaluate_memory_allocation_methods.py) to:
      - Evaluates multiple predictors/heuristics under a common framework.
      - Applies importance weighting to debias sample-vs-population differences.
      - Runs weighted Monte Carlo estimates with confidence intervals.
      - Provides plotting helpers for method comparison.

## Results


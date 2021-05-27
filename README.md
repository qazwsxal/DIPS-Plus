# DIPS-Plus

The enhanced Database of Interacting Protein Structures (DIPS)

## How to run creation tools

First, install and configure Conda environment:

```bash
# Clone project:
git clone https://github.com/amorehead/DIPS-Plus

# Change to project directory:
cd DIPS-Plus

# (If on HPC cluster) Load 'open-ce' module
module load open-ce-1.1.3-py38-0

# (If on HPC cluster) Clone Conda environment into this directory using provided 'open-ce' environment:
conda create --name DIPS-Plus --clone open-ce-1.1.3-py38-0

# (If on HPC cluster - Optional) Create Conda environment in a particular directory using provided 'open-ce' environment:
conda create --prefix MY_VENV_DIR --clone open-ce-1.1.3-py38-0

# (Else, if on local machine) Set up Conda environment locally
conda env create --name DIPS-Plus -f environment.yml

# (Else, if on local machine - Optional) Create Conda environment in a particular directory using local 'environment.yml' file:
conda env create --prefix MY-VENV-DIR -f environment.yml

# Activate Conda environment located in the current directory:
conda activate DIPS-Plus

# (Optional) Activate Conda environment located in another directory:
conda activate MY-VENV-DIR

# (Optional) Deactivate the currently-activated Conda environment:
conda deactivate

# (If on local machine - Optional) Perform a full update on the Conda environment described in 'environment.yml':
conda env update -f environment.yml --prune

# (Optional) To remove this long prefix in your shell prompt, modify the env_prompt setting in your .condarc file with:
conda config --set env_prompt '({name})'
 ```

(If on HPC cluster) Install all project dependencies:

```bash
# Install project as a pip dependency in the Conda environment currently activated:
pip3 install -e .

# Install external pip dependencies in the Conda environment currently activated:
pip3 install -r requirements.txt

# Install pip dependencies used for unit testing in the Conda environment currently activated:
pip3 install -r tests/requirements.txt
 ```

## How to compile DIPS-Plus from scratch

Retrieve protein complexes from the RCSB PDB:

```bash
# Remove all existing training/testing sample lists
rm project/datasets/DIPS/final/raw/pairs-postprocessed.txt project/datasets/DIPS/final/raw/pairs-postprocessed-train.txt project/datasets/DIPS/final/raw/pairs-postprocessed-val.txt project/datasets/DIPS/final/raw/pairs-postprocessed-test.txt

# Create data directories (if not already created):
mkdir project/datasets/DIPS/raw project/datasets/DIPS/raw/pdb project/datasets/DIPS/interim project/datasets/DIPS/interim/external_feats project/datasets/DIPS/final project/datasets/DIPS/final/raw project/datasets/DIPS/final/processed

# Download the raw PDB files:
rsync -rlpt -v -z --delete --port=33444 --include='*.gz' --include='*.xz' --include='*/' --exclude '*' \
rsync.rcsb.org::ftp_data/biounit/coordinates/divided/ project/datasets/DIPS/raw/pdb

# Extract the raw PDB files:
python3 project/datasets/builder/extract_raw_pdb_gz_archives.py project/datasets/DIPS/raw/pdb

# Process the raw PDB data into associated pair files:
python3 project/datasets/builder/make_dataset.py project/datasets/DIPS/raw/pdb project/datasets/DIPS/interim --num_cpus 28 --source_type rcsb --bound

# Apply additional filtering criteria:
python3 project/datasets/builder/prune_pairs.py project/datasets/DIPS/interim/pairs project/datasets/DIPS/filters project/datasets/DIPS/interim/pairs-pruned --num_cpus 28

# Generate externally-sourced features:
python3 project/datasets/builder/generate_psaia_features.py "$PSAIADIR" "$PROJDIR"/project/datasets/builder/psaia_config_file_dips.txt "$PROJDIR"/project/datasets/DIPS/raw/pdb "$PROJDIR"/project/datasets/DIPS/interim/parsed "$PROJDIR"/project/datasets/DIPS/interim/pairs-pruned "$PROJDIR"/project/datasets/DIPS/interim/external_feats --source_type rcsb
python3 project/datasets/builder/generate_hhsuite_features.py "$PROJDIR"/project/datasets/DIPS/interim/parsed "$PROJDIR"/project/datasets/DIPS/interim/pairs-pruned "$HHSUITE_DB" "$PROJDIR"/project/datasets/DIPS/interim/external_feats --num_cpu_jobs 4 --num_cpus_per_job 8 --num_iter 2 --source_type rcsb --write_file

# Add new features to the filtered pairs, ensuring that the pruned pairs' original PDB files are stored locally for DSSP:
python3 project/datasets/builder/download_missing_pruned_pair_pdbs.py "$PROJDIR"/project/datasets/DIPS/raw/pdb "$PROJDIR"/project/datasets/DIPS/interim/pairs-pruned --num_cpus 32 --rank "$1" --size "$2"
python3 project/datasets/builder/postprocess_pruned_pairs.py "$PROJDIR"/project/datasets/DIPS/raw/pdb "$PROJDIR"/project/datasets/DIPS/interim/pairs-pruned "$PROJDIR"/project/datasets/DIPS/interim/external_feats "$PROJDIR"/project/datasets/DIPS/final/raw --num_cpus 32

# Partition dataset filenames, aggregate statistics, and impute missing features
python3 project/datasets/builder/partition_dataset_filenames.py "$PROJDIR"/project/datasets/DIPS/final/raw --source_type rcsb --filter_by_seq_length True --max_seq_length 1000 --rank "$1" --size "$2"
python3 project/datasets/builder/collect_dataset_statistics.py "$PROJDIR"/project/datasets/DIPS/final/raw --rank "$1" --size "$2"
python3 project/datasets/builder/impute_missing_feature_values.py "$PROJDIR"/project/datasets/DIPS/final/raw --num_cpus 32 --rank "$1" --size "$2"
```

## How to assemble DB5-Plus

Fetch prepared protein complexes from Dataverse:

```bash
# Download the prepared DB5 files:
wget -O project/datasets/DB5.tar.gz https://dataverse.harvard.edu/api/access/datafile/:persistentId?persistentId=doi:10.7910/DVN/H93ZKK/BXXQCG

# Extract downloaded DB5 archive:
tar -xzf project/datasets/DB5.tar.gz --directory project/datasets/

# Remove (now) redundant DB5 archive and other miscellaneous files:
rm project/datasets/DB5.tar.gz project/datasets/DB5/.README.swp
rm project/datasets/DB5.tar.gz "$MYLOCAL"/datasets/DB5/.README.swp
rm -rf project/datasets/DB5/interim "$MYLOCAL"/datasets/DB5/processed

# Create relevant interim and final data directories:
mkdir project/datasets/DB5/interim project/datasets/DB5/interim/external_feats
mkdir project/datasets/DB5/final project/datasets/DB5/final/raw project/datasets/DB5/final/processed

# Construct DB5 dataset pairs:
python3 project/datasets/builder/make_dataset.py "$PROJDIR"/project/datasets/DB5/raw "$PROJDIR"/project/datasets/DB5/interim --num_cpus 32 --source_type db5 --unbound

# Generate externally-sourced features:
python3 project/datasets/builder/generate_psaia_features.py "$PSAIADIR" "$PROJDIR"/project/datasets/builder/psaia_config_file_db5.txt "$PROJDIR"/project/datasets/DB5/raw "$PROJDIR"/project/datasets/DB5/interim/parsed "$PROJDIR"/project/datasets/DB5/interim/parsed "$PROJDIR"/project/datasets/DB5/interim/external_feats --source_type db5
python3 project/datasets/builder/generate_hhsuite_features.py "$PROJDIR"/project/datasets/DB5/interim/parsed "$PROJDIR"/project/datasets/DB5/interim/parsed "$HHSUITE_DB" "$PROJDIR"/project/datasets/DB5/interim/external_feats --num_cpu_jobs 4 --num_cpus_per_job 8 --num_iter 2 --source_type db5 --write_file

# Add new features to the filtered pairs:
python3 project/datasets/builder/postprocess_pruned_pairs.py "$PROJDIR"/project/datasets/DB5/raw "$PROJDIR"/project/datasets/DB5/interim/pairs "$PROJDIR"/project/datasets/DB5/interim/external_feats "$PROJDIR"/project/datasets/DB5/final/raw --num_cpus 32 --source_type db5

# Partition dataset filenames, aggregate statistics, and impute missing features
python3 project/datasets/builder/partition_dataset_filenames.py "$PROJDIR"/project/datasets/DB5/final/raw --source_type db5 --rank "$1" --size "$2"
python3 project/datasets/builder/collect_dataset_statistics.py "$PROJDIR"/project/datasets/DB5/final/raw --rank "$1" --size "$2"
python3 project/datasets/builder/impute_missing_feature_values.py "$PROJDIR"/project/datasets/DB5/final/raw --num_cpus 32 --rank "$1" --size "$2"
```

## Python 2 to 3 pickle file solution

While using Python 3 in this project, you may encounter the following error if you try to postprocess '.dill' pruned
pairs that were created using Python 2.

ModuleNotFoundError: No module named 'dill.dill'

1. To resolve it, ensure that the 'dill' package's version is greater than 0.3.2.
2. If the problem persists, edit the pickle.py file corresponding to your Conda environment's Python 3 installation (
   e.g. ~/DIPS-Plus/venv/lib/python3.8/pickle.py) and add the statement

```python
if module == 'dill.dill': module = 'dill._dill'
```

to the end of the

```python
if self.proto < 3 and self.fix_imports:
```

block in the Unpickler class' find_class() function
(e.g. line 1577 of Python 3.8.5's pickle.py).

### Citation

```
@article{DIPS-Plus,
  title={DIPS-Plus: The Enhanced Database of Interacting Protein Structures},
  author={Alex Morehead, Chen Chen, Ada Sedova, and Jianlin Cheng},
  journal={Datasets of Machine Learning Research},
  year={2021}
}
```

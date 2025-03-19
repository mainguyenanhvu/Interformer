Interformer
--------------------
Interformer is a protein-ligand complex structure prediction neural network that can predict interaction-aware energy functions for each pair of protein-ligand atoms. Such energy functions can be used in traditional protein-ligand docking sampling methods (Monte Carlo) to generate high-quality and reasonable binding poses.

The second application of the model involves using a module for pose-sensitive affinity and pose score based on contrastive learning. 
This  module assigns confidence scores to the generated docking poses and predicts corresponding affinity values. Notably, the lower the quality of the docking pose, the more probably predicting the lower affinity value (pIC50).

Below are the instructions for setting up and using the proposed method.

## Table of Contents

- [Installation](#set-up)
- [Preparation](#prepare)
- [Inference of Docking Pipeline](#docking)
- [Inference of PoseScore and Affinity Pipeline](#affinity)
- [Data Analysis](#Data-Analysis)
- [Training](#training)
- [Conclusion](#conclusion)

<a id="set-up"></a>

### Installation

Create Interformer environment using conda

```
cd Interformer
conda env create --file environment.yml
conda activate interformer

# Install PLIP without wrong openbabel dependency
pip install --no-deps plip

# Compile Docking sampling program from source code
cd docking && python setup.py install && cd ../
```


### Docking and PoseScore/Affinity Pipeline
<a id="prepare"></a>
Here we are using the data from `examples/` as demo.

0. Raw data preparation

The preparation of ligand and protein data involves several steps. First, hydrogen atoms are added to the ligand, and its protonation states are determined using [OpenBabel](https://github.com/openbabel/openbabel). Following this, the initial ligand conformation is generated using the UFF force field with [RDKit](https://github.com/rdkit/rdkit).

For the protein, preprocessing is carried out using the [Reduce](https://github.com/rlabduke/reduce) program, which includes the addition of hydrogen atoms along with their protonation states. The protein is then segmented into a pocket based on a 10Å radius around a reference ligand.

It should be noted that these procedures can be substituted with any other computational drug design (CADD) software. The prediction quality of the model is directly proportional to the quality of the preprocessing steps. Therefore, any improvements in this stage will likely enhance the model's predictive capabilities.

```
# Ligand
# 2qbr_ligand.sdf is the reference ligand used for locating the binding pocket, it can be replaced by a docking pose inside the binding pocket.
# To protonate and add hydrogen atoms to the reference ligand.
obabel examples/raw/2qbr_ligand.sdf -p 7.4 -O examples/ligand/2qbr_docked.sdf

# Generate inital ligand conformation using UFF (or any other ligand prepare program of your choice).  
python tools/rdkit_ETKDG_3d_gen.py examples/ligand/ examples/uff  
####
# Protein
# Use the Reduce program to preprocess the entire protein.
mkdir -p examples/raw/pocket && reduce examples/raw/2qbr.pdb > examples/raw/pocket/2qbr_reduce.pdb

# Extract the pocket within 10 Å around the reference ligand. The third argument 1 indicates removal of the CCD ligand from the PDB, use 0 if you do not wish to remove it.
python tools/extract_pocket_by_ligand.py examples/raw/pocket/ examples/ligand/ 1 && mv examples/raw/pocket/output/2qbr_pocket.pdb examples/pocket
```

1. Input data strcutures

The prepared data files should be saved as the following file structure and all files are required.

Firstly the program will locate files in the `ligand/`, `pocket/`, and `uff/` folders using the `$PDB` retrieved from `Target` column in `demo_dock.csv`.

Secondly, the reference ligand is located by using `pose_rank` column (default=0, representing the nth molecule in the SDF file), if you want to use the name of the SDF file to locate the reference ligand, please use parameter `--use_mid`.

Thirdly, the program will intercept a new binding pocket based on `$PDB_pocket.pdb` (ensure that the reference ligand is removed; this file can be the entire protein PDB file) using the `$PDB_docked.sdf` with 7A distance. Any cofactor such as Mg, Zn, or other cofactor ligand will be included in the binding pocket. Notably, the reference ligand should be as same as the UFF ligand, but if you want to dock a variety of different molecules using a distinct reference ligand, you can use parameter `--uff_as_ligand`. For more details, please refer to [virtual screening](https://github.com/tencent-ailab/Interformer/tree/master/applications/virtual_screening).

Finally, if everything is correct, the program will perform the prediction without any errors. For more details, please refer to [bindingdata.py](https://github.com/tencent-ailab/Interformer/blob/master/interformer/data/dataset/bindingdata.py).
```
examples/
├── demo_dock.csv  # the query csv for interformer prediction [Target=PDB, Molecule ID=name in SDF file, pose_rank=the nth molecule in sdf file]
├── ligand/$PDB_docked.sdf  # [reference ligand] foler contains the reference ligand conformation from PDB
├── pocket/$PDB_pocket.pdb # [binding pocket site] foler contains the target protein PDB structure 
├── uff/$PDB_uff.sdf # [initial ligand conformation] foler contains a single ligand conformation minimized by field foce
```
<a id="docking"></a>
2. Predicting energy functions file. Download checkpoints from [zenodo](https://zenodo.org/doi/10.5281/zenodo.10828798).

```
DOCK_FOLDER=energy_output

PYTHONPATH=interformer/ python inference.py -test_csv examples/demo_dock.csv \
-work_path examples/ \
-ensemble checkpoints/v0.2_energy_model \
-batch_size 1 \
-posfix *val_loss* \
-energy_output_folder $DOCK_FOLDER \
-reload \
-debug
```

It will produce an energy_output folder.

```
energy_output/
├── complex  # pocket intercepted through reference ligand
├── gaussian_predict  # predicted energy functions file
├── ligand  # copy from $work_path ligand folder [used for locate an autobox (20Ax20Ax20A sampling space) from reference ligand]
└── uff  # copy from $work_path uff folder
```

3. Giving the energy files produce docking poses via MonteCarlo sampling.

```
# Start docking
# If you planning to use uff ligand conformation to dock, you can use argument `--uff_folder uff`
OMP_NUM_THREADS="64,64" python docking/reconstruct_ligands.py -y --cwd $DOCK_FOLDER -y --find_all find

# Make a docking summary csv 
python docking/reconstruct_ligands.py --cwd $DOCK_FOLDER --find_all stat

# Merging original csv with the docking summary, gather information of rmsd, enery, num_torsions and poserank(cid, id of the conformation in a sdf)
python docking/merge_summary_input.py $DOCK_FOLDER/ligand_reconstructing/stat_concated.csv examples/demo_dock.csv

# You can review docking poses in the file located at $DOCK_FOLDER/ligand_reconstructing/$PDB_docked.sdf. 
# The 21st conformation is the inital input ligand with its binding pocket, which is provided for reference only.
```
<a id="affinity"></a>
4. Giving the docking poses, predicting its PoseScore and Affinity values.

```
# Copy docking pose to the $work_path
mkdir -p examples/infer && cp -r $DOCK_FOLDER/ligand_reconstructing/*.sdf examples/infer

# Scoring the docking pose, this step will generate a tmp_beta folder. Ensure that you delete this cache before running a new prediction.
PYTHONPATH=interformer/ python inference.py -test_csv examples/demo_dock.round0.csv \
-work_path examples/ \
-ligand_folder infer/ \
-ensemble checkpoints/v0.2_affinity_model/model* \
-gpus 1 \
-batch_size 20 \
-posfix *val_loss* \
--pose_sel True

# you can review your scoring results on the docking pose
# energy=the predicted total sum of energy for each pair of protein-ligand atoms (the lower is better), pred_pIC50=the predicted pIC50 affinity value, pred_pose=the predicted PoseScore(a confident score for the docking pose)
# Each prediction can be used for comparision across different ligands within the same protein or even between various protein systems.
cat result/demo_dock.round0_ensemble.csv 
```

### FAQ

1. Reproduce the time-split, core-set, and PoseBusters docking pose from the reported paper. The parepared data in the Benchmark.zip on [zenodo](https://zenodo.org/doi/10.5281/zenodo.10828798). 

```
bash scripts/infer/docking_timesplit.sh
bash scripts/infer/docking_posebuster.sh
```
2. Reproduce the affinity benchmark results from the reported paper

```
bash scripts/infer/affinity.sh
```

<a id="Data-Analysis"></a>

### Data Analysis

please review the readme file in eda/ folder.

<a id="training"></a>

### Training

```
# Please review these scripts for further deatils
# The training data is uploaded to zenodo.
# poses/ contains all structure need for training, poses/ligand contains the re-docking poses generated by interformer energy, poses/ligand/rcsb contains the conformation of reference ligand, poses/pocket contains all pocket extract by raw PDB from rcsb, poses/uff contains all ligand conformation minimized using UFF from reference ligand, and train/ contains the training csv.
bash scripts/train/preprocess.sh # used for preprocess training data
bash scripts/train/train.sh # train on one single machine
```

<a id="conclusion"></a>

### Conclusion

We anticipate the Interformer will be a powerful tool in practical drug development. In future work, we plan to enhance the model by incorporating more critical specific interactions and expanding its application to a broader range of biomolecular systems, including protein-nucleic acid and protein-protein interactions.


### Citation
```
@article{Lai2024,
  title = {Interformer: an interaction-aware model for protein-ligand docking and affinity prediction},
  volume = {15},
  ISSN = {2041-1723},
  url = {http://dx.doi.org/10.1038/s41467-024-54440-6},
  DOI = {10.1038/s41467-024-54440-6},
  number = {1},
  journal = {Nature Communications},
  publisher = {Springer Science and Business Media LLC},
  author = {Lai,  Houtim and Wang,  Longyue and Qian,  Ruiyuan and Huang,  Junhong and Zhou,  Peng and Ye,  Geyan and Wu,  Fandi and Wu,  Fang and Zeng,  Xiangxiang and Liu,  Wei},
  year = {2024},
  month = nov 
}
```

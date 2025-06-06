<p align="center">
  <h1 align="center">Reason3D: Searching and Reasoning 3D Segmentation via Large Language Model [3DV 2025]
  </h1>
  <p align="center">
    <a href="https://kuanchihhuang.github.io/"><strong>Kuan-Chih Huang</strong></a>,
    <a href="https://lxtgh.github.io/"><strong>Xiangtai Li</strong></a>,
    <a href="https://luqi.info/"><strong>Lu Qi</strong></a>,
    <a href="https://yanshuicheng.info/"><strong>Shuicheng Yan</strong></a>,
    <a href="https://faculty.ucmerced.edu/mhyang/"><strong>Ming-Hsuan Yang</strong></a>
  </p>

<div align="center">

[![arXiv](https://img.shields.io/badge/arXiv-2405.17427-red)](https://arxiv.org/abs/2405.17427)
[![Project](https://img.shields.io/badge/project-page-green)](https://kuanchihhuang.github.io/project/reason3d/)

</div>

## 🔥 Update
- 2025/01/19:  Initial code for 3D referring segmentation has been released.
- 2025/04/05:  Code and dataset for 3D reasoning segmentation has been released.
- 2025/05/18:  We release the hierarchical searching code in the [search branch](https://github.com/KuanchihHuang/Reason3D/tree/search).


## Overview

<img src="figs/reason3d_arch.jpg" alt="vis" style="zoom:50%;" />

We introduce Reason3D, a novel LLM for comprehensive 3D understanding that processes point cloud data and text prompts to produce textual responses and segmentation masks. This enables advanced tasks such as 3D reasoning segmentation, hierarchical searching, referring expressions, and question answering with detailed mask outputs.

## Installation

1. Create conda environment. We use `python=3.8` `pytorch=1.11.0` and `cuda=11.3`.
```bash
conda create -n reason3d python=3.8
conda activate reason3d
conda install pytorch==1.11.0 torchvision==0.12.0 torchaudio==0.11.0 cudatoolkit=11.3 -c pytorch
pip install -r requirements.txt
```

2. Install [LAVIS](https://github.com/salesforce/LAVIS)
```bash
git clone https://github.com/salesforce/LAVIS.git SalesForce-LAVIS
cd SalesForce-LAVIS
pip install -e .
```

3. Install segmentor from this [repo](https://github.com/Karbo123/segmentator) (used for superpoint construction). We also provide an alternative PyTorch implementation `segmentator_pytorch.py`, though it may yield slightly lower performance.


4. Install pointgroup_ops
```bash
cd lavis/models/reason3d_models/lib
sudo apt-get install libsparsehash-dev
python setup.py develop
```

## Data Preparation

### ScanNet v2 dataset

Download the [ScanNet](http://www.scan-net.org/) v2 dataset.

Put the downloaded `scans` folder as follows.
```
Reason3D
├── data
│   ├── scannetv2
│   │   ├── scans
```

Split and preprocess point cloud data for 3D referring and 3D reasoning segmentation tasks:

```
cd data/scannetv2
bash prepare_data.sh #Scanrefer
bash prepare_data_reason.sh #Reason3D
```

After running the script, the scannetv2 dataset structure should look like below.
```
Reason3D
├── data
│   ├── scannetv2
│   │   ├── scans
│   │   ├── train
│   │   │   ├──XXX_refer.pth
│   │   │   ├──XXX_reason.pth
│   │   ├── val
```
You can directly download our preprocessed data ([train.zip](https://drive.google.com/file/d/1Y41Y6H0To9qB71kUlLISYn4RhvwFe4KZ/view) and [val.zip](https://drive.google.com/file/d/1y9MSXFGh80W46201bbgCoW95go5DJY7k/view)), please agree the official license before download it.

### ScanRefer dataset

Download [ScanRefer](https://github.com/daveredrum/ScanRefer) annotations.

```
Reason3D
├── data
│   ├── ScanRefer
│   │   ├── ScanRefer_filtered_train.json
│   │   ├── ScanRefer_filtered_val.json
```

### Matterport3D dataset

Please follow the instructions [here](https://niessner.github.io/Matterport/) to access official `download_mp.py` script, run the following in `data/matterport/`:
```
python2 download_mp.py -o . --type region_segmentations
```
Extract files and organize data as follows:
```
Reason3D
├── data
│   ├── matterport
│   │   ├── scans
│   │   │   ├── 17DRP5sb8fy
│   │   │   │   ├──region_segmentations
│   │   │   │   │   ├──region0.ply
│   │   │   ├── ...
```
Process data on Matterport3D dataset for 3D reasoning segmentation task:
```
cd data/matterport
python3 process_mp3d.py
```
After running the script, the Matterport3D dataset structure should look like below.
```
Reason3D
├── data
│   ├── matterport
│   │   ├── mp3d_data
│   │   │   ├── XXXXX_regionX.pth
│   │   │   ├── ...
```
You can directly download our preprocessed data ([mp3d_data.zip](https://drive.google.com/file/d/1OXT_hmv-9eHgqpcl3A0V28y-DfC5v0-y/view)), please agree the official license before download it.

### Reason3D dataset

Download our Reason3D annotations [here](https://drive.google.com/file/d/1Z4kzr_4oJxTJgzILaycDuHlRYeZCXOu6/view).

```
Reason3D
├── data
│   ├── reason3d
│   │   ├── reason3d_train.json
│   │   ├── reason3d_val.json
```

## Pretrained Backbone
Download the [SPFormer](https://github.com/sunjiahao1999/SPFormer) pretrained backbone (or provided by [3D-STMN](https://github.com/sosppxo/3D-STMN)) and move it to checkpoints.
```
mkdir checkpoints
mv ${Download_PATH}/sp_unet_backbone.pth checkpoints/
```
You can also pretrain the backbone by yourself and modify the path [here](lavis/projects/reason3d/train/reason3d_scanrefer_scratch.yaml#L15).

## Training
- **3D referring segmentation:** Train on ScanRefer dataset from scratch:
```
python -m torch.distributed.run --nproc_per_node=4 --master_port=29501 train.py --cfg-path lavis/projects/reason3d/train/reason3d_scanrefer_scratch.yaml
```
- **3D reasoning segmentation:** Train on Reason3D dataset using the pretrained checkpoint from the 3D referring segmentation model:
```
python -m torch.distributed.run --nproc_per_node=2 --master_port=29501 train.py --cfg-path lavis/projects/reason3d/train/reason3d_reason.yaml --options model.pretrained=<path_to_pretrained_checkpoint>
```
Replace `<path_to_pretrained_checkpoint>` with the path to your pretrained 3D referring segmentation model. For example: `./lavis/output/reason3d/xxxx/checkpoint_xx.pth`


## Evaluation
- **3D referring segmentation:** Evaluate on ScanRefer dataset: 
```
python evaluate.py --cfg-path lavis/projects/reason3d/val/reason3d_scanrefer_scratch.yaml --options model.pretrained=<path_to_pretrained_checkpoint> run.save_results=True
```
Note: this repo currently only supports batch size = 1 for inference. 

- **3D reasoning segmentation:** Evaluate on our Reason3D dataset: 
```
python evaluate.py --cfg-path lavis/projects/reason3d/val/reason3d_reason.yaml --options model.pretrained=<path_to_pretrained_checkpoint> run.save_results=True
```
Add `run.save_results=True` option if you want to save prediction results.

We provide a pre-trained [checkpoint](https://drive.google.com/file/d/1FEKy5uu70Z3S5GCDjnt8VX8cXB1m9eVx/view?usp=sharing) for 3D Reasoning segmentation task. See the below table to check the performance.

|                   |      Sample Number | mIoU      | Acc50     | Acc25     |
| ----------------- |  ----------------- | --------- | --------- | --------- |
| ScanNet           |          308       |   0.32    |    0.32   |   0.44     |
| Matterport3D      |          837       |   0.22    |    0.21   |   0.33     | 

## Visualization

You can visualize prediction results using:
```
python visualize.py --idx <sample_index> --result_dir <results_directory>
```
`<sample_index>`: Index of the sample you wish to display. `<results_directory>`: Path to either the `reason_preds` or `refer_preds` directory containing the results.

## Results

<img src="figs/visualization.jpg" alt="vis" style="zoom:50%;" />


## TODO List

- [x] Release the initial code for 3D referring segmentation task.
- [X] Release final version paper.
- [X] Release the dataset and code for 3D reasoning segmentation task.
- [X] Release hierarchical mask decoder code. 
- [X] Release demo and visualization code.
- [ ] ...

## Acknowlegment

Our codes are mainly based on [LAVIS](https://github.com/salesforce/LAVIS), [3D-LLM](https://github.com/UMass-Foundation-Model/3D-LLM) and [3D-STMN](https://github.com/sosppxo/3D-STMN). Thanks for their contributions!


## Citation

If you find our work useful for your project, please consider citing our paper:


```bibtex
@article{reason3d,
  title={Reason3D: Searching and Reasoning 3D Segmentation via Large Language Model},
  author={Kuan-Chih Huang and Xiangtai Li and Lu Qi and Shuicheng Yan and Ming-Hsuan Yang},
  journal={3DV},
  year={2025}
}
```

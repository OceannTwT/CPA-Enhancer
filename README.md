#  🚀 Introduction

**Abstract** : Object detection methods under known single degradations have been extensively investigated. However, existing approaches require prior knowledge of the degradation type and train a separate model for each, limiting their practical applications in unpredictable environments. To address this challenge, we propose a chain-of-thought (CoT) prompted adaptive enhancer, CPA-Enhancer, for object detection under unknown degradations. Specifically, CPA-Enhancer progressively adapts its enhancement strategy under the step-by-step guidance of CoT prompts, that encode degradation-related information. To the best of our knowledge, it’s the first work that exploits CoT prompting for object detection tasks. Overall, CPA-Enhancer is a plug-and-play enhancement model that can be integrated into any generic detectors to achieve substantial gains on degraded images, without knowing the degradation type priorly. Experimental results demonstrate that CPA-Enhancer not only sets the new state of the art for object detection but also boosts the performance of other downstream vision tasks under multiple unknown degradations.

# 🛠️ Installation

- **Step0**. Download and install [Miniconda](https://docs.anaconda.com/free/miniconda/) from the official website.
- **Step1**. Create a conda environment and activate it.

```shell
conda create --name openmmlab python=3.8 -y
conda activate openmmlab
```

- **Step2**.Install PyTorch following [official instructions](https://pytorch.org/get-started/locally/), e.g.

```shell
conda install pytorch==1.11.0 torchvision==0.12.0 torchaudio==0.11.0 cudatoolkit=11.3 -c pytorch
```

- **Step3**. Install [MMEngine](https://github.com/open-mmlab/mmengine) and [MMCV](https://github.com/open-mmlab/mmcv) using [MIM](https://github.com/open-mmlab/mim).

```shell
pip install -U openmim
mim install mmengine
mim install "mmcv>=2.0.0"
```

- **Step4**. Install other related packages.

```shell
cd CPA_Enhancer
pip install -r ./cpa/requirements.txt
```

# 📁 Data Preparation

## Synthetic Datasets

- **Step1**. Download [VOC PASCAL](http://host.robots.ox.ac.uk/pascal/VOC/) trainval and test data

```shell
$ wget http://host.robots.ox.ac.uk/pascal/VOC/voc2007/VOCtrainval_06-Nov-2007.tar
$ wget http://host.robots.ox.ac.uk/pascal/VOC/voc2012/VOCtrainval_11-May-2012.tar
$ wget http://host.robots.ox.ac.uk/pascal/VOC/voc2007/VOCtest_06-Nov-2007.tar
```

- **Step2**.   Construct `VnA-T` ( containing 5 categories, with a total of 8111 images) / `VnB-T` (containing 10 categories, with a total of 12334 images) from `VOCtrainval_06-Nov-2007.tar` and `VOCtrainval_11-May-2012.tar`; Construct `VnA-T` ( containing 5 categories, with a total of 2734 images) / `VnB-T` (containing 10 categories, with a total of 3760 images) from `VOCtest_06-Nov-2007.tar`.  

  We also provide a list of image names included in each dataset, which you can find in the `cpa/dataSyn/datalist`.

```python
# 5 class
target_classes = ['person','car','bus','bicycle','motorbike']
# 10 class
target_classes = ['bicycle','boat','bottle','bus','car','cat','chair','dog','motorbike','person']
```

Make sure the directory follows this basic VOC structure.

```shell
data_vocnorm  (data_vocnorm_10) 	# path\to\vocnorm
├── train   # VnA-T (VnB-T)      
|    ├── Annotations
|    |    └──xxx.xml
|    |       ...
|    └── ImageSets
|    |    └──Main
|    |        └──train_voc.txt  # you can find it in cpa\dataSyn\datalist
|    └── JPEGImages
|         └──xxx.jpg
|            ...
├── test  # VnA (VnB)        
|    ├── Annotations
|    |    └──xxx.xml
|    |       ...
|    └── ImageSets
|    |    └──Main
|    |        └──test_voc.txt # you can find it in cpa\dataSyn\datalist
|    └── JPEGImages
|         └──xxx.jpg
|            ...
```

- **Step3.** Sythesize degraded datasets from VnA and VnB by executing the following command and restructure them into VOC format.

```shell
# Modify the paths in the code to match your actual paths.
# all-in-one setting 
python cpa/dataSyn/data_make_fog.py   		# VF/VF-T 
python cpa/dataSyn/data_make_lowlight.py 	# VD/VD-T/VDB
python cpa/dataSyn/data_make_snow.py 		# VS/VS-T
python cpa/dataSyn/data_make_rain.py 		# VR/VR-T
# one-by-one setting 
python cpa/dataSyn/data_make_fog_hybrid.py		 	# VF-HT
python cpa/dataSyn/data_make_lowlight_hybrid.py 	# VD-HT
```

## Real-world Datasets

- **Step1**. Download [Exdark](https://github.com/cs-chan/Exclusively-Dark-Image-Dataset/tree/master/Dataset) and [RTTS](https://sites.google.com/view/reside-dehaze-datasets/reside-%CE%B2) datasets.
- **Step2**. Restructure the RTTS dataset (4322 images) into VOC format, ensuring that the directory conforms to this basic structure.

```shell
RTTS          # path:  G:\dataset\detection\RTTS
├── Annotations
|    └──xxx.xml
|       ...
└── ImageSets
|    └──Main
|        └──test_rtts.txt
└── JPEGImages
     └──xxx.jpg
        ...
```

- **Step3**. Similarly, restructure the ExdarkA dataset (containing 5 categories, with a total of 1283 images) and the ExdarkB dataset (containing 10 categories, with a total of 2563 images) into VOC format.

```shell
exdark_5 (exdark_10)         # path:  G:\dataset\detection\ExDark\ExDarkA (ExDarkB)
├── Annotations
|    └──xxx.xml
|       ...
└── ImageSets
|    └──Main
|        └──test_exdark_5.txt (test_exdark_10.txt) # you can find it in cpa\dataSyn\datalist
└── JPEGImages
     └──xxx.jpg
        ...
```

# 🎯 Usage

## 📍 All-in-One Setting

- **Step 1.** Modify the `METAINFO` in `mmdet/datasets/voc.py`

````python
METAINFO = {
        'classes': ('person', 'car', 'bus', 'bicycle',  'motorbike'), # 5 classes
        'palette': [(106, 0, 228), (119, 11, 32), (165, 42, 42), (0, 0, 192),(197, 226, 255)]
    }
````

- **Step 2.** Modify the `voc_classes` in `mmdet/evaluation/functional/class_names.py`

```python
def voc_classes() -> list:
    return [
        'person', 'car', 'bus', 'bicycle',  'motorbike' # 5 classes
    ]
```

- **Step 3.** Modify the `num_classes` in `configs\yolo\cpa_config.py`

```python
bbox_head=dict(
        type='YOLOV3Head',
        num_classes=5, # 5 classes
				...
)
```

- **Step 4.** Recompile the code.

```
cd CPA_Enhancer
pip install -v -e .
```

- **Step 5.** Modify the `data_root` ,`ann_file`and `data_prefix` in `configs\yolo\cpa_config.py` to match your actual paths of the used datasets.

🔹 **Train** 

```shell
# Train our model from scratch.  
python tools/train.py configs/yolo/cpa_config.py  
```

🔹 **Test**

```shell
# you can download our pretrained model for testing 
python tools/test.py configs/yolo/cpa_config.py path/to/checkpoint/xx.pth
```

🔹 **Demo**

```shell
# you can download our pretrained model for inference
python demo/cpa_demo.py \
	--inputs ../cpa/testimage  # path to your input images or dictionary
	--model ../configs/yolo/cpa_config.py 
	--weights path/to/checkpoint/xx.pth 
	--out-dir ../cpa/output # output file
```

## 📍 One-by-One Setting

For the foggy conditions (containing five categories), the overall process is the same as above (Step1-5).

For the low-light conditions ( containing ten categories ) , You only need to modify a few places as follows (Step1-3).

- **Step 1.** Modify the `METAINFO` in `mmdet/datasets/voc.py`

````python
# 10 classes
METAINFO = {
        'classes': ('bicycle', 'boat', 'bottle','bus', 'car', 'cat', 'chair','dog','motorbike','person'),
        'palette': [(106, 0, 228), (119, 11, 32), (165, 42, 42), (0, 0, 192),(197, 226, 255),
										(0, 60, 100), (0, 0, 142), (255, 77, 255), (153, 69, 1), (120, 166, 157),]
    }
````

- **Step 2.** Modify the `voc_classes` in `mmdet/evaluation/functional/class_names.py`

```python
def voc_classes() -> list:
    return [
        'bicycle', 'boat', 'bottle','bus', 'car', 'cat', 'chair','dog','motorbike','person' # 10 classes
    ]
```

- **Step 3.** Modify the `num_classes` in `configs/yolo/cpa_config.py`

```python
bbox_head=dict(
        type='YOLOV3Head',
        num_classes=10, # 10 classes
				...
)
```


# 📊 Results

You can download our training and testing logs.

# 💐 Acknowledgments

Special thanks to the creators of [mmdetection](https://github.com/open-mmlab/mmdetection) upon which this code is built, for their valuable work in advancing object detection research.

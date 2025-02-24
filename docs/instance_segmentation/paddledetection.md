---
title: PaddleDetection (instance segmentation)
---

[PaddleDetection](https://github.com/PaddlePaddle/PaddleDetection) is an Object Detection toolkit based on PaddlePaddle. 
It supports object detection, instance segmentation, multiple object tracking and real-time multi-person keypoint detection. 
Custom docker images with additional tools are available from here:

[https://github.com/waikato-datamining/paddledetection](https://github.com/waikato-datamining/paddledetection)


# Prerequisites
Make sure you have the directory structure created as outlined in the [Prerequisites](../prerequisites.md).


# Data

In this example, we will use the [Oxford Pets](https://datasets.cms.waikato.ac.nz/ufdl/oxford-pets/)
dataset, which consists of 37 different categories of cats and dogs.

Download the dataset from the following URL into the *data* directory and extract it:

[https://datasets.cms.waikato.ac.nz/ufdl/data/oxford-pets/oxford-pets-adams.zip](https://datasets.cms.waikato.ac.nz/ufdl/data/oxford-pets/oxford-pets-adams.zip)

Rename the `adams` directory to `pets-adams`. 

To speed up training, we only use two labels: `cat:abyssinian` and `dog:yorkshire_terrier`.
The label filtering and splitting it into *train*, *validation* and *test* subsets is done 
using [image-dataset-converter](https://github.com/waikato-datamining/image-dataset-converter):

```bash
docker run --rm -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatodatamining/image-dataset-converter:0.0.4 \
  idc-convert \
    -l INFO \
    from-adams-od \
      -i "/workspace/data/pets-adams/*.report" \
    filter-labels \
      --labels cat:abyssinian dog:yorkshire_terrier \
    discard-negatives \
    coerce-mask \
    to-coco-od \
      -o /workspace/data/pets2-coco-split \
      --categories cat:abyssinian dog:yorkshire_terrier \
      --category_output_file labels.txt \
      --split_names train val test \
      --split_ratios 70 15 15
```


# Training

For training, we will use the following docker image:

```
waikatodatamining/paddledetection:2.8.0_cuda11.8
```

The training script is called `paddledet_train`, for which we can invoke the help screen as follows:

```bash
docker run --rm \
  -t waikatodatamining/paddledetection:2.8.0_cuda11.8 \
  paddledet_train --help 
```

It is good practice creating a separate sub-directory for each training run, with a directory name that hints at
what dataset and model were used. So for our first training run, which will use mainly default parameters, we will 
create the following directory in the `output` folder:

```
pets2-paddledet-maskrcnn
```

Before we can train, we will need to obtain and customize a config file. Within the container,
you can find example configurations for various architectures in the following directory:

```
/opt/PaddleDetection/configs
```

Using the `paddledet_export_config` command, we can expand and dump one of these configurations for our
own purposes:

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache/visualdl:/.visualdl \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache:/opt/PaddleDetection/~/.cache \
  -t waikatodatamining/paddledetection:2.8.0_cuda11.8 \
  paddledet_export_config \
  -i /opt/PaddleDetection/configs/mask_rcnn/mask_rcnn_r50_1x_coco.yml \
  -o /workspace/output/pets2-paddledet-maskrcnn/mask_rcnn_r50_1x_coco.yml \
  -O /workspace/output/pets2-paddledet-maskrcnn \
  -t /workspace/data/pets2-coco-split/train/annotations.json \
  -v /workspace/data/pets2-coco-split/val/annotations.json \
  --save_interval 10 \
  --num_epochs 30 \
  --num_classes 2
```

Open the `mask_rcnn_r50_1x_coco.yml` file in a text editor and perform the following operations:

* change `base_lr` to `0.0005`


Kick off the training with the following command:

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache/visualdl:/.visualdl \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache:/opt/PaddleDetection/~/.cache \
  -t waikatodatamining/paddledetection:2.8.0_cuda11.8 \
  paddledet_train \
  -c /workspace/output/pets2-paddledet-maskrcnn/mask_rcnn_r50_1x_coco.yml \
  --eval \
  -o use_gpu=true
```

Export the model using the `paddledet_export_model` script:

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache/visualdl:/.visualdl \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache:/opt/PaddleDetection/~/.cache \
  -t waikatodatamining/paddledetection:2.8.0_cuda11.8 \
  paddledet_export_model \
  -c /workspace/output/pets2-paddledet-maskrcnn/mask_rcnn_r50_1x_coco.yml \
  --output_dir /workspace/output/pets2-paddledet-maskrcnn/inference
```


# Predicting

Using the `paddledet_predict_poll` script, we can batch-process images placed in the `predictions/in` directory
as follows (e.g., from our *test* subset): 

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache/visualdl:/.visualdl \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache:/opt/PaddleDetection/~/.cache \
  -t waikatodatamining/paddledetection:2.8.0_cuda11.8 \
  paddledet_predict_poll \
  --model_path /workspace/output/pets2-paddledet-maskrcnn/inference/mask_rcnn_r50_1x_coco \
  --device gpu \
  --label_list /workspace/data/pets2-coco-split/train/labels.txt \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out \
  --threshold 0.3 \
  --mask_nth 2
```

**Notes** 

* The predictions get output in [OPEX JSON format](https://github.com/WaikatoLink2020/objdet-predictions-exchange-format),
  which you can view the predictions with the ADAMS *Preview browser*:
  
    * [OPEX](../../previewing_predictions/#objdet_opex)


**Example prediction**

![Screenshot](img/paddledet-Abyssinian_117.png) ![Screenshot](img/paddledet-Abyssinian_117-overlay.png)

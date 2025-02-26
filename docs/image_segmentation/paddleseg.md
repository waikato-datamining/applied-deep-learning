---
title: PaddleSeg
---

[PaddleSeg](https://github.com/PaddlePaddle/PaddleSeg) is an easy-to-use image segmentation library with 
a large pre-trained model zoo, supporting wide-range of practical tasks in Semantic Segmentation, Interactive 
Segmentation, Panoptic Segmentation, Image Matting, 3D Segmentation, etc.  Custom docker images with 
additional tools are available from here:

[https://github.com/waikato-datamining/paddleseg](https://github.com/waikato-datamining/paddleseg)


# Prerequisites
Make sure you have the directory structure created as outlined in the [Prerequisites](../prerequisites.md).


# Data

In this example, we will use the [CamVid-12](https://datasets.cms.waikato.ac.nz/ufdl/camvid12/)
dataset, which consists of still images from dashcam videos from a city environment (12 different labels).

Download the dataset from the following URL into the *data* directory and extract it:

[https://datasets.cms.waikato.ac.nz/ufdl/data/camvid12/camvid12-grayscale.zip](https://datasets.cms.waikato.ac.nz/ufdl/data/camvid12/camvid12-grayscale.zip)

Once extracted, rename the *grayscale* directory to *camvid-grayscale*.

Now we have to convert the format from *grayscale* into *indexed PNG*, which PaddleSeg uses.
We can do this by using the [image-dataset-converter](https://github.com/waikato-datamining/image-dataset-converter) library. 
At the same time, we can split the dataset into *train*, *validation* and *test* subsets.

From within the `applied_deep_learning` directory, run the following command:

```bash
docker run --rm -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatodatamining/image-dataset-converter:0.0.7 \
  idc-convert \
    -l INFO \
    from-grayscale-is \
      -i "/workspace/data/camvid-grayscale/*.png" \
      --labels unlabelled sky building pole road pavement tree signsymbol fence car pedestrian bicyclist \
      --background 11 \
    to-paddle-is \
      -o /workspace/data/camvid-paddleseg-split \
      --split_names train val test \
      --split_ratios 70 15 15
```


# Training

For training, we will use the following docker image:

```
waikatodatamining/paddleseg:2.10.0_cuda11.8
```

Training and inference are possible on just a CPU as well (though much, much slower).
For utilizing a CPU you can use the following docker image:

```
waikatodatamining/paddleseg:2.10.0_cpu
```

The training script is called `paddleseg_train`, for which we can invoke the help screen as follows:

```bash
docker run --rm -t waikatodatamining/paddleseg:2.10.0_cuda11.8 paddleseg_train --help 
```

It is good practice creating a separate sub-directory for each training run, with a directory name that hints at
what dataset and model were used. So for our first training run, which will use mainly default parameters, we will 
create the following directory in the `output` folder:

```
camvid12-paddleseg-unet
```

Before we can train, we will need to obtain and customize a config file. Within the container,
you can find example configurations for various architectures in the following directory:

```
/opt/PaddleSeg/configs
```

Using the `paddleseg_export_config` command, we can expand and dump one of these configurations for our
own purposes:

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.paddleseg \
  -v `pwd`/cache/visualdl:/.visualdl \
  -t waikatodatamining/paddleseg:2.10.0_cuda11.8 \
  paddleseg_export_config \
  -i /opt/PaddleSeg/configs/unet/unet_cityscapes_1024x512_160k.yml \
  -o /workspace/output/camvid12-paddleseg-unet/unet_cityscapes_1024x512_160k.yml \
  -c `cat data/camvid-paddleseg-split/labels.txt | wc -l` \
  -t /workspace/data/camvid-paddleseg-split/data-train.txt \
  -v /workspace/data/camvid-paddleseg-split/data-val.txt
```


Kick off the training with the following command:

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  --shm-size=8g \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.paddleseg \
  -v `pwd`/cache/visualdl:/.visualdl \
  -t waikatodatamining/paddleseg:2.10.0_cuda11.8 \
  paddleseg_train \
  --config /workspace/output/camvid12-paddleseg-unet/unet_cityscapes_1024x512_160k.yml \
  --save_interval 500 \
  --do_eval \
  --save_dir /workspace/output/camvid12-paddleseg-unet \
  --iters 1000
```

**NB:** You may want to increase the number of iterations to produce a more accurate model.


# Predicting

Using the `paddleseg_predict_poll` script, we can batch-process images placed in the `predictions/in` directory
as follows (e.g., from our *test* subset): 

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.paddleseg \
  -v `pwd`/cache/visualdl:/.visualdl \
  -t waikatodatamining/paddleseg:2.10.0_cuda11.8 \
  paddleseg_predict_poll \
  --model_path /workspace/output/camvid12-paddleseg-unet/best_model/model.pdparams \
  --config /workspace/output/camvid12-paddleseg-unet/unet_cityscapes_1024x512_160k.yml \
  --labels /workspace/data/camvid-paddleseg-split/labels.txt \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out
```

**Notes** 

* The predictions can either be output in *grayscale* (default), *bluechannel* format,
  *indexed PNG* or *OPEX JSON* (`--prediction_format`).
* You can view the predictions with the ADAMS *Preview browser*:
  
    * [Indexed PNG](../../previewing_predictions.md/#imgseg_indexed)

**Example prediction**

![Screenshot](img/paddleseg-0016E5_05310.png) 

![Screenshot](img/paddleseg-0016E5_05310-overlay.png)

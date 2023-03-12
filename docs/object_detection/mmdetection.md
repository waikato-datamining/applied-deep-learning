---
title: MMDetection (object detection)
---

[MMDetection](https://github.com/open-mmlab/mmdetection) is a comprehensive and flexible
framework for object detection that offers a wide variety of architectures. Custom docker
images with additional tools are available from here:

[https://github.com/waikato-datamining/mmdetection](https://github.com/waikato-datamining/mmdetection)


# Prerequisites
Make sure you have the directory structure created as outlined in the [Prerequisites](../prerequisites.md).


# Data

In this example, we will use the [American Sign Language Letters](https://datasets.cms.waikato.ac.nz/ufdl/american-sign-language-letters/)
dataset, which consists of sets of images of hands, one per letter in the English alphabet (26 labels).

Download the dataset from the following URL into the *data* directory and extract it:

[https://datasets.cms.waikato.ac.nz/ufdl/data/american-sign-language-letters/american-sign-language-letters-voc.zip](https://datasets.cms.waikato.ac.nz/ufdl/data/american-sign-language-letters/american-sign-language-letters-voc.zip)

Once extracted, rename the *voc* directory to *sign-voc*.

Now we have to convert the format from *VOC XML* into *MS COCO*. We can do this by using the 
[wai.annotations](https://github.com/waikato-ufdl/wai-annotations) library. 
At the same time, we can split the dataset into *train*, *validation* and *test* subsets.

From within the `applied_deep_learning` directory, run the following command:

```bash
docker run --rm -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatoufdl/wai.annotations:0.8.0 \
  wai-annotations convert \
    from-voc-od \
      -i "/workspace/data/sign-voc/*.xml" \
    to-coco-od \
      -o /workspace/data/sign-coco-split/annotations.json \
      --sort-categories \
      --category-output-file labels.txt \
      --split-names train val test \
      --split-ratios 70 15 15
```


# Training

For training, we will use the following docker image:

```
waikatodatamining/mmdetection:2.27.0_cuda11.1
```

Inference is possible without a GPU as well (though much, much slower).
For utilizing a CPU you can use the following docker image:

```
waikatodatamining/mmdetection:2.27.0_cpu
```

The training script is called `mmdet_train`, for which we can invoke the help screen as follows
(unfortunately, we need to set the `MMDET_CLASSES` environment variable to avoid an exception):

```bash
docker run --rm \
  -e MMDET_CLASSES= \
  -t waikatodatamining/mmdetection:2.27.0_cuda11.1 \
  mmdet_train --help 
```

It is good practice creating a separate sub-directory for each training run, with a directory name that hints at
what dataset and model were used. So for our first training run, which will use mainly default parameters, we will 
create the following directory in the `output` folder:

```
sign-mmdet-fr50
```

Before we can train, we will need to obtain and customize a config file. Within the container,
you can find example configurations for various architectures in the following directory:

```
/mmdetection/configs
```

Using the `mmdet_config` command, we can expand and dump one of these configurations for our
own purposes:

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache/torch:/.cache/torch \
  -t waikatodatamining/mmdetection:2.27.0_cuda11.1 \
  mmdet_config \
  /mmdetection/configs/faster_rcnn/faster_rcnn_r50_fpn_1x_coco.py \
  > output/sign-mmdet-fr50/faster_rcnn_r50_fpn_1x_coco.py
```

Open the `faster_rcnn_r50_fpn_1x_coco.py` file in a text editor and perform the following operations:

* remove any lines before `model = dict(`
* change `num_classes` to 26
* change `dataset_type` to `Dataset` and any occurrences of `type` in the `train`, `test`, `val` sections of the `data` dictionary
* change `data_root` occurrences to `/workspace/data/sign-coco-split` (the directory above the `train` and `val` directories)
* change `img_prefix` occurrences to `img_prefix=data_root+'/DIR',` with `DIR` being the appropriate `train`, `val` or `test`
* change `ann_file` occurrences to `ann_file=data_root+'/DIR/annotations.json',` with `DIR` being the appropriate `train`, `val` or `test`
* change `max_epochs` in `runner` to an appropriate value, e.g., 5
* change `interval` in `checkpoint_config` to a higher value, e.g., 5


Kick off the training with the following command:

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache/torch:/.cache/torch \
  -e MMDET_CLASSES=/workspace/data/sign-coco-split/train/labels.txt \
  -t waikatodatamining/mmdetection:2.27.0_cuda11.1 \
  mmdet_train \
  /workspace/output/sign-mmdet-fr50/faster_rcnn_r50_fpn_1x_coco.py \
  --work-dir /workspace/output/sign-mmdet-fr50
```


# Predicting

Using the `mmdet_predict` script, we can batch-process images placed in the `predictions/in` directory
as follows (e.g., from our *test* subset): 

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache/torch:/.cache/torch \
  -e MMDET_CLASSES=/workspace/data/sign-coco-split/train/labels.txt \
  -t waikatodatamining/mmdetection:2.27.0_cuda11.1 \
  mmdet_predict \
  --checkpoint /workspace/output/sign-mmdet-fr50/latest.pth \
  --config /workspace/output/sign-mmdet-fr50/faster_rcnn_r50_fpn_1x_coco.py \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out
```

**Notes** 

* By default, the predictions get output in [ROI CSV format](https://github.com/waikato-ufdl/wai-annotations-roi).
  But you can also output them in the [OPEX JSON format](https://github.com/WaikatoLink2020/objdet-predictions-exchange-format) 
  by adding `--prediction_format opex --prediction_suffix .json` to the command.

* You can view the predictions with the ADAMS *Preview browser*:
  
    * [ROIS CSV](../../previewing_predictions/#objdet_rois)
    * [OPEX](../../previewing_predictions/#objdet_opex)

**Example prediction**

![Screenshot](img/mmdet-A2_jpg.rf.e4d1f7a2679ab0140ad27a794db563c9-overlay.png) 

![Screenshot](img/mmdet-I0_jpg.rf.9dc9f090b7475a65889b24265858d240-overlay.png)

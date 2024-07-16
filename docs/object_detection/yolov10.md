---
title: Yolov10
---

[Yolov10](https://github.com/THU-MIG/yolov10) represents the implementation of 
[YOLOv10: Real-Time End-to-End Object Detection](https://arxiv.org/abs/2405.14458). 
Custom docker images with additional tools are available from here:

[https://github.com/waikato-datamining/pytorch/tree/master/yolov10](https://github.com/waikato-datamining/pytorch/tree/master/yolov10)


# Prerequisites
Make sure you have the directory structure created as outlined in the [Prerequisites](../prerequisites.md).


# Data

In this example, we will use the [American Sign Language Letters](https://datasets.cms.waikato.ac.nz/ufdl/american-sign-language-letters/)
dataset, which consists of sets of images of hands, one per letter in the English alphabet (26 labels).

Download the dataset from the following URL into the *data* directory and extract it:

[https://datasets.cms.waikato.ac.nz/ufdl/data/american-sign-language-letters/american-sign-language-letters-voc.zip](https://datasets.cms.waikato.ac.nz/ufdl/data/american-sign-language-letters/american-sign-language-letters-voc.zip)

Once extracted, rename the *voc* directory to *sign-voc*.

Now we have to convert the format from *VOC XML* into *YOLO*. We can do this by using the 
[image-dataset-converter](https://github.com/waikato-datamining/image-dataset-converter) library. 
At the same time, we can split the dataset into *train*, *validation* and *test* subsets.

From within the `applied_deep_learning` directory, run the following command:

```bash
docker run --rm -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatodatamining/image-dataset-converter:0.0.4 \
  idc-convert \
    -l INFO \
    from-voc-od \
      -i "/workspace/data/sign-voc/*.xml" \
    to-yolo-od \
      -o /workspace/data/sign-yolo-split \
      --labels labels.txt \
      --labels_csv labels.csv \
      --split_names train val test \
      --split_ratios 70 15 15
```

Finally, download the [dataset10.yaml](img/dataset10.yaml) file, place it in the `sign-yolo-split`
directory. It contains information about the dataset directory, the splits and the class labels.
If you want to adapt this configuration for different labels, then you can automatically 
transform the `labels.csv` file into a numbered list using the following command:

```bash
grep -v "Index" labels.csv | sed s/","/": "/g | sed s/^/"  "/g
```


# Training

For training, we will use the following docker image:

```
waikatodatamining/pytorch-yolov10:2024-06-23_cuda11.7
```

If you do not have a GPU, you can use the CPU-only image:

```
waikatodatamining/pytorch-yolov10:2024-06-23_cpu
```

The training script is called `yolov10_train`, for which we can invoke the help screen as follows:

```bash
docker run --rm -t waikatodatamining/pytorch-yolov10:2024-06-23_cuda11.7 yolov10_train help 
```

Since we will be performing transfer larning, we need to download a base model to use for training. 
Yolov10 offers different models, which differ in speed and accuracy. We will use the *fastest* one 
called `yolov10n.pt` (*"nano"*) from the `v1.1` release:

[https://github.com/THU-MIG/yolov10/releases/download/v1.1/yolov10n.pt](https://github.com/THU-MIG/yolov10/releases/download/v1.1/yolov10n.pt)

Download it and place it in the `models` directory.

It is good practice creating a separate sub-directory for each training run, with a directory name that hints at
what dataset and model were used. So for our first training run, which will use mainly default parameters, we will 
create the following directory in the `output` folder:

```
sign-yolov10
```

Since the image size should be a multiple of 32, we use 640 for this experiment.

Kick off the training with the following command:

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -t waikatodatamining/pytorch-yolov10:2024-06-23_cuda11.7 \
  yolov10_train \
  model=/workspace/models/yolov10n.pt \
  data=/workspace/data/sign-yolo-split/dataset10.yaml \
  imgsz=640 \
  exist_ok=true \
  project=/workspace/output/ \
  name=sign-yolov10 \
  amp=false \
  batch=4 \
  epochs=20
```


# Predicting

Using the `yolov10_predict_poll` script, we can batch-process images placed in the `predictions/in` directory
as follows (e.g., from our *test* subset): 

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  --gpus=all \
  -v `pwd`:/workspace \
  -t waikatodatamining/pytorch-yolov10:2024-06-23_cuda11.7 \
  yolov10_predict_poll \
  --model /workspace/output/sign-yolov10/weights/best.pt \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out
```

**Notes** 

* You can view the predictions with the ADAMS *Preview browser*: [OPEX](../../previewing_predictions/#objdet_opex)


**Example prediction**

![Screenshot](img/yolov10-A2_jpg.rf.e4d1f7a2679ab0140ad27a794db563c9.jpg) 

![Screenshot](img/yolov10-I6_jpg.rf.e777252b29b845d02452566d242b3e6b.jpg)


# Exporting to ONNX (optional)

Before we can use our trained model, we will need to export it to [ONNX format](https://onnx.ai/)
using the `yolov10_export` script:

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  --gpus=all \
  -v `pwd`:/workspace \
  -t waikatodatamining/pytorch-yolov10:2024-06-23_cuda11.7 \
  yolov10_export \
  model=/workspace/output/sign-yolov10/weights/best.pt \
  format=onnx \
  opset=13 \
  simplify
```

This will create a file called `best.onnx` in the weights directory.


# Troubleshooting

* If you are re-using a dataset that was used by another YolovX framework, you
  may get strange error messages when reading the data. This can be due to 
  incompatible cache files that get generated to speed up loading the data. 
  Make sure to remove all files in the `labels` directory that have a `.cache` 
  extension.
  
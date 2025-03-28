---
title: MMClassification
---

[MMClassification](https://github.com/open-mmlab/mmclassification) is a comprehensive and flexible
framework for image segmentation that offers a wide variety of models. Custom docker
images with additional tools are available from here:

[https://github.com/waikato-datamining/mmclassification](https://github.com/waikato-datamining/mmclassification)

# Prerequisites
Make sure you have the directory structure created as outlined in the [Prerequisites](../prerequisites.md).


# Data

In this example, we will use the [102 flowers](http://datasets.cms.waikato.ac.nz/ufdl/image_classification/102flowers/)
dataset, which consists of 102 different categories (~ species) of flowers. More precisely, we will download the
dataset with the flowers already split into categories from which we will use a subset to speed up the training process.

Download the dataset from the following URL into the *data* directory and extract it:

[https://datasets.cms.waikato.ac.nz/ufdl/data/102flowers/102flowers-subdir.zip](https://datasets.cms.waikato.ac.nz/ufdl/data/102flowers/102flowers-subdir.zip)

Once extracted, you can delete all sub-directories apart from:

* alpine_sea_holly
* anthurium
* artichoke
* azalea
* ball_moss

Rename the `subdir` directory to `5flowers` and move it into the `data` folder of our directory structure 
outlined. 

Split the data into *train*, *validation* and *test* subsets using 
[image-dataset-converter](https://github.com/waikato-datamining/image-dataset-converter):

```bash
docker run --rm -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatodatamining/image-dataset-converter:0.0.4 \
  idc-convert \
    -l INFO \
    from-subdir-ic \
      -i "/workspace/data/5flowers/" \
    to-subdir-ic \
      -o /workspace/data/5flowers-split \
      --split_names train val test \
      --split_ratios 70 15 15
```


# Training

For training, we will use the following docker image:

```
waikatodatamining/mmclassification:0.25.0_cuda11.1
```

If you only have a CPU machine available, then use this one instead:

```
waikatodatamining/mmclassification:0.25.0_cpu
```

The training script is called `mmcls_train`, for which we can invoke the help screen as follows:

```bash
docker run --rm -t waikatodatamining/mmclassification:0.25.0_cuda11.1 mmcls_train --help   # GPU
docker run --rm -t waikatodatamining/mmclassification:0.25.0_cpu mmcls_train --help        # CPU
```

It is good practice creating a separate sub-directory for each training run, with a directory name that hints at
what dataset and model were used. So for our first training run, which will use mainly default parameters, we will 
create the following directory in the `output` folder:

```
5flowers-mmcl-r18
```


Before we can train, we will need to obtain and customize a config file. Within the container,
you can find example configurations for various architectures in the following directory:

```
/mmclassification/configs
```

Using the `mmcls_config` command, we can expand and dump one of these configurations for our
own purposes:

GPU:

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache/torch:/.cache/torch \
  -t waikatodatamining/mmclassification:0.25.0_cuda11.1 \
  mmcls_config \
  /mmclassification/configs/resnet/resnet18_b32x8_imagenet.py \
  > `pwd`/output/5flowers-mmcl-r18/resnet18_b32x8_imagenet.py
```

CPU:

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache/torch:/.cache/torch \
  -t waikatodatamining/mmclassification:0.25.0_cpu \
  mmcls_config \
  /mmclassification/configs/resnet/resnet18_b32x8_imagenet.py \
  > `pwd`/output/5flowers-mmcl-r18/resnet18_b32x8_imagenet.py
```

Open the `resnet18_b32x8_imagenet.py` file in a text editor and perform the following operations:

* remove the lines before `model = dict(`
* change `num_classes` to `5`
* change `dataset_type` to `ExternalDataset` and any occurrences of `type` in the `train`, `test`, `val` sections of the `data` dictionary
* change `data_prefix` to `/workspace/data/5flowers-split/train`, `../test` and `../val` in the relevant sections
* change `ann_file` occurrences to `None`
* change `max_epochs` in the `runner` to a suitable value, e.g., `10`
* change the `interval` of the `checkpoint_config` to a value that makes sense with `max_epochs`, e.g., `5`  
* if we want to perform transfer learning, we can re-use a pretrained model from the [model zoo](https://mmclassification.readthedocs.io/en/latest/model_zoo.html) 
  and just need to specify it in the `backbone` dictionary, by inserting the following before the `style='pytorch'` setting (see also 
  [documentation](https://mmclassification.readthedocs.io/en/latest/tutorials/finetune.html)):

```
        init_cfg=dict(
            type='Pretrained',
            checkpoint='https://download.openmmlab.com/mmclassification/v0/resnet/resnet18_batch256_imagenet_20200708-34ab8f90.pth',
            prefix='backbone',
        ),
```

GPU:

```bash
docker run --rm \
  -u $(id -u):$(id -g) -e USER=$USER \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache/torch:/.cache/torch \
  -e MMCLS_CLASSES=alpine_sea_holly,anthurium,artichoke,azalea,ball_moss \
  -t waikatodatamining/mmclassification:0.25.0_cuda11.1 \
  mmcls_train \
  /workspace/output/5flowers-mmcl-r18/resnet18_b32x8_imagenet.py \
  --work-dir /workspace/output/5flowers-mmcl-r18/runs
```

CPU:

```bash
docker run --rm \
  -u $(id -u):$(id -g) -e USER=$USER \
  --shm-size 8G \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache/torch:/.cache/torch \
  -e MMCLS_CLASSES=alpine_sea_holly,anthurium,artichoke,azalea,ball_moss \
  -t waikatodatamining/mmclassification:0.25.0_cpu \
  mmcls_train \
  /workspace/output/5flowers-mmcl-r18/resnet18_b32x8_imagenet.py \
  --work-dir /workspace/output/5flowers-mmcl-r18/runs
```


# Predicting

Using the `mmcls_predict_poll` script, we can batch-process images placed in the `predictions/in` directory
as follows (e.g., from our *test* subset): 

GPU:

```bash
docker run --rm \
  -u $(id -u):$(id -g) -e USER=$USER \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -e MMCLS_CLASSES=alpine_sea_holly,anthurium,artichoke,azalea,ball_moss \
  -t waikatodatamining/mmclassification:0.25.0_cuda11.1 \
  mmcls_predict_poll \
  --model /workspace/output/5flowers-mmcl-r18/runs/latest.pth \
  --config /workspace/output/5flowers-mmcl-r18/resnet18_b32x8_imagenet.py \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out
```

CPU:

```bash
docker run --rm \
  -u $(id -u):$(id -g) -e USER=$USER \
  --shm-size 8G \
  -v `pwd`:/workspace \
  -e MMCLS_CLASSES=alpine_sea_holly,anthurium,artichoke,azalea,ball_moss \
  -t waikatodatamining/mmclassification:0.25.0_cpu \
  mmcls_predict_poll \
  --model /workspace/output/5flowers-mmcl-r18/runs/latest.pth \
  --config /workspace/output/5flowers-mmcl-r18/resnet18_b32x8_imagenet.py \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out
```

E.g., for the `image_08085.jpg` from the `alpine_sea_holly` class, we will get a JSON file similar to 
[this one](img/image_08085.json):

```json
{
  "alpine_sea_holly": 0.9981447458267212,
  "anthurium": 1.6010968465707265e-05,
  "artichoke": 0.0015235934406518936,
  "azalea": 2.2820820504421135e-06,
  "ball_moss": 0.00031330727506428957
}
```

**Notes**

* You can view the predictions with the ADAMS *Preview browser*:
  
    * [Image classification (JSON)](../../previewing_predictions/#imgcls_json)


# Troubleshooting

* **RuntimeError: selected index k out of range**
  
    This can occur if you have less than 5 class labels.
    You need to update the config file as follows ([source](https://raw.githubusercontent.com/open-mmlab/mmclassification/master/docs/en/tutorials/MMClassification_python.ipynb)):

    * set `model/head/topk` to `(1, )` rather than `(1, 5)`
    * add `metric_options={'topk': (1, )` to the `evaluation` dictionary 

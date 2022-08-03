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
[wai.annotations](https://github.com/waikato-ufdl/wai-annotations):

```bash
docker run -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatoufdl/wai.annotations:latest \
  wai-annotations convert \
    from-subdir-ic \
      -i "/workspace/data/5flowers/**/*.jpg" \
    to-subdir-ic \
      -o /workspace/data/5flowers-split \
      --split-names train val test \
      --split-ratios 70 15 15
```


# Training

For training, we will use the following docker image:

```
waikatodatamining/mmclassification:0.23.1_cuda11.1
```

If you only have a CPU machine available, then use this one instead:

```
waikatodatamining/mmclassification:0.23.1_cpu
```

The training script is called `mmcls_train`, for which we can invoke the help screen as follows:

```bash
docker run -t waikatodatamining/mmclassification:0.23.1_cuda11.1 mmcls_train --help   # GPU
docker run -t waikatodatamining/mmclassification:0.23.1_cpu mmcls_train --help        # CPU
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

Using the `mmseg_config` command, we can expand and dump one of these configurations for our
own purposes:

CPU:

```bash
docker run \
  -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache/torch:/.cache/torch \
  -t waikatodatamining/mmclassification:0.23.1_cpu \
  mmcls_config \
  /mmclassification/configs/resnet/resnet18_b32x8_imagenet.py \
  > `pwd`/output/5flowers-mmcl-r18/resnet18_b32x8_imagenet.py
```

GPU:

```bash
docker run \
  -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache/torch:/.cache/torch \
  -t waikatodatamining/mmclassification:0.23.1_cuda11.1 \
  mmcls_config \
  /mmclassification/configs/resnet/resnet18_b32x8_imagenet.py \
  > `pwd`/output/5flowers-mmcl-r18/resnet18_b32x8_imagenet.py
```

Open the `resnet18_b32x8_imagenet.py` file in a text editor and perform the following operations:

* remove the lines before `model = dict(`
* change `num_classes` to `5`
* change `dataset_type` to `ExternalDataset` and any occurrences of `type` in the `train`, `test`, `val` sections of the `data` dictionary
* change `dataset_prefix` to `/workspace/data/5flowers-split/train`, `../test` and `../val` in the relevant sections
* change `ann_file` occurrences to `None`
* change `max_epochs` in the `runner` to a suitable value
* change the `interval` of the `checkpoint_config` to a value that makes sense with `max_epochs`  

GPU:

```bash
docker run \
  -u $(id -u):$(id -g) -e USER=$USER \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache/torch:/.cache/torch \
  -e MMCLS_CLASSES=alpine_sea_holly,anthurium,artichoke,azalea,ball_moss \
  -t waikatodatamining/mmclassification:0.23.1_cuda11.1 \
  mmcls_train \
  /workspace/output/5flowers-mmcl-r18/resnet18_b32x8_imagenet.py \
  --work-dir /workspace/output/5flowers-mmcl-r18
```

CPU:

```bash
docker run \
  -u $(id -u):$(id -g) -e USER=$USER \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.cache \
  -v `pwd`/cache/torch:/.cache/torch \
  -e MMCLS_CLASSES=alpine_sea_holly,anthurium,artichoke,azalea,ball_moss \
  -t waikatodatamining/mmclassification:0.23.1_cpu \
  mmcls_train \
  /workspace/output/5flowers-mmcl-r18/resnet18_b32x8_imagenet.py \
  --work-dir /workspace/output/5flowers-mmcl-r18
```


# Predicting

Using the `mmcls_predict_poll` script, we can batch-process images placed in the `predictions/in` directory
as follows (e.g., from our *test* subset): 

GPU:

```bash
docker run \
  -u $(id -u):$(id -g) -e USER=$USER \
  --gpus=all \
  -v `pwd`:/workspace \
  -e MMCLS_CLASSES=alpine_sea_holly,anthurium,artichoke,azalea,ball_moss \
  -t waikatodatamining/mmclassification:0.23.1_cuda11.1 \
  mmcls_predict_poll \
  --model /workspace/output/5flowers-mmcl-r18/latest.pth \
  --config /workspace/output/5flowers-mmcl-r18/resnet18_b32x8_imagenet.py \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out
```

CPU:

```bash
docker run \
  -u $(id -u):$(id -g) -e USER=$USER \
  -v `pwd`:/workspace \
  -e MMCLS_CLASSES=alpine_sea_holly,anthurium,artichoke,azalea,ball_moss \
  -t waikatodatamining/mmclassification:0.23.1_cpu \
  mmcls_predict_poll \
  --model /workspace/output/5flowers-mmcl-r18/latest.pth \
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

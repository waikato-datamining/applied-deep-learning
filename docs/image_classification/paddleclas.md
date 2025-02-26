---
title: PaddleClas
---

[PaddleClas](https://github.com/PaddlePaddle/PaddleClas) is a comprehensive and flexible
framework for image segmentation that offers a wide variety of models. Custom docker
images with additional tools are available from here:

[https://github.com/waikato-datamining/paddleclas](https://github.com/waikato-datamining/paddleclas)

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
  -t waikatodatamining/image-dataset-converter:0.0.7 \
  idc-convert \
    -l INFO \
    from-subdir-ic \
      -i "/workspace/data/5flowers/" \
    to-paddle-ic \
      -o /workspace/data/5flowers-paddlecls-split \
      --split_names train val test \
      --split_ratios 70 15 15
```


# Training

For training, we will use the following docker image:

```
waikatodatamining/paddleclas:2.6.0_cuda11.8
```

If you only have a CPU machine available, then use this one instead:

```
waikatodatamining/paddleclas:2.6.0_cpu
```

The training script is called `paddleclas_train`, for which we can invoke the help screen as follows:

```bash
docker run --rm -t waikatodatamining/paddleclas:2.6.0_cuda11.8 paddleclas_train --help   # GPU
docker run --rm -t waikatodatamining/paddleclas:2.6.0_cpu paddleclas_train --help        # CPU
```

It is good practice creating a separate sub-directory for each training run, with a directory name that hints at
what dataset and model were used. So for our first training run, which will use mainly default parameters, we will 
create the following directory in the `output` folder:

```
5flowers-paddleclas-r50
```


Before we can train, we will need to obtain and customize a config file. Within the container,
you can find example configurations for various architectures in the following directory:

```
/opt/PaddleClas/configs
```

Using the `paddleclas_export_config` command, we can expand and dump one of these configurations for our
own purposes:

GPU:

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.paddleclas \
  -v `pwd`/cache/visualdl:/.visualdl \
  -t waikatodatamining/paddleclas:2.6.0_cuda11.8 \
  paddleclas_export_config \
  -i /opt/PaddleClas/ppcls/configs/quick_start/ResNet50_vd.yaml \
  -o /workspace/output/5flowers-paddleclas-r50/ResNet50_vd.yaml \
  -t /workspace/data/5flowers-paddlecls-split/train/annotations.txt \
  -v /workspace/data/5flowers-paddlecls-split/val/annotations.txt \
  -l /workspace/data/5flowers-paddlecls-split/train/labels.map \
  -c `cat ./data/5flowers-paddlecls-split/train/labels.map | wc -l` \
  -e 20 \
  --save_interval 10 \
  -r Global.train_mode \
  -a Arch.pretrained:True \
  -O /workspace/output/5flowers-paddleclas-r50
```

CPU:

```bash
docker run --rm \
  -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.paddleclas \
  -v `pwd`/cache/visualdl:/.visualdl \
  -t waikatodatamining/paddleclas:2.6.0_cpu \
  paddleclas_export_config \
  -i /opt/PaddleClas/ppcls/configs/quick_start/ResNet50_vd.yaml \
  -o /workspace/output/5flowers-paddleclas-r50/ResNet50_vd.yaml \
  -t /workspace/data/5flowers-paddlecls-split/train/annotations.txt \
  -v /workspace/data/5flowers-paddlecls-split/val/annotations.txt \
  -l /workspace/data/5flowers-paddlecls-split/train/labels.map \
  -c `cat ./data/5flowers-paddlecls-split/train/labels.map | wc -l` \
  -e 20 \
  --save_interval 10 \
  -r Global.train_mode \
  -a Arch.pretrained:True \
  -O /workspace/output/5flowers-paddleclas-r50
```

Kick off training using the `paddleclas_train` script as follows:

GPU:

```bash
docker run --rm \
  -u $(id -u):$(id -g) -e USER=$USER \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.paddleclas \
  -v `pwd`/cache/visualdl:/.visualdl \
  -t waikatodatamining/paddleclas:2.6.0_cuda11.8 \
  paddleclas_train \
  -c /workspace/output/5flowers-paddleclas-r50/ResNet50_vd.yaml
```

CPU:

```bash
docker run --rm \
  -u $(id -u):$(id -g) -e USER=$USER \
  --shm-size 8G \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.paddleclas \
  -v `pwd`/cache/visualdl:/.visualdl \
  -t waikatodatamining/paddleclas:2.6.0_cpu \
  paddleclas_train \
  -c /workspace/output/5flowers-paddleclas-r50/ResNet50_vd.yaml
```


# Predicting

Using the `paddleclas_predict_poll` script, we can batch-process images placed in the `predictions/in` directory
as follows (e.g., from our *test* subset): 

GPU:

```bash
docker run --rm \
  -u $(id -u):$(id -g) -e USER=$USER \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.paddleclas \
  -v `pwd`/cache/visualdl:/.visualdl \
  -t waikatodatamining/paddleclas:2.6.0_cuda11.8 \
  paddleclas_predict_poll \
  --model_path /workspace/output/5flowers-paddleclas-r50/best_model \
  --config /workspace/output/5flowers-paddleclas-r50/ResNet50_vd.yaml \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out
```

CPU:

```bash
docker run --rm \
  -u $(id -u):$(id -g) -e USER=$USER \
  --shm-size 8G \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/.paddleclas \
  -v `pwd`/cache/visualdl:/.visualdl \
  -t waikatodatamining/paddleclas:2.6.0_cpu \
  paddleclas_predict_poll \
  --model_path /workspace/output/5flowers-paddleclas-r50/best_model \
  --config /workspace/output/5flowers-paddleclas-r50/ResNet50_vd.yaml \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out
```

E.g., for the `image_06982.jpg` from the `alpine_sea_holly` class, we will get a JSON file similar to 
[this one](img/image_06982.json):

```json
{"alpine_sea_holly": 0.99714, "anthurium": 0.00229, "artichoke": 0.00038, "ball_moss": 0.00012, "azalea": 7e-05}
```

**Notes**

* You can view the predictions with the ADAMS *Preview browser*:
  
    * [Image classification (JSON)](../../previewing_predictions/#imgcls_json)


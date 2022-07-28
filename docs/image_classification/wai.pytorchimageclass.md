With [wai.pytorchimageclass](https://github.com/waikato-datamining/pytorch/tree/master/image-classification)
it is possible to train various image classification network architectures and also perform inference.

The code is based on this Pytorch example:

[https://github.com/pytorch/examples/tree/master/imagenet](https://github.com/pytorch/examples/tree/49e1a8847c8c4d8d3c576479cb2fe2fd2ac583de/imagenet)


# Prerequisites
Make sure you have the directory structure created as outlined in the [Prerequisites](../prerequisites.md).


# Data

In this example, we will use the [102 flowers](http://datasets.cms.waikato.ac.nz/ufdl/image_classification/102flowers/)
dataset, which consists of 102 different categories (~ species) of flowers. More precisely, we will download the
dataset with the flowers already split into categories from which will use a subset to speed up the training process.

Download the dataset from the following URL and extract it:

[https://datasets.cms.waikato.ac.nz/ufdl/data/102flowers/102flowers-subdir.zip](https://datasets.cms.waikato.ac.nz/ufdl/data/102flowers/102flowers-subdir.zip)

Once extracted, you can delete all sub-directories apart from:

* alpine_sea_holly
* anthurium
* artichoke

Rename the `subdir` directory to `3flowers` and move it into the `data` folder of our directory structure 
outlined. 

Split the data into *train*, *validation* and *test* sets using a 70/15/15 split ratio:

```bash
docker run \
  -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatoufdl/wai.annotations:0.7.8 \
  wai-annotations convert \
    from-subdir-ic \
      -i "/workspace/data/3flowers/**/*.jpg" \
    to-subdir-ic \
      -o /workspace/data/3flowers-split \
      --split-names train val test \
      --split-ratios 70 15 15
```


# Training

For training, we will use the following docker image:

```
waikatodatamining/pytorch-image-classification:1.6
```

If you only have a CPU machine available, then use this one instead:

```
waikatodatamining/pytorch-image-classification:1.6_cpu
```

The training script is called `pic-main`, for which we can invoke the help screen as follows:

```bash
docker run -t waikatodatamining/pytorch-image-classification:1.6 pic-main --help      # GPU
docker run -t waikatodatamining/pytorch-image-classification:1.6_cpu pic-main --help  # CPU
```

It is good practice creating a separate sub-directory for each training run, with a directory name that hints at
what dataset and model were used. So for our first training run, which will use mainly default parameters, we will 
create the following directory in the `output` folder:

```
3flowers-pt-default
```

The following command, issued in the `applied_deep_learning` directory, will map the `applied_deep_learning`
directory (= the current working directory) onto the `/workspace` directory within the docker container, giving
us access to all the sub-directories there, and train our 3flowers dataset (`-u $(id -u):$(id -g)` maps the user 
and group ID):

GPU:

```bash
docker run \
  -u $(id -u):$(id -g) \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -t waikatodatamining/pytorch-image-classification:1.6 \
  pic-main \
  -t /workspace/data/3flowers-split/train \
  -T /workspace/data/3flowers-split/val \
  -o /workspace/output/3flowers-pt-default \
  -a resnet50 \
  -b 16 \
  --epochs 100
```

CPU:

```bash
docker run \
  -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatodatamining/pytorch-image-classification:1.6_cpu \
  pic-main \
  -t /workspace/data/3flowers-split/train \
  -T /workspace/data/3flowers-split/val \
  -o /workspace/output/3flowers-pt-default \
  -a resnet50 \
  -b 16 \
  --epochs 100
```


# Predicting

For making predictions for a single image, you can use the script `tfic-labelimage`.

Since we will want to batch predict multiple images, will use the script `tfic-poll` instead: 

GPU:

```bash
docker run \
  -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatodatamining/pytorch-image-classification:1.6 \
  pic-poll \
  --model /workspace/output/3flowers-pt-default/graph.tflite \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out
```

CPU:

```bash
docker run \
  -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatodatamining/pytorch-image-classification:1.6_cpu \
  pic-poll \
  --model /workspace/output/3flowers-pt-default/graph.tflite \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out
```

E.g., for the `image_01965.jpg` from the `anthurium` class, we will get a CSV file similar to 
[this one](img/image_01965.csv):

{{ read_csv('docs/image_classification/img/image_01965.csv') }}

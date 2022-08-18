---
title: tf2_make_image_classifier
---

The [make_image_classifier](https://github.com/tensorflow/hub/tree/master/tensorflow_hub/tools/make_image_classifier) 
Python library can be used for training various TensorFlow 2 image classification models that are available from 
[Tensorflow Hub](https://tfhub.dev/).

Code for the Docker images and additional Python code is available from here:

[https://github.com/waikato-datamining/tensorflow/tree/master/image_classification2](https://github.com/waikato-datamining/tensorflow/tree/master/image_classification2)


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

Rename the `subdir` directory to `3flowers` and move it into the `data` folder of our directory structure 
outlined. 


# Training

For training, we will use the following docker image:

```
waikatodatamining/tf_image_classification2:2.9.1_cuda11.1
```

If you only have a CPU machine available, then use this one instead:

```
waikatodatamining/tf_image_classification2:2.9.1_cpu
```

The training script is called `make_image_classifier`, for which we can invoke the help screen as follows:

```bash
docker run -t waikatodatamining/tf_image_classification2:2.9.1_cuda11.1 make_image_classifier --helpfull   # GPU
docker run -t waikatodatamining/tf_image_classification2:2.9.1_cpu make_image_classifier --helpfull        # CPU
```

It is good practice creating a separate sub-directory for each training run, with a directory name that hints at
what dataset and model were used. So for our first training run, which will use mainly default parameters, we will 
create the following directory in the `output` folder:

```
3flowers-tf2-default
```

The following command will train an [EfficientNet b0](https://tfhub.dev/google/imagenet/efficientnet_v2_imagenet21k_ft1k_b0/feature_vector/2) 
model for 10 epochs:

GPU:

```bash
docker run \
  -u $(id -u):$(id -g) -e USER=$USER \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/tmp/tfhub_modules \
  -t waikatodatamining/tf_image_classification2:2.9.1_cuda11.1 \
  make_image_classifier \
  --image_dir /workspace/data/3flowers \
  --image_size 224 \
  --saved_model_dir /workspace/output/3flowers-tf2-default \
  --labels_output_file /workspace/output/3flowers-tf2-default/labels.txt \
  --tflite_output_file /workspace/output/3flowers-tf2-default/model.tflite \
  --tfhub_module https://tfhub.dev/google/imagenet/efficientnet_v2_imagenet21k_ft1k_b0/feature_vector/2 \
  --train_epochs 10
```

CPU:

```bash
docker run \
  -u $(id -u):$(id -g) -e USER=$USER \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/tmp/tfhub_modules \
  -t waikatodatamining/tf_image_classification2:2.9.1_cpu \
  make_image_classifier \
  make_image_classifier \
  --image_dir /workspace/data/3flowers \
  --image_size 224 \
  --saved_model_dir /workspace/output/3flowers-tf2-default \
  --labels_output_file /workspace/output/3flowers-tf2-default/labels.txt \
  --tflite_output_file /workspace/output/3flowers-tf2-default/model.tflite \
  --tfhub_module https://tfhub.dev/google/imagenet/efficientnet_v2_imagenet21k_ft1k_b0/feature_vector/2 \
  --train_epochs 10
```


# Predicting

For making predictions for a single image, you can use the script `label_image`.

Since we will want to batch predict multiple images, will use the script `predict_poll` instead: 

GPU:

```bash
docker run \
  -u $(id -u):$(id -g) -e USER=$USER \
  -v `pwd`:/workspace \
  -t waikatodatamining/tf_image_classification2:2.9.1_cuda11.1 \
  predict_poll \
  --model /workspace/output/3flowers-tf2-default/model.tflite \
  --labels /workspace/output/3flowers-tf2-default/labels.txt \
  --input_mean 0 \
  --input_std 255 \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out
```

CPU:

```bash
docker run \
  -u $(id -u):$(id -g) -e USER=$USER \
  -v `pwd`:/workspace \
  -t waikatodatamining/tf_image_classification2:2.9.1_cpu \
  predict_poll \
  --model /workspace/output/3flowers-tf2-default/model.tflite \
  --labels /workspace/output/3flowers-tf2-default/labels.txt \
  --input_mean 0 \
  --input_std 255 \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out
```

E.g., for the `image_02048.jpg` from the `anthurium` class, we will get a JSON file similar to 
[this one](img/image_02048.json):

```json
{
  "anthurium": 0.8697358965873718,
  "alpine_sea_holly": 0.06686040759086609,
  "artichoke": 0.0634036660194397
}
```

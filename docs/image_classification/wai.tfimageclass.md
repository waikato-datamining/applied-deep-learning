The [wai.tfimageclass](https://github.com/waikato-datamining/tensorflow/tree/master/image_classification) Python library
([PyPI](https://pypi.org/project/wai.tfimageclass/)) can be used for training various models that are available from 
[Tensorflow Hub](https://tfhub.dev/).

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
waikatodatamining/tf_image_classification:1.14
```

If you only have a CPU machine available, then use this one instead:

```
waikatodatamining/tf_image_classification:1.14_cpu
```

The training script is called `tfic-retrain`, for which we can invoke the help screen as follows:

```bash
docker run -t waikatodatamining/tf_image_classification:1.14 tfic-retrain --help      # GPU
docker run -t waikatodatamining/tf_image_classification:1.14_cpu tfic-retrain --help  # CPU
```

It is good practice creating a separate sub-directory for each training run, with a directory name that hints at
what dataset and model were used. So for our first training run, which will use mainly default parameters, we will 
create the following directory in the `output` folder:

```
3flowers-tf-default
```

The following command will train an [Inception v3](https://tfhub.dev/google/imagenet/inception_v3/feature_vector/3) 
model for 500 steps:

GPU:

```bash
docker run \
  -u $(id -u):$(id -g) -e USER=$USER \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/tmp/tfhub_modules \
  -t waikatodatamining/tf_image_classification:1.14 \
  tfic-retrain \
  --image_dir /workspace/data/3flowers \
  --output_graph /workspace/output/3flowers-tf-default/graph.pb \
  --output_info /workspace/output/3flowers-tf-default/graph.json \
  --saved_model_dir /workspace/output/3flowers-tf-default/saved_model \
  --image_lists_dir /workspace/output/3flowers-tf-default \
  --tfhub_module https://tfhub.dev/google/imagenet/inception_v3/feature_vector/3 \
  --training_steps 500
```

CPU:

```bash
docker run \
  -u $(id -u):$(id -g) -e USER=$USER \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/tmp/tfhub_modules \
  -t waikatodatamining/tf_image_classification:1.14_cpu \
  tfic-retrain \
  --image_dir /workspace/data/3flowers \
  --output_graph /workspace/output/3flowers-tf-default/graph.pb \
  --output_info /workspace/output/3flowers-tf-default/graph.json \
  --saved_model_dir /workspace/output/3flowers-tf-default/saved_model \
  --image_lists_dir /workspace/output/3flowers-tf-default \
  --tfhub_module https://tfhub.dev/google/imagenet/inception_v3/feature_vector/3 \
  --training_steps 500
```

# Exporting model

Before we can use the model, we need to export it to *Tensorflow lite* or *tflite*:

GPU:

```bash
docker run \
  -u $(id -u):$(id -g) -e USER=$USER \
  --gpus=all \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/tmp/tfhub_modules \
  -t waikatodatamining/tf_image_classification:1.14 \
  tfic-export \
  --saved_model_dir /workspace/output/3flowers-tf-default/saved_model \
  --tflite_model /workspace/output/3flowers-tf-default/graph.tflite
```

CPU:

```bash
docker run \
  -u $(id -u):$(id -g) -e USER=$USER \
  -v `pwd`:/workspace \
  -v `pwd`/cache:/tmp/tfhub_modules \
  -t waikatodatamining/tf_image_classification:1.14_cpu \
  tfic-export \
  --saved_model_dir /workspace/output/3flowers-tf-default/saved_model \
  --tflite_model /workspace/output/3flowers-tf-default/graph.tflite
```


# Predicting

For making predictions for a single image, you can use the script `tfic-labelimage`.

Since we will want to batch predict multiple images, will use the script `tfic-poll` instead: 

GPU:

```bash
docker run \
  -u $(id -u):$(id -g) -e USER=$USER \
  -v `pwd`:/workspace \
  -t waikatodatamining/tf_image_classification:1.14 \
  tfic-poll \
  --graph /workspace/output/3flowers-tf-default/graph.tflite \
  --graph_type tflite \
  --info /workspace/output/3flowers-tf-default/graph.json \
  --in_dir /workspace/predictions/in \
  --out_dir /workspace/predictions/out
```

CPU:

```bash
docker run \
  -u $(id -u):$(id -g) -e USER=$USER \
  -v `pwd`:/workspace \
  -t waikatodatamining/tf_image_classification:1.14_cpu \
  tfic-poll \
  --graph /workspace/output/3flowers-tf-default/graph.tflite \
  --graph_type tflite \
  --info /workspace/output/3flowers-tf-default/graph.json \
  --in_dir /workspace/predictions/in \
  --out_dir /workspace/predictions/out
```

E.g., for the `image_01965.jpg` from the `anthurium` class, we will get a CSV file similar to 
[this one](img/image_01965.csv):

{{ read_csv('docs/image_classification/img/image_01965.csv') }}

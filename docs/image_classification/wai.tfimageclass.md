The [wai.tfimageclass](https://github.com/waikato-datamining/tensorflow/tree/master/image_classification) Python library
([PyPI](https://pypi.org/project/wai.tfimageclass/)) can be used for training various models that are available from 
the [Tensorflow Hub](https://tfhub.dev/).

# Prerequisites
Make sure you have the directory structure created as outlined in the [Prerequisites](../prerequisites.md).


# Data

In this example, we will use the [102 flowers](http://datasets.cms.waikato.ac.nz/ufdl/image_classification/102flowers/)
dataset, which consists of 102 different categories (~ species) of flowers. More precisely, we will download the
dataset with the flowers already split into categories from which will use a subset to speed up the training process.

Download the dataset from the following URL and extract it:

[http://datasets.cms.waikato.ac.nz/ufdl/image_classification/102flowers/102flowers-categories.tgz](http://datasets.cms.waikato.ac.nz/ufdl/image_classification/102flowers/102flowers-categories.tgz)

Once extracted, you can delete all sub-directories apart from:

* alpine_sea_holly
* anthurium
* artichoke

Rename the `categories` directory to `3flowers` and move it into the `data` folder of our directory structure 
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
docker run -t waikatodatamining/tf_image_classification:1.14_cpu tfic-retrain --help
```

It is good practice creating a separate sub-directory for each training run, with a directory name that hints at
what dataset and model were used. So for our first training run, which will use mainly default parameters, we will 
create the following directory in the `output` folder:

```
3flowers-default
```

The following command, issued in the `applied_deep_learning` directory, will map the `applied_deep_learning`
directory (= the current working directory) onto the `/workspace` directory within the docker container, giving
us access to all the sub-directories there, and train our 3flowsers dataset:

```bash
docker run \
  -v `pwd`:/workspace \
  -t waikatodatamining/tf_image_classification:1.14_cpu \
  tfic-retrain \
  --image_dir /workspace/data/3flowers \
  --output_graph /workspace/output/3flowers-default/graph.pb \
  --training_steps 500
```



# Predicting
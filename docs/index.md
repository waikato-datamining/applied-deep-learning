Over the years, the [Applied Machine Learning Group](https://www.data-mining.co.nz) here at the 
University of Waikato has been working on a number of projects that involved applying deep learning 
algorithms to image problems. Setting up deep learning frameworks is always a slow and painstaking process,
getting all the library dependencies right (CUDA, cuDNN, numpy, etc). To speed things up, we have developed 
(and maintain) a number of Python libraries and [Docker](https://www.docker.com/) images that can be used 
for various deep learning tasks. Here we are showing examples on how you can use these ready-to-use images 
on datasets to train your own models and how to make predictions with them.

The use of Docker made it a lot easier and faster to apply algorithms to new datasets. 
If you have not used Docker before, we recommend you to have a look at our introduction called 
[Docker for Data Scientists](https://www.data-mining.co.nz/docker-for-data-scientists/).

The following domains are covered by the examples:

* [Image classification](image_classification/index.md)
* [Object detection](object_detection/index.md)
* [Instance segmentation](instance_segmentation/index.md)
* [Image segmentation](image_segmentation/index.md)


# Redis

To keep things simple, these examples use *file-polling* when making predictions, i.e.,
looking for images in an input directory and outputting predictions (and original 
input images in another directory).

However, this may not be optimal when using SSDs, as they can wear out quickly when
processing large amounts of files on a 24/7 basis. Having model pipelines can acerbate
things even further.

To alleviate wear and tear on the hardware, these frameworks also allow processing images
via a [Redis](https://redis.io/) in-memory database backend. In this case, the models
listen for images being broadcast on a Redis channel, pick them up, make predictions and
then broadcast the predictions on another Redis channel. This allows for the construction
of efficient, low-latency processing pipelines.

When using this approach, the predictions are commonly broadcast in a [JSON](https://www.json.org/) 
format, which can be easily processed in most programming languages.

Redis itself has [clients](https://redis.io/docs/clients/) available in a wide range of programming 
languages as well.

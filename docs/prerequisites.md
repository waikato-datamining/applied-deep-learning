The following prerequisites apply to most data domains:

# Annotations
In order to annotate data (e.g., for object detection, instance segmentation or image segmentation), you should 
download the *ADAMS Annotator* application (you only need [Java 11 or 17 installed](https://adoptium.net/)):

  * [Snapshot](https://adams.cms.waikato.ac.nz/download/snapshot/)


# Format conversions
For turning annotations from one format into another, we will utilize the *wai.annotations* Python library: 

  * [wai.annotations project](https://github.com/waikato-ufdl/wai-annotations)
  * [wai.annotations manual](https://ufdl.cms.waikato.ac.nz/wai-annotations-manual)

Since we will use this library through Docker images, there is no need to install anything.
If you should decide to install it locally, it is recommended to do so in a virtual environment


# Hardware
For building models, a computer with NVIDIA GPU (8GB+) and Linux operating system is recommended. Only 
image classification models can be built in a reasonable amount of time on CPU-only machines.


# Directory structure  
Create the following directory structure for the examples of this tutorial:
    
```
|
+-- applied_deep_learning
    |
    +-- cache
    |   |
    |   +-- torch
    |   |
    |   +-- iopath_cache
    |
    +-- data
    |
    +-- models
    |
    +-- output
    |
    +-- predictions
        |
        +-- in
        |
        +-- out
```

Use this command-line to create the directories:

```bash
mkdir -p \ 
  applied_deep_learning/cache/torch \
  applied_deep_learning/cache/iopath_cache \
  applied_deep_learning/data \
  applied_deep_learning/models \
  applied_deep_learning/output \
  applied_deep_learning/predictions/in \
  applied_deep_learning/predictions/out
```


# Docker notes

To make the Docker commands as easy as possible, they will all get issued from the within
the `applied_deep_learning` directory. In order to get access to the *data*, *output* and 
*predictions* directories, we use the following mapping:

```
-v `pwd`:/workspace
```

This will map the `applied_deep_learning` directory onto the `/workspace` directory within
the container. 

If you have spaces in any of the parent directories, you need to use double quotes around
the part before the `:` (otherwise you will get an error like `docker: invalid reference format.`): 

```
-v "`pwd`":/workspace
```

Since Docker usually runs as `root` within the container, we want to make sure that the user
group of any files that get generated are being owned by the current user. This can be achieved
by adding the following to the Docker command:

```
-u $(id -u):$(id -g) -e USER=$USER
```

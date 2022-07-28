The following prerequisites apply to most data domains:

# Annotations
In order to annotate data (e.g., for object detection, instance segmentation or image segmentation), you should 
download the *ADAMS Annotator* application (you only need [Java 11 installed](https://adoptopenjdk.net/)):

  * [Snapshot](https://adams.cms.waikato.ac.nz/download/snapshot/)
  * [Release 2021.12.0 or later](https://adams.cms.waikato.ac.nz/download/release/)  

# Format conversions
For turning annotations from one format into another, you need to install the *wai.annotations* Python library: 

  * [wai.annotations project](https://github.com/waikato-ufdl/wai-annotations)
  * [wai.annotations manual](https://ufdl.cms.waikato.ac.nz/wai-annotations-manual)

**NB:** it is recommended to install it in a virtual environment to avoid any conflicts with the host system.


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
    |
    +-- data
    |
    +-- output
    |
    +-- predictions
        |
        +-- in
        |
        +-- out
```

# wai.annotations

Within the `applied_deep_learning` directory, we will install [wai.annotations](https://github.com/waikato-ufdl/wai-annotations),
for converting data:

```bash
python3 -m venv waiann
./waiann/bin/pip install --upgrade pip setuptools
./waiann/bin/pip install "numpy<1.23.0"
./waiann/bin/pip install wai.pycocotools
./waiann/bin/pip install wai.annotations
```

---
title: Coqui STT
---

[Coqui STT](https://github.com/coqui-ai/STT) is a deep learning toolkit for Speech-to-Text. 
Custom docker images with additional tools are available from here:

[https://github.com/waikato-datamining/tensorflow/tree/master/coqui/stt](https://github.com/waikato-datamining/tensorflow/tree/master/coqui/stt)


# Prerequisites
Make sure you have the directory structure created as outlined in the [Prerequisites](../prerequisites.md).


# Data

In this example, we will use the [Dutch Common Voice](https://datasets.cms.waikato.ac.nz/ufdl/common-voice/)
dataset.

Download the dataset from the following URL into the *data* directory and extract it:

[https://datasets.cms.waikato.ac.nz/ufdl/data/common-voice/cv-corpus-10.0-2022-07-04-nl.tar.gz](https://datasets.cms.waikato.ac.nz/ufdl/data/common-voice/cv-corpus-10.0-2022-07-04-nl.tar.gz)

Now we have to convert the format from *Common Voice* into *Coqui STT*. We can do this by using the 
[wai.annotations](https://github.com/waikato-ufdl/wai-annotations) library. 

From within the `applied_deep_learning` directory, run the following command to process the
three datasets (train, dev, test):

```bash
for i in train dev test
do
    echo $i
    docker run -u $(id -u):$(id -g) \
      -v `pwd`:/workspace \
      -t waikatoufdl/wai.annotations:latest \
      wai-annotations convert \
          from-common-voice-sp \
             -i "/workspace/data/cv-corpus-10.0-2022-07-04/nl/$i.tsv" \
             --rel-path ./clips \
          clean-transcript \
             --quotes \
          discard-negatives \
          convert-to-wav \
          convert-to-mono \
          resample-audio \
             -s 16000 \
             -t kaiser_fast \
          to-coqui-stt-sp \
             -o "/workspace/data/cv-nl/$i.csv"
done
```

For generating the alphabet specific to this dataset, use the following command:

```bash
docker run \
  -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatodatamining/tf_coqui_stt:1.3.0_cuda11.0 \
  stt_alphabet \
  -i /workspace/data/cv-nl/train.csv \
     /workspace/data/cv-nl/dev.csv \
     /workspace/data/cv-nl/test.csv \
  -o /workspace/data/cv-nl/alphabet.txt
```


# Training

For training, we will use the following docker image:

```
waikatodatamining/tf_coqui_stt:1.3.0_cuda11.0
```

At prediction time, we no longer need a GPU and can use this image:

```
waikatodatamining/tf_coqui_stt:1.3.0_cpu
```

The training script is called `stt_train`, for which we can invoke the help screen as follows:

```bash
docker run -t waikatodatamining/tf_coqui_stt:1.3.0_cuda11.0 stt_train --help 
```

Instead of using config files, we can just tweak parameters via 
[command-line options](https://stt.readthedocs.io/en/stable/TRAINING_FLAGS.html).

However, we still need to download a base model to use for transfer learning:

[https://github.com/coqui-ai/STT/releases/download/v1.3.0/coqui-stt-1.3.0-checkpoint.tar.gz](https://github.com/coqui-ai/STT/releases/download/v1.3.0/coqui-stt-1.3.0-checkpoint.tar.gz) (648MB)

Download it into the `models` directory and decompress it (`tar -xzf coqui-stt-1.3.0-checkpoint.tar.gz`).

It is good practice creating a separate sub-directory for each training run, with a directory name that hints at
what dataset and model were used. So for our first training run, which will use mainly default parameters, we will 
create the following directory in the `output` folder:

```
cv-nl-coqui
```

Kick off transfer learning with the following command:

```bash
docker run \
  -u $(id -u):$(id -g) \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -t waikatodatamining/tf_coqui_stt:1.3.0_cuda11.0 \
  stt_train \
  --alphabet_config_path /workspace/data/cv-nl/alphabet.txt \
  --train_files /workspace/data/cv-nl/train.csv \
  --dev_files /workspace/data/cv-nl/dev.csv \
  --test_files /workspace/data/cv-nl/test.csv \
  --drop_source_layers 1 \
  --n_hidden 2048 \
  --use_allow_growth true \
  --train_cudnn true \
  --train_batch_size 16 \
  --dev_batch_size 16 \
  --export_batch_size 16 \
  --epochs 75 \
  --load_checkpoint_dir /workspace/models/coqui-stt-1.3.0-checkpoint \
  --save_checkpoint_dir /workspace/output/cv-nl-coqui
```


# Evaluating the model

Once the model has been built, we can evaluate it using the `stt_eval`. This script
will use the *test* set and output the best and worst transcripts. You can run 
it like this:

```bash
docker run \
  -u $(id -u):$(id -g) \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -t waikatodatamining/tf_coqui_stt:1.3.0_cuda11.0 \
  stt_eval \
  --alphabet_config_path /workspace/data/cv-nl/alphabet.txt \
  --train_files /workspace/data/cv-nl/train.csv \
  --dev_files /workspace/data/cv-nl/dev.csv \
  --test_files /workspace/data/cv-nl/test.csv \
  --test_batch_size 16 \
  --export_batch_size 16 \
  --checkpoint_dir /workspace/output/cv-nl-coqui
```


# Exporting to tflite

Before we can use our trained model, we will need to export it to TensorFlow lite (aka tflite)
using the `stt_export` script: 

```bash
docker run \
  -u $(id -u):$(id -g) \
  --gpus=all \
  -v `pwd`:/workspace \
  -t waikatodatamining/tf_coqui_stt:1.3.0_cuda11.0 \
  stt_export \
  --export_quantize false \
  --checkpoint_dir /workspace/output/cv-nl-coqui \
  --export_dir /workspace/output/cv-nl-coqui/export
```

This will create a file called `output_graph.tflite` in the `export` sub-directory of the output directory.

**Notes**

* `--export_quantize false` - will create a more accurate but slower model
* `--export_quantize true` - will create a model optimized for speed/memory but less accurate


# Predicting

Using the `stt_transcribe_poll` script, we can batch-process audio files placed in the `predictions/in` directory
as follows (e.g., from our *test* subset): 

```bash
docker run \
  -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatodatamining/tf_coqui_stt:1.3.0_cuda11.0 \
  stt_transcribe_poll \
  --model /workspace/output/cv-nl-coqui/export/output_graph.tflite \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out
```
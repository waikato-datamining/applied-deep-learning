---
title: Coqui STT
---

[Coqui STT](https://github.com/coqui-ai/STT) is a deep learning toolkit for Speech-to-Text. 
Custom docker images with additional tools are available from here:

[https://github.com/waikato-datamining/tensorflow/tree/master/coqui/stt](https://github.com/waikato-datamining/tensorflow/tree/master/coqui/stt)


# Prerequisites
Make sure you have the directory structure created as outlined in the [Prerequisites](../prerequisites.md).


# Data

In this example, we will use the *Irish dataset* of the [Living Audio Datasets](https://datasets.cms.waikato.ac.nz/ufdl/living-audio-datasets/)
collection.

Download the dataset from the following URL into the *data* directory and extract it:

[https://datasets.cms.waikato.ac.nz/ufdl/data/living-audio-datasets/irish-coqui-stt.zip](https://datasets.cms.waikato.ac.nz/ufdl/data/living-audio-datasets/irish-coqui-stt.zip)

Rename the `coqui-stt` directory to `lad-irish`.


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
lad-irish-coqui
```

For generating the alphabet specific to this dataset, use the following command (if necessary, you can 
supply multiple CSV files to the `stt_alphabet` script):

```bash
docker run \
  -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatodatamining/tf_coqui_stt:1.3.0_cuda11.0 \
  stt_alphabet \
  -i /workspace/data/lad-irish/samples.csv \
  -o /workspace/data/lad-irish/alphabet.txt
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
  --alphabet_config_path /workspace/data/lad-irish/alphabet.txt \
  --auto_input_dataset /workspace/data/lad-irish/samples.csv \
  --drop_source_layers 2 \
  --n_hidden 2048 \
  --use_allow_growth true \
  --train_cudnn true \
  --train_batch_size 16 \
  --dev_batch_size 16 \
  --export_batch_size 16 \
  --epochs 75 \
  --load_checkpoint_dir /workspace/models/coqui-stt-1.3.0-checkpoint \
  --save_checkpoint_dir /workspace/output/lad-irish-coqui
```

**Notes:**

* Coqui STT 1.3.0 has a bug and does not exit after training. Once you see 
  `I FINISHED optimization in 0:25:52.838710`, you can safely stop the process using the `kill` command.
* How many layers you drop when training with a new dataset (`--drop_source_layers X`) requires some 
  experimentation, dropping just one does not always work.
* `--auto_input_dataset` will split the single dataset into `train.csv`, `dev.csv` and `test.csv`.
  If you already have these datasets (e.g., obtained from Common Voice), then you can use the 
  `--train_files`, `--dev_files` and `--test_files` options to supply them explicitly.


# Evaluating the model

Once the model has been built, we can evaluate it using `stt_eval`. This script
will use the *test* set and output the best and worst transcripts (using 
[CER](https://torchmetrics.readthedocs.io/en/stable/text/char_error_rate.html) and
[WER](https://torchmetrics.readthedocs.io/en/stable/text/word_error_rate.html) as metrics). 
You can run it like this:

```bash
docker run \
  -u $(id -u):$(id -g) \
  --shm-size 8G \
  --gpus=all \
  -v `pwd`:/workspace \
  -t waikatodatamining/tf_coqui_stt:1.3.0_cuda11.0 \
  stt_eval \
  --alphabet_config_path /workspace/data/lad-irish/alphabet.txt \
  --train_files /workspace/data/lad-irish/train.csv \
  --dev_files /workspace/data/lad-irish/dev.csv \
  --test_files /workspace/data/lad-irish/test.csv \
  --test_batch_size 16 \
  --export_batch_size 16 \
  --checkpoint_dir /workspace/output/lad-irish-coqui
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
  --checkpoint_dir /workspace/output/lad-irish-coqui \
  --export_dir /workspace/output/lad-irish-coqui/export
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
  --model /workspace/output/lad-irish-coqui/export/output_graph.tflite \
  --prediction_in /workspace/predictions/in \
  --prediction_out /workspace/predictions/out
```

If you just want to test with a single audio file, then use `stt_transcribe_single` instead
(point to the audio file with `--audio`):

```bash
docker run \
  -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatodatamining/tf_coqui_stt:1.3.0_cuda11.0 \
  stt_transcribe_single \
  --model /workspace/output/lad-irish-coqui/export/output_graph.tflite \
  --audio /workspace/data/lad-irish/cll_z0001_018.wav
```

This will output something like this:

```
Loading model from file /workspace/output/lad-irish-coqui/export/output_graph.tflite
TensorFlow: v2.8.0-8-g06c8fea58fd
 Coqui STT: v1.3.0-0-g148fa743
INFO: Created TensorFlow Lite XNNPACK delegate for CPU.
Loaded model in 0.0431s.
Running inference.
Iir scoláira béleidis é eidrsan.
Inference took 3.284s for 2.464s audio file.
```

The ground truth for `cll_z0001_018.wav` is as follows:

```
Níor scoláire béaloidis é Pedersen.
```

# Audio

The audio recordings should be sampled with 16kHz, in mono and stored in 
[WAV](https://en.wikipedia.org/wiki/WAV) files.

You can do this using [wai.annotations](https://github.com/waikato-ufdl/wai-annotations).
E.g., the following command converts all WAV files in `INPUT_DIR` and stores them in 
`OUTPUT_DIR`:

```bash
docker run -u $(id -u):$(id -g) \
  -v `pwd`:/workspace \
  -t waikatoufdl/wai.annotations:latest \
  wai-annotations convert \
    from-audio-files-ac \
      -i "INPUT_DIR/*.wav" \
    convert-to-mono \
    resample-audio \
      -s 16000 \
    to-audio-files-ac \
      -o OUTPUT_DIR
```


# Annotations
The simplest file formats for speech annotations are probably:

* Festvox 
* Coqui TTS


## Festvox

The annotations are stored in a text file, each row for a separated recording,
using the following format:

```
( FILENAME "TRANSCRIPT" )
```

* `FILENAME` - the name of the file **without** extension
* `TRANSCRIPT` - the text associated with the recording 

It is best to store the annotations file alongside the WAV files to avoid paths.


## Coqui TTS

In this [format](https://stt.readthedocs.io/en/latest/TRAINING_INTRO.html#data-format), 
the annotations are stored in a CSV file with the following columns:

* `wav_filename` - the name of the file **with** extension
* `wav_filesize` - the size of the WAV file in bytes
* `transcript` - the text associated with the recording

It is best to store the annotations file alongside the WAV files to avoid paths.

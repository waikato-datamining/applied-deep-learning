Annotating data for image classification is straight forward. Since we are classifying whole images, we can
simply use the directory layout for annotating images. In the example below, we have a dataset called *flowers*.
The sub-directories below the *flowers* directory represent the labels for the images containing within the
sub-directories.

```
   |
   +- flowers
      |
      +- daisy
      |
      +- dandelion
      |
      +- roses
      |
      +- sunflowers
      |
      +- tulip
```

**NB:** To avoid any issues, the labels should be lowercase and underscores should be used instead of blanks/spaces. 

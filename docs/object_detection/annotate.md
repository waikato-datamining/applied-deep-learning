You can use the [ADAMS](https://adams.cms.waikato.ac.nz/) framework for annotating your images. 
ADAMS comes pre-bundled in various setups and we need the *adams-annotator* bundle. You can get either:

* [snapshot](https://adams.cms.waikato.ac.nz/download/snapshot/)
* [release](https://adams.cms.waikato.ac.nz/download/release/) (21.12.0 or later)

In the Flow editor (available from the *Tools* menu in ADAMS), you can load and execute the follow workflow
(which is part of your ADAMS installation when downloading it as zip file):

[adams-imaging-annotate_objects.flow](https://github.com/waikato-datamining/adams-base/blob/master/adams-imaging/src/main/flows/adams-imaging-annotate_objects.flow)

When prompted, use *bounding_box* as *Selection type*.

The following video takes you through the process:

![type:video](https://www.youtube.com/embed/BCmPZ0Dbwmg)


Since mid-April 2025, snapshots of ADAMS have the new **SAM2** tool
available from the *Tools* panel. *SAM2* stands for [Segment Anything Model 2](https://github.com/facebookresearch/sam2),
a public model from Meta that can be used for determining outlines of objects,
making the annotation process much more efficient. The model can be used
on CPU-only and GPU machines alike, as well as on Linux and Windows.
The following video demonstrates the tool:

![type:video](https://youtu.be/5ln2c2kgMAc)

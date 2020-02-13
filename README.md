# *Newspaper Navigator*

## By Benjamin Charles Germain Lee (2020 Library of Congress Innovator-in-Residence)

## Introduction

*Newspaper Navigator* is a project to re-imagine searching over <a href="https://chroniclingamerica.loc.gov/">*Chronicling America*</a>. The first stage of *Newspaper Navigator* is to extract content such as photographs, illustrations, cartoons, and news topics from the Chronicling America newspaper scans and corresponding OCR using emerging machine learning techniques. The second stage is to reimagine an exploratory search interface over the collection in order to enable a wide range of people to navigate the collection according to their interests.

**This project is currently under development, and updates to the documentation will be made as the project unfolds throughout the year.**


## What's Implemented So Far
This code base explores using the <a href="http://beyondwords.labs.loc.gov/#/">*Beyond Words*</a> crowdsourced annotations of photographs, illustrations, comics, cartoons, and maps from <a href="https://chroniclingamerica.loc.gov/">*Chronicling America*</a> to finetune a pre-trained object detection model to detect visual content in historical newspaper scans. This finetuned model can then be used as the first step in a pipeline for automating the extraction of visual content from *Chronicling America*, as well as captions and corresponding textual content from the METS/ALTO OCR of each *Chronicling America* page. This code base also contains a script for visualizing the extracted photographs, illustrations, comics, cartoons, and maps from *Chronicling America* using ResNet-18 embeddings and T-SNE.

## Whitepaper

If you'd like to read about this work in depth, you can find a whitepaper describing the progress made so far in <a href="https://github.com/bcglee/beyond_words/tree/master/whitepaper">/whitepaper/</a>. The paper contains a more detailed description of the code, benchmarks, and related work (*as with the rest of this repo, the paper is still a work-in-progress and will be updated regularly*).

## Dataset

The *Beyond Words* dataset for photograph, illustration, comic, cartoon, and map detection in historical newspaper scans, is formatted according to the <a href="http://cocodataset.org/#format-data">COCO</a> standard for object detection and can be found in <a href="https://github.com/bcglee/beyond_words/tree/master/beyond_words_data">/beyond_words_data/</a>. The images are stored in <a href="https://github.com/bcglee/beyond_words/tree/master/beyond_words_data/images">/beyond_words_data/images/</a>, and the JSON can be found in <a href="https://github.com/bcglee/beyond_words/blob/master/beyond_words_data/trainval.json">/beyond_words_data/trainval.json</a>.   The dataset contains 3,437 images with 6,732 verified annotations.  Here is a breakdown of categories:

* Photographs: 4,193
* Illustrations: 1,028
* Maps: 79
* Comics/Cartoons: 1,139
* Editorial Cartoons: 293


For an 80\%-20\% split, see <a href="https://github.com/bcglee/beyond_words/blob/master/beyond_words_data/trainval_80_percent.json">/beyond_words_data/trainval_80_percent.json</a> and <a href="https://github.com/bcglee/beyond_words/blob/master/beyond_words_data/test_80_percent.json">/beyond_words_data/test_80_percent.json</a>.  Lastly, the original annotations from the *Beyond Words* site can be found at <a href="https://github.com/bcglee/beyond_words/blob/master/beyond_words_data/beyond_words.txt">/beyond_words_data/beyond_words.txt</a>.

To construct the dataset using updated annotations, first update the annotations file from the Beyond Words website, then run the script <a href="https://github.com/bcglee/beyond_words/blob/master/process_beyond_words_images_COCO.py">process_beyond_words_images_COCO.py</a>.  This file also contains some commented-out, vestigial code for creating pixel masks to work with the <a href="https://dhsegment.readthedocs.io/en/latest/start/demo.html">dhSegment</a> format.

## Detecting and Extracting Visual Content from Historic Newspaper Scans

With this dataset fully constructed, the next step is to train a deep learning model to identify visual content and classify it according to the *Beyond Words* taxonomy (photograph, illustration, comics/cartoon, editorial cartoon, and map).  The approach that I've taken is to finetune a pre-trained Faster-RCNN impelementation in <a href="https://github.com/facebookresearch/detectron2">Detectron2</a>'s <a href="https://github.com/facebookresearch/detectron2/blob/master/MODEL_ZOO.md">Model Zoo</a> in PyTorch.  

I have included scripts and notebooks designed to run out-of-the-box on an *AWS EC2 instance* with a *Deep Learning Ubuntu AMI*.  First, run <a href="https://github.com/bcglee/beyond_words/blob/master/install-scripts/install_detectron_2.sh">install_detectron_2.sh</a> in order to install Detectron2, as well as all of its dependencies. Due to some deprecated code in pycocotools, it is worth changing "unicode" to "bytes" in line 308 of ~/anaconda3/lib/python3.6/site-packages/pycocotools/coco.py in order to enable the test evaluation in Detectron2 to work correctly. If you are not using an EC2 instance with a Deep Learning Ubuntu AMI, I recommend following the steps on the Detectron2 repo for installation.

Next, you will find the Jupyter notebook <a href="https://github.com/bcglee/beyond_words/blob/master/notebooks/train_model.ipynb">Beyond_Words_Notebook_AWS.ipynb</a>, which contains code for finetuning different pre-trained Faster-RCNN implementations from <a href="https://github.com/facebookresearch/detectron2">Detectron2</a>'s <a href="https://github.com/facebookresearch/detectron2/blob/master/MODEL_ZOO.md">Model Zoo </a> and benchmarking them. This file saves the predicted bounding boxes for each scan as a JSON file. If everything is installed correctly, the notebook should run without any additional steps.

Here are performance metrics on two pretrained Faster-RCNN models from Detectron2's Model Zoo (all metrics reported are on the 20\% validation set in the repo using a single NVIDIA Tesla K80 GPU provided by Google Colab - *note:  I will report these benchmarks using a stable AWS EC2 instance*):

| Model | average precision | inference time per image |
| ----- | ----------------- | ------------------------ |
|faster\_rcnn\_R\_50\_FPN\_3x | 56.1 \% | 0.1 s / img|
| faster\_rcnn\_X\_101\_32x8d\_FPN\_3x | 57.4 \% | 0.25 s / img |

For a slideshow showing the performance of faster\_rcnn\_R\_50\_FPN\_3x on 50 sample pages from the *Beyond Words* test set, please see <a href="https://github.com/bcglee/beyond_words/blob/master/demos/slideshow.mp4">/demos/slideshow.mp4</a>.

## Extracting Captions and Textual Content using METS/ALTO OCR

Now that we have a finetuned model for extracting visual content from newspaper scans in *Chronicling America*, we can leverage the OCR of each scan to weakly supervise captions and corresponding textual content. Because *Beyond Words* volunteers were instructed to draw bounding boxes over corresponding textual content, the finetuned model has learned how to do this as well.  As a baseline approach, we can find all of the textual content that falls within each predicted bounding box. Note that this is precisely what happens in *Beyond Words* during the "Transcribe" step, where volunteers correct the OCR within each bounding box.  

The script <a href="https://github.com/bcglee/beyond_words/blob/master/parse_xml.py">parse_xml.py</a> utilizes the predicted bounding boxes in JSON format to extract textual content within each predicted bounding box from the METS/ALTO OCR XML file for each *Chronicling America* page.  The textual content is then added to the JSON. This script also uses PIL to crop the predicted bounding boxes from the newspaper scans to produce a set of extracted visual content.

## Visualizing a Day in Newspaper History

Using the <a href="https://github.com/bcglee/chronam-get-images">chronam-get-images</a> repo, we can pull down all of the *Chronicling America* content for a specific day in history (or, a larger date range if you're interested - the world is your oyster!).  Running the above scripts, we've gone from a set of scans and OCR XML files to extracted visual content. How do we then visualize this content?

One answer is to use image embeddings and T-SNE to cluster the images in 2D.  To accomplish this, I've used <a href="https://github.com/christiansafka/img2vec">img2vec</a>, a Python package that produces embeddings for images using ResNet-18 and AlexNet.  Here, I've chosen to use the 512-dimensional embeddings from ResNet-18 produced by feeding in an image and grabbing the weights in the last hidden layer.  Using sklearn's implementation of T-SNE, it's easy to perform dimensionality reduction down to 2D - perfect for a visualization. We can then visualize a day in history!

For a sample visualization of June 7th, 1944 (the day after D-Day), please see <a href="https://github.com/bcglee/beyond_words/blob/master/demos/visualizing_6_7_1944.png">/demos/visualizing_6_7_1944.png</a> (*NOTE: the image is 20 MB, enabling high resolution of images even when zooming in*).  If you search around in this visualization, you will find clusters of maps showing the Western Front, photographs of military action, and photographs of people.  Currently, the aspect ratio of the extracted visual content is not preserved, but this is to be added in future iterations.


The script <a href="https://github.com/bcglee/beyond_words/blob/master/generate_visualization.py">generate_visualization.py</a> contains all of my code for generating visualizations.  Note that it requires <a href="https://github.com/christiansafka/img2vec">img2vec</a> and scikit-learn.


(*This README will be updated as new commits are added*).

Automatic Number (License) Plate Recognition
============================================

##### mturk.html:
Defines the web interface that will be used by the MTurk workers to label the images.
Modified from [original](https://github.com/kyamagu/bbox-annotator)
Use this html/js code with Amazon mechanical Turk. Instructions [here](https://blog.mturk.com/tutorial-annotating-images-with-bounding-boxes-using-amazon-mechanical-turk-42ab71e5068a).  
This code needs to be improved to allow image zoom. Without zoom capability it is too difficult to create the boxes for the small characters of the license plate text. 
Consequently you can spend a lot of time re-arranging the boxes in the labelimg utility.

##### genImageListForAWS.py
Use this module to generate a csv file that can be uploaded to MTurk. You will need the csv file when you
publish a batch of images for processing

##### inspectHITs.py:
Once the batch has been completed by the workers you will need to download the results in csv file format,
and approve or reject each HIT. This application will read the HIT results and overlay the bounding boxes
and labels onto the images. A text box is provided for accepting or rejecting each HIT. Once complete, your
accept/reject response will be added to the downloaded csv file, and the new csv file can be uploaded to MTurk
 
##### csvToPascalXml.py:
Reads the csv file generated by inspectHITs.py, and generates PASCAL VOC style xml annotation files. 
One xml file for each image.

##### labelImg
Once the annotations are in Pascal VOC style xml, you can use [labelImg](https://tzutalin.github.io/labelImg/) to fix any mistakes in the labelled images.  
Make sure that you take one pass through your dataset, and click on verify image.

##### build_anpr_records.py:
Reads a group of PASCAL VOC style xml annotation files, and combines with associated images 
to build a TFrecord dataset. Requires a predefined label map file that maps labels to integers.  
Will only use images where the corresponding annotation file has the verified field set to 'yes'.  
````
python build_anpr_records.py --image_dir=images --record_dir=datasets/records \  
--annotations_dir=images --label_map_file=datasets/records/classes.pbtxt --view_mode=False
````
##### Directory layout
It is important to spend some time figuring out the best directory layout.   
Images and annotations are grouped by camera and date of image capture
Annotations are stored in a directory with the same name as the image director, but suffixed with '_nn'
If you are extracting images from video clips, then these can be stored in the video directory.
````
images  
└── SJ7STAR  
    ├── images  
    │   ├── 2018_02_24_11-00  
    │   ├── 2018_02_24_11-00_ann  
    │   ├── 2018_02_24_15-00  
    │   ├── 2018_02_24_9-00  
    │   ├── 2018_02_24_9-00_ann  
    │   ├── 2018_02_26  
    │   └── 2018_02_27  
    └── video
        └── 2018_02_27
````
Datasets are grouped by experiment name and date of run.
There is a separate directory, records, which contains the tfrecord files, and the classes.pbtxt   
You will need to create the evaluation, exported_model and training directories. 
faster_rcnn_resnet101_coco_2018_01_28 is created when you unzip the pre-trained base model.
The other sub-directories are created when you run object_detection/train.py  
'tensorflow_object_detection_datasets' is linked to 'tensorflow/models/research/object_detection/anpr'
````
tensorflow_object_detection_datasets  
├── experiment_faster_rcnn  
│   ├── 2018_05_28  
│   │   ├── evaluation  
│   │   ├── exported_model  
│   │   │   └── saved_model  
│   │   │       └── variables  
│   │   └── training  
│   │       └── faster_rcnn_resnet101_coco_2018_01_28  
│   │           └── saved_model  
│   │               └── variables  
│   └── 2018_06_01  
├── experiment_ssd  
└── records
````

##### Train the object_detection model
Now you can use tensorflow/models/research/object_detection to train the model
It goes something like this. Assuming python virtualenv called tensorflow, 
a single GPU for training and CPU for eval:

cd tensorflow/models/research/object_detection

###### Training
````
workon tensoflow  
python train.py --logtostderr \  
--pipeline_config_path ../anpr/experiment_faster_rcnn/2018_06_12/training/faster_rcnn_anpr.config \  
--train_dir ../anpr/experiment_faster_rcnn/2018_06_12/training
````
###### Eval
If you are running the eval on CPU, then limit the number of images to evaluate by modifing your config file:  
````
130 eval_config: {  
131 num_examples: 5  
````
New terminal 
```` 
workon tensoflow  
export CUDA_VISIBLE_DEVICES=""  
python eval.py --logtostderr \  
--checkpoint_dir ../anpr/experiment_faster_rcnn/2018_06_12/training \  
--pipeline_config_path ../anpr/experiment_faster_rcnn/2018_06_12/training/faster_rcnn_anpr.config \  
--eval_dir ../anpr/experiment_faster_rcnn/2018_06_12/evaluation
````
New terminal
````
cd tensorflow/models/research  
workon tensoflow  
tensorboard --logdir anpr/experiment_faster_rcnn  
````
###### Export model
````
cd tensorflow/models/research/object_detection
workon tensorflow  
python export_inference_graph.py --input_type image_tensor \  
--pipeline_config_path ../anpr/experiment_faster_rcnn/2018_06_12/training/faster_rcnn_anpr.config \  
--trained_checkpoint_prefix ../anpr/experiment_faster_rcnn/2018_06_12/training/model.ckpt-60296 \  
--output_directory ../anpr/experiment_faster_rcnn/2018_06_12/exported_model
````
###### predict.py
Back to this project directory to run predict.py
Test your exported model against an image dataset.
Prints the detected plate text, and displays the annotated image.
````
workon tensorflow  
python predict.py --model datasets/experiment_faster_rcnn/2018_06_12/exported_model/frozen_inference_graph.pb \  
--labels datasets/records/classes.pbtxt --imagePath images/SJ7STAR_images/2018_05_27 \  
--num-classes 37
````
##### predict_video.py
Test your exported model against a video dataset.
Outputs an annotated video and a series of still images. The still images are grouped to reduce 
the output of images with duplicate plates.
````
python predict_video.py --conf conf/lplates_smallset.json
````
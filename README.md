# Sub-cortical brain structure segmentation of MRI

Subcortical brain structure segmentation of MRI images using a combination of convolutional and a-priori spatial feautures. Related paper of the method is available here: [https://www.medicalimageanalysisjournal.com/article/S1361-8415(18)30383-9/abstract](https://www.medicalimageanalysisjournal.com/article/S1361-8415(18)30383-9/abstract)

## Overview: 

This repository implements a voxelwise convolutional neural network based approach for accurate segmentation of the sub-cortical brain structures. In order to increase the sensitivity of the method on tissue boundaries and voxels with poor contrast or lack of contrast, we fuse a-priori class probabilities with convolutional features in the first fully convolutional layers of the net.

![pipeline](/imgs/pipeline.png)	


Contrary to other available works, the proposed network is trained using a restricted sample selection to force the network to learn only the most challenging voxels from structure boundaries. For more information about the method of the evaluation performed, please see the related publication. 

## Coming soon:
- Supplementary data for the related publications
- Updated documentation for the latest features (transfer learning, GUI, etc.)

# Install:
Keras and Theano implementations of the CNN architecture are available. Installing requirements as shown below should be enough to be able to run the method using either CNN implementations.

If the method is run with the Theano implementation using GPU, please be sure that the Theano ```cuda*``` backend has been installed [correctly](https://github.com/Theano/Theano/wiki/Converting-to-the-new-gpu-back-end%28gpuarray%29). In the case of CPU, be sure that the fast linear algebra [libraries](http://lasagne.readthedocs.io/en/latest/user/installation.html#numpy-scipy-blas) are also installed. 

Once these requirements are met, the rest of python libraries may be easily installed using ```pip```: 

```python
pip install -r requirements.txt 
```


# How to use it:
The method could be run using GUI or from a command line. Example usages are shown below for both cases.

## GUI
### Training from scratch
![network training](/imgs/full_training_example.gif)

### Transfer learning
![transfer learnign](/imgs/transfer_learning_example.gif)

### Testing
![network testing](/imgs/testing_example.gif)

## Command line

This subcortical brain tissue segmentation method only relies on T1-w images. All method parameters and options are easily tuned from the `configuration.cfg` file. Dataset options are selected as follows, including `train_folder`, `inference_folder` and image names:  

```python
[database]
train_folder = /path/to/training_dataset/ 
inference_folder = /path/to/images_to_infer_seg/
t1_name = t1_name #ex. t1.nii, t1.nii.gz
roi_name = gt_15_classes.nii.gz
output_name = segmentation_result.nii.gz
```

Unless the model `name` and the training mode `mode`, the rest of parameters work well in most of the situations with default parameters.  

```python
[model]
name = model_name (to reuse aftwerwards)
mode = cuda*  (select GPU number cuda0, cuda1 or cpu) 
patch_size = 32 
batch_size = 256 
patience = 20 (i.e maximum number of iterations of early-stopping)
net_verbose = 1 (show messages (1) or not (0))
max_epochs = 100 (number of epochs to train)
train_split = 0.25 (number of training samples used for validation)
test_batch_size = 2000 (number of samples for each testing batch)
load_weights = True (permit the model to load the content of trained model)
out_probabilities = False (Compute probability maps for each output class)
speedup_segmentation = True (Reduce the segmentation only to the subcortical space)
post_process = True (Postprocess the segmentation excluding sporious regions)
debug = True (Show messsages that can be useful for degugging the model)
```

### Training: 

For training,  manual sub-cortical labels have to be provided along with T1-w images. When training, the method expects the data to be stored as:

```
[training_folder]
	     [image_1_folder]
			    t1_image
				manual_annotation (roi)
	     [image_2_folder]
			    t1_image
				manual_annotation (roi)
	     
		 ....
		 
		 
	     [image_n_folder]
			    t1_image
				manual_annotation (roi)

```

#### Example: 

Load the configuration file from the `configuration.cfg` file:

```python

import os, sys, ConfigParser
import nibabel as nib
from cnn_cort.load_options import *


user_config = ConfigParser.RawConfigParser()
user_config.read(os.path.join(os.getcwd(), 'configuration.cfg'))
options = load_options(user_config)

```
Then, access data according to the selected `train_folder` building a set training patches to be used for training: 

```python
from cnn_cort.base import load_data, generate_training_set,


# get data patches from all orthogonal views 
x_axial, x_cor, x_sag, y, x_atlas, names = load_data(options)

# build the training dataset
x_train_axial, x_train_cor, x_train_sag, x_train_atlas, y_train = generate_training_set(x_axial,
                                                                                        x_cor,
                                                                                        x_sag,
                                                                                        x_atlas,
                                                                                        y,
                                                                                        options)
```

Once data patches are extracted, the network model can be compiled. The best network weights are stored in `weights_path`. These weights can be then accessed for future inference. For a more concise description of the network structure and details please visit the related publication.

Keras ```main.py```
```python
from cnn_cort.keras_net import get_callbacks, get_model
TRANSFER_LEARNING = True  # Apply transfer learning
FEATURE_WEIGHTS_FILE = '/path/to/pretrained/model.h5' # or set it to None if no transfer learning is needed

weights_save_path = os.path.join(CURRENT_PATH, 'nets', options['experiment'])

model = get_model(dropout_rate=0.3, transfer_learning=TRANSFER_LEARNING, feature_weights_file=FEATURE_WEIGHTS_FILE)
model.compile(optimizer='adam', loss=['categorical_crossentropy'], metrics=['accuracy'])
model.fit([x_train_axial, x_train_cor, x_train_sag, x_train_atlas],
	  [np_utils.to_categorical(y_train, num_classes=15)],
	  [validation_split=options['train_split'],
	  epochs=options['max_epochs'],
	  batch_size=options['batch_size'],
	  verbose=options['net_verbose'],
	  shuffle=True,
	  callbacks=get_callbacks(options, weights_save_path))
model.save(os.path.join(weights_save_path, 'model.h5'))
```



Theano ```train_theano_model.py```
```python

from cnn_cort.nets import build_model
weights_path = os.path.join(os.getcwd(), 'nets')
net = build_model(weights_path, options)

# train the net
net.fit({'in1': x_train_axial,
         'in2': x_train_cor,
         'in3': x_train_sag,
         'in4': x_train_atlas}, y_train)

```

### Testing:

Once a trained model exist, this can be used to infer sub-cortical classes on other images of the same image domain. The next example shows a simple script for batch-processing. We assume here that testing images follow also the same folder structure seen for training: 

```
[testing_folder]
	     [image_1_folder]
			    t1_image

	     [image_2_folder]
			    t1_image	     
		 ....
		 
	     [image_n_folder]
			    t1_image
```


#### Example: 

First, we load the paths of all the images to infer: 

```python 
from cnn_cort.base import load_test_names, test_scan 

# get the testing image paths
t1_test_paths, folder_names  = load_test_names(options)
```

Then, the trained network model set in the configuration as (`name`) is loaded with the best trained weights. We assume that the `load_weights` option is set to `True` in the `configuration.cfg` file:

Theano ```train_theano_model.py```
```python
from cnn_cort.nets import build_model

weights_path = os.path.join(os.getcwd(), 'nets')
net = build_model(weights_path, options)
```
Finally, for each image to test, we call the `test_scan` function that will save the final segmentation on the same folder than the input image: 

```python
# iterate through all test scans
for t1, current_scan in zip(t1_test_paths, folder_names):
    t = test_scan(net, t1, options)
    print "    -->  tested subject :", current_scan, "(elapsed time:", t, "min.)"
```

Keras ```main.py```
```python
from cnn_cort.base import testing
from cnn_cort.keras_net import get_model

model = get_model(0.0, False, os.path.join(weights_save_path, 'model.h5'))
testing(options, model)

```

The next figure depicts an output example, where the first image shows the output of the proposed CNN method and the second figure the manual annotation of subcortical structures also with the selected boundary voxels used as negatives (red): 

![pipeline](/imgs/example.png)	


# Limitations:

+ The current method would only work on GNU/Linux 64bit systems. We deal with image registration between spatial atlases and T1-w inside the training routines. So far, image registration is done using included binaries for [NiftiReg](http://niftyreg.sourceforge.net/) which have been compiled for GNU/Linux 64 bits. _We are working to move the method to Docker containers._

# Citing this work:

Please cite this work as:

```
@article{kushibar2018automated,
  title={Automated sub-cortical brain structure segmentation combining spatial and deep convolutional features},
  author={Kushibar, Kaisar and Valverde, Sergi and Gonz{\'a}lez-Vill{\`a}, Sandra and Bernal, Jose and Cabezas, Mariano and Oliver, Arnau and Llad{\'o}, Xavier},
  journal={Medical image analysis},
  volume={48},
  pages={177--186},
  year={2018},
  publisher={Elsevier}
} 
```

 
# License:

License for this software software: BSD 3-Clause License. A copy of this license is present in the root directory.

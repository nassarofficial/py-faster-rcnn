# Train Py-Faster-RCNN on Another Dataset

This tutorial is a fine-tuned clone of [zeyuanxy's one](https://github.com/zeyuanxy/fast-rcnn/tree/master/help/train) for the [py-faster-rcnn](https://github.com/rbgirshick/py-faster-rcnn) code and [Deboc's](https://github.com/debocs/py-faster-rcnn).

We will illustrate how to train Py-Faster-RCNN on another dataset, in this case Pascal Voc 2012 in the following steps we will only provide you with a sample for you to follow, please refer to the original tutorials above for full datasets.

## Clone py-faster-rcnn repository
The current tutorial need you to have clone and tested the regular py-faster-rcnn repository from rbgirshick.
```sh
$ git clone --recursive https://github.com/rbgirshick/py-faster-rcnn.git
```
We will refer to the root directory with $PY_FASTER_RCNN.

You will also need to follow the installation steps from [the original py-faster-rcnn readme](https://github.com/rbgirshick/py-faster-rcnn/blob/master/README.md)

## Build the train-set

### Format the Dataset

But we will use this common architecture for every dataset in $PY_FASTER_RCNN/data, I have added a sample dataset, and used  [labelimg](https://github.com/tzutalin/labelImg). We will be using 3 images with a different class each, 'cat', 'dog', 'person'.
```
py-faster-rcnn/Example_Dataset/
|-- data/
    |-- Annotations/
         |-- *.xml (Annotation files)
    |-- Images/
         |-- *.png (Image files)
    |-- ImageSets/
         |-- train.txt
```

Now we need to write `train.txt` that contains all the names(without extensions) of images files that will be used for training.
Basically with the following:
```sh
$ cd $PY_FASTER_RCNN/data/Example_Dataset/data/
$ mkdir ImageSets
$ ls Annotations/ -m | sed s/\\s/\\n/g | sed s/.txt//g | sed s/,//g > ImageSets/train.txt
```

### Add lib/datasets/yourdatabase.py
You need to add a new python file describing the dataset we will use to the directory `$PY_FASTER_RCNN/lib/datasets`, see [example.py](https://github.com/deboc/py-faster-rcnn/blob/master/lib/datasets/inria.py). Then the following steps should be taken.
  - Modify `self._classes` in the constructor function to fit your dataset, leave '__background__', and add your own. In our example we added 'cat','dog','person'.
  - Be careful with the extensions of your image files. See `image_path_from_index` in `example.py`.
  - Write the function for parsing annotations. See `_load_example_annotation` in `example.py`. We modified it to be able to use Pascal Voc 2012, so if you labeled using the tool specified in the "Format the Dataset" step, and placed them in the right place, you do not need to worry.
  - Do not forget to add `import` syntaxes in your own python file and other python files in the same directory. 

### Update lib/datasets/factory.py

Then you should modify the [factory.py](https://github.com/deboc/py-faster-rcnn/blob/master/lib/datasets/factory.py) in the same directory. For example, to add **Example Dataset**, we should add

```py
example_dataset_path = 'data/Example_Dataset'
for split in ['train', 'test']:
    name = '{}_{}'.format('example', split)
    __sets[name] = (lambda split=split: example(split, example_dataset_path))
```
**NB** : $PY_FASTER_RCNN must be replaced by its actual value !

## Adapt the network model

For example, if you want to use the model **VGG_CNN_M_1024** with alternated optimizations, then you should adapt the solvers in `$PY_FASTER_RCNN/models/VGG_CNN_M_1024/faster_rcnn_alt_opt/`

```sh
$ cd $PY_FASTER_RCNN/models/
$ mkdir Example/
$ cp -r pascal_voc/VGG_CNN_M_1024/faster_rcnn_alt_opt/ Example_Dataset/
```

It mainly concerns with the number of classes you want to train. Let's assume that the number of classes is C (do not forget to count the `background` class). Then you should 
  - Modify `num_classes` to `C`;
  - Modify `num_output` in the `cls_score` layer to `C`
  - Modify `num_output` in the `bbox_pred` layer to `4 * C`

Basically we are classifying 3 things (Person, Cat, Dog and Background) so C=4 and you have:

    - 7 lines to be modified from 21 to 4
    
    - 3 lines to be modified from 84 to 16
    
```sh
$ grep 21 VGG_CNN_M_1024/faster_rcnn_alt_opt/*.pt
   INRIA_Person/faster_rcnn_alt_opt/faster_rcnn_test.pt:    num_output: 21
   INRIA_Person/faster_rcnn_alt_opt/stage1_fast_rcnn_train.pt:    param_str: "'num_classes': 21"
   INRIA_Person/faster_rcnn_alt_opt/stage1_fast_rcnn_train.pt:    num_output: 21
   INRIA_Person/faster_rcnn_alt_opt/stage1_rpn_train.pt:    param_str: "'num_classes': 21"
   INRIA_Person/faster_rcnn_alt_opt/stage2_fast_rcnn_train.pt:    param_str: "'num_classes': 21"
   INRIA_Person/faster_rcnn_alt_opt/stage2_fast_rcnn_train.pt:    num_output: 21
   INRIA_Person/faster_rcnn_alt_opt/stage2_rpn_train.pt:    param_str: "'num_classes': 21"
$ grep 84 VGG_CNN_M_1024/faster_rcnn_alt_opt/*.pt
   INRIA_Person/faster_rcnn_alt_opt/faster_rcnn_test.pt:    num_output: 84
   INRIA_Person/faster_rcnn_alt_opt/stage1_fast_rcnn_train.pt:    num_output: 84
   INRIA_Person/faster_rcnn_alt_opt/stage2_fast_rcnn_train.pt:    num_output: 84
```

## Build config file

The $PY_FASTER_RCNN/models folder must be specified by a config file as in [faster_rcnn_alt_opt.yml](https://github.com/deboc/py-faster-rcnn/blob/master/help/faster_rcnn_alt_opt.yml)
```sh
$ echo 'MODELS_DIR: "$PY_FASTER_RCNN/models"' >> config.yml
```
**NB** : $PY_FASTER_RCNN must be replaced by its actual value !

## Launch the training

In the directory $PY_FASTER_RCNN, run the following command in the shell.

```sh
$ ./tools/train_faster_rcnn_alt_opt.py --gpu 0 --net_name INRIA_Person --weights data/imagenet_models/VGG_CNN_M_1024.v2.caffemodel --imdb inria_train --cfg config.yml
```

Where:    
>--net_name is the folder name in $PY_FASTER_RCNN/models    
>    (nb: the train_faster_rcnn_alt_opt.py script will automatically look into the /faster_rcnn_alt_opt/ subfolder for the .pt files)    
>--weights is the optional location of pretrained weights in .caffemodel    
>--imdb is the full name of the database as specified in the lib/datasets/factory.py file    
>    (nb: dont forget to add the test/train suffix !)    

Or the following connection-proof version if you're afraid of ctrl+C or using your hardware remotely like me :)
```sh
$ nohup ./tools/train_faster_rcnn_alt_opt.py --gpu 0 --net_name INRIA_Person --weights data/imagenet_models/VGG_CNN_M_1024.v2.caffemodel --imdb inria_train --cfg config.yml &
$ tail nohup.out
```
(Just needs you to launch a "$ killall python" to interrupt training. Yes...)

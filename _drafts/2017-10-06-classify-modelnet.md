---
layout: post
title: Classifying 3D Shapes using Keras on FloydHub
---

In this tutorial, we'll train a 3D convolutional neural network (CNN) to classify CAD models of real-world objects.

<!-- The reference code/data for this tutorial can be found [on GitHub](https://github.com/guoguo12/modelnet-cnn).] -->

<!-- I'll assume basic familiarity with ML and neural nets.
In particular, you should understand how regular 2D CNNs work - I recommend Stanford's [CS231n notes](https://cs231n.github.io/convolutional-networks/). -->

We'll be implementing our neural network using [Keras](keras.io).
If you've never used it before, read ["30 seconds to Keras"](https://keras.io/#getting-started-30-seconds-to-keras) first to get a feel for the syntax and style of the API.

When we finish our Keras code, we'll run it on FloydHub, an up-and-coming platform for running deep learning experiments in the cloud.

But first, let's meet the dataset.
<!-- [right float box? - Surprise, it's a peephole connection! If you're *only* interested in FloydHub, download [the reference code and data](TODO) and skip to the "Training on FloydHub" section below.] -->

## About the data

We'll be using [ModelNet10](http://modelnet.cs.princeton.edu/), from [Princeton's Vision and Robotics Group](http://3dvision.princeton.edu/). Modelnet10 contains 4,899 CAD models across 10 categories of common household objects: bathtubs, dressers, toilets, etc. Each example is given as a [OFF file](https://en.wikipedia.org/wiki/OFF_(file_format)), but the standard approach is to discretize each CAD model into a 30 x 30 x 30 grid of *[voxels](https://en.wikipedia.org/wiki/Voxel)*, or volumetric pixels&mdash;think Minecraft.

The data is already split into a training set (of size 3,991) and a test set (of size 908).

The ModelNet website showcases a leaderboard of current best results on ModelNet10. We'll be using the neural network architecture presented in [Paper #14](https://arxiv.org/abs/1612.04774) by Xu and Todorovic. Their reported result: 88% classification accuracy.

We'll first need to download the data and voxelize it. If you'd like to skip to training the neural net, you can download the data [here](https://github.com/guoguo12/modelnet-cnn/tree/master/data) (2 MB).

## Data prep

The Xu and Todorovic paper describes the exact format of the ModelNet10 data:

<blockquote>
Each shape is represented as a set of binary indicators corresponding to 3D voxels of a uniform 3D grid centered on the shape. The indicators take value 1 if the corresponding 3D voxels are occupied by the 3D shape; and 0, otherwise. Hence, each 3D shape is represented by a binary three-dimensional tensor. The grid size is set to 30 × 30 × 30 voxels. The shape size is normalized such that a cube of 24 × 24 × 24 voxels fully contains the shape, and the remaining empty voxels serve for padding in all directions around the shape.
</blockquote>

### Download

We'll start by grabbing the data:

```bash
# About 550 MB compressed, 2.2 GB uncompressed
wget http://3dshapenets.cs.princeton.edu/ModelNet10.zip
unzip ModelNet10.zip
```

Take a look around! The directory structure should be self-explanatory.

### Voxelize

Next, we'll use [binvox](www.patrickmin.com/binvox/) to voxelize the CAD models. Download the binvox executable for your OS and put it somewhere in your PATH. Then:

```bash
for f in ModelNet10/*/*/*.off; do binvox -d 24 -cb $f; done
```

(Note: If you're running binvox on a headless server, you may need to fake a display buffer using `Xvfb`. See the binvox website for details.)

The `-d 24` tells binvox to output a 24 x 24 x 24 voxel grid. The `-cb` tells it to center the shape in the grid.

Give this command some time to run. You will likely see pop-up windows flicker into and out of existence, rendering your computer unusable. Go take a walk or something.

When the command finishes, you should have 4,899 `.binvox` files:

```bash
$ find ModelNet10/ -type f -name '*.binvox' | wc -l
4899
```

### Aggregate

Our last step is to aggregate the 4,899 data files into a single NumPy archive, for ease of access. We'll also pad the data using [`np.pad`](https://docs.scipy.org/doc/numpy/reference/generated/numpy.pad.html) so that each example is 30 x 30 x 30.

We'll use [binvox_rw](https://github.com/dimatura/binvox-rw-py) to read the `.binvox` files as NumPy arrays. Download [`binvox_rw.py`](https://raw.githubusercontent.com/dimatura/binvox-rw-py/public/binvox_rw.py). Then, in a Python file:

```python
import os

import numpy as np

import binvox_rw

ROOT = 'ModelNet10'
CLASSES = ['bathtub', 'bed', 'chair', 'desk', 'dresser',
           'monitor', 'night_stand', 'sofa', 'table', 'toilet']

# We'll put the data into these arrays
X = {'train': [], 'test': []}
y = {'train': [], 'test': []}

# Iterate over the classes and train/test directories
for label, cl in enumerate(CLASSES):
    for split in ['train', 'test']:
        examples_dir = os.path.join('.', ROOT, cl, split)
        for example in os.listdir(examples_dir):
            if 'binvox' in example:  # Ignore OFF files
                with open(os.path.join(examples_dir, example), 'rb') as file:
                    data = np.int32(binvox_rw.read_as_3d_array(file).data)
                    padded_data = np.pad(data, 3, 'constant')
                    X[split].append(padded_data)
                    y[split].append(label)

# Save to a NumPy archive called "modelnet10.npz"
np.savez_compressed('modelnet10.npz',
                    X_train=X['train'],
                    X_test=X['test'],
                    y_train=y['train'],
                    y_test=y['test'])
```

Phew! Don't worry, that was the most tedious part of this tutorial. Now we can get on with the interesting part, namely...

## Graph construction and training

In our training script, we begin by reading and shuffling the data:

```python
import numpy as np
from sklearn.utils import shuffle

data = np.load('modelnet10.npz')
X, Y = shuffle(data['X_train'], data['y_train'])
X_test, Y_test = shuffle(data['X_test'], data['y_test'])
```

We'll be doing 10-way classification, so our neural network will end with a 10-way softmax. So we need to [one-hot encode](https://www.quora.com/What-is-one-hot-encoding-and-when-is-it-used-in-data-science) our targets, which are currently integers between 0 and 9. Keras comes with a nice one-liner for doing this:

```python
import keras

Y = keras.utils.to_categorical(Y, 10)
```

The Xu-Todorovic architecture is

<blockquote>
Our best found model consists of three convolutional layers and one fully-connected layer. Our first layer has 16 filters of size 6 and stride2 <i>[sic]</i>; the second layer has 64 filters of size 5 and stride 2; the third layer has 64 filters of size 5 and stride 2; the last fully-connected layer has <i>C</i> hidden units [where <i>C</i> is the number of classes].
</blockquote>

which translates into

```python
from keras.models import Sequential
from keras.layers import Dense, Flatten, Reshape
from keras.layers.convolutional import Conv3D

model = Sequential()
model.add(Reshape((30, 30, 30, 1), input_shape=(30, 30, 30)))
model.add(Conv3D(16, 6, strides=2, activation='relu'))
model.add(Conv3D(64, 5, strides=2, activation='relu'))
model.add(Conv3D(64, 5, strides=2, activation='relu'))
model.add(Flatten())
model.add(Dense(10, activation='softmax'))
```

Then we compile our graph and start training:

```python
model.compile(loss='categorical_crossentropy',
              optimizer=keras.optimizers.Adam(lr=0.001),
              metrics=['accuracy'])
model.fit(X, Y, batch_size=256, epochs=30, verbose=2,
          validation_split=0.2, shuffle=True)
```

To evaluate our model on the test set, we compute the most likely class for each example using the softmax outputs:

```python
from sklearn.metrics import accuracy_score

Y_test_pred = np.argmax(model.predict(X_test), axis=1)
print('Test accuracy: {:.3f}'.format(accuracy_score(Y_test, Y_test_pred)))
```

Because the classes are inbalanced, the standard accuracy metric for ModelNet10 is average per-class accuracy. We can compute this from the [confusion matrix](https://en.wikipedia.org/wiki/Confusion_matrix):

```python
from sklearn.metrics import confusion_matrix

conf = confusion_matrix(Y_test, Y_test_pred)
avg_per_class_acc = np.mean(np.diagonal(conf) / np.sum(conf, axis=1))

print('Confusion matrix:\n{}'.format(conf))
print('Average per-class accuracy: {:.3f}'.format(avg_per_class_acc))
```

The complete code is [here](https://github.com/guoguo12/modelnet-cnn/blob/master/src/main.py). Try it out! If you're lucky enough to have access to GPUs, you should be able to run this code without any hassle. You should get around 86% average per-class accuracy.

If you're stuck with CPUs, the code will run slowly or not at all. Fear not, we have an alternative, namely...

## Training on FloydHub

FloydHub is YC-backed startup specializing in ML infrastructure and deployment. A self-proclaimed "Heroku for deep learning", FloydHub lets you launch jobs using a slick command-line interface and monitor your jobs using an equally slick web UI. They also have a generous introductory promo: 100 hours of GPU training for free.

To start, follow the first three steps [here](http://docs.floydhub.com/home/getting_started/) to install the Floyd CLI, create an account, and log into the CLI.

Next, we'll upload the dataset to FloydHub. Move the `modelnet10.npz` file into its own directory. (If you're using the [reference code](https://github.com/guoguo12/modelnet-cnn/), this has already been done for you: you'll find the file under `data/`.) Then `cd` into the directory and run the following:

```bash
floyd data init modelnet
floyd data upload
```

Run `floyd data status` to check that the dataset was uploaded successfully. Note the ID of the dataset.

Next, move the training script (`main.py`) into its own directory (`src/`). Then `cd` into the directory and run

```bash
floyd init classify-modelnet
floyd run --env keras --gpu --data [dataset-id] "python main.py /input/modelnet10.npz /output"
```

Be sure to replace `[dataset-id]` with the ID of ModelNet dataset. It should look something like `eo4LZmQVs9JMwW6xB8hta5`. (You can't just use the short name, `modelnet`.)

Your job should now be either queued or running. After it starts running, you can view the output using `floyd logs -t [job-id]`. You can also monitor the job at the [FloydHub website](https://www.floydhub.com), although you won't be able to see the full output.

At the end of the output, you should see the confusion matrix and average per-class accuracy for the test set:

```
2017-05-26 05:27:18,497 INFO - 
2017-05-26 05:27:19,199 INFO - Test accuracy: 0.879
2017-05-26 05:27:19,200 INFO - Confusion matrix:
2017-05-26 05:27:19,200 INFO - [[36  7  0  0  0  0  0  5  2  0]
2017-05-26 05:27:19,201 INFO - [ 1 99  0  0  0  0  0  0  0  0]
2017-05-26 05:27:19,201 INFO - [ 0  3 96  0  0  0  0  0  1  0]
2017-05-26 05:27:19,202 INFO - [ 2  0  1 60  3  1  4  4 11  0]
2017-05-26 05:27:19,202 INFO - [ 0  1  1  0 78  2  4  0  0  0]
2017-05-26 05:27:19,203 INFO - [ 0  0  2  1  1 95  1  0  0  0]
2017-05-26 05:27:19,203 INFO - [ 0  1  0  1 22  0 56  0  5  1]
2017-05-26 05:27:19,204 INFO - [ 0  0  0  0  1  0  1 98  0  0]
2017-05-26 05:27:19,204 INFO - [ 0  1  0 16  0  0  1  0 82  0]
2017-05-26 05:27:19,205 INFO - [ 0  0  2  0  0  0  0  0  0 98]]
2017-05-26 05:27:19,205 INFO - Average per-class accuracy: 0.866
2017-05-26 05:27:19,547 INFO -
```

If your job seems to be stuck in the queue, try killing the job with `floyd stop` and then rerunning it.

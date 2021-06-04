# Video Classification with a CNN-RNN Architecture

**Author:** [Sayak Paul](https://twitter.com/RisingSayak)<br>
**Date created:** 2021/05/28<br>
**Last modified:** 2021/06/03<br>
**Description:** Training a video classifier with transfer learning and a recurrent model on the UCF101 dataset.


<img class="k-inline-icon" src="https://colab.research.google.com/img/colab_favicon.ico"/> [**View in Colab**](https://colab.research.google.com/github/keras-team/keras-io/blob/master/examples/vision/ipynb/video_classification.ipynb)  <span class="k-dot">•</span><img class="k-inline-icon" src="https://github.com/favicon.ico"/> [**GitHub source**](https://github.com/keras-team/keras-io/blob/master/examples/vision/video_classification.py)



This example demonstrates video classification. It is an important use-case with
applications in surveillance, security, and so on. We will be using the
[UCF101 dataset](https://www.crcv.ucf.edu/data/UCF101.php) to build our video classifier.
The dataset consists of videos categorized into different actions like cricket shot,
punching, biking, etc. This is why the dataset is known to build action recognizers which
is just an extension of video classification.

A video is made of an ordered sequence of frames. While the frames constitue
**spatiality** the sequence of those frames constitute the **temporality** of a video. To
systematically model both of these aspects we generally use a hybrid architecture that
consists of convolutions (for spatiality) as well as recurrent layers (for temporality).
In this example, we will be using such a hybrid architecture consisting of a
Convolutional Neural Network (CNN) and a Recurrent Neural Network (RNN) consisting of
[GRU layers](https://keras.io/api/layers/recurrent_layers/gru/). These kinds of hybrid
architectures are popularly known as **CNN-RNN**.

This example requires TensorFlow 2.5 or higher, as well as TensorFlow Docs, which can be
installed using the following command:


```python
!pip install -q git+https://github.com/tensorflow/docs
```

---
## Data collection

In order to keep the runtime of this example relatively short, we will be using a
subsampled version of the original UCF101 dataset. You can refer to [this notebook](https://colab.research.google.com/github/sayakpaul/Action-Recognition-in-TensorFlow/blob/main/Data_Preparation_UCF101.ipynb) to know how the subsampling was done.


```python
!wget -q https://git.io/JGc31 -O ucf101_top5.tar.gz
!tar xf ucf101_top5.tar.gz
```

---
## Setup


```python
from tensorflow_docs.vis import embed
from tensorflow import keras
from imutils import paths

import matplotlib.pyplot as plt
import tensorflow as tf
import pandas as pd
import numpy as np
import imageio
import cv2
import os
```

---
## Define hyperparameters


```python
IMG_SIZE = 224
BATCH_SIZE = 64
EPOCHS = 10

MAX_SEQ_LENGTH = 20
NUM_FEATURES = 2048
```

---
## Data preparation


```python
train_df = pd.read_csv("train.csv")
test_df = pd.read_csv("test.csv")

print(f"Total videos for training: {len(train_df)}")
print(f"Total videos for testing: {len(test_df)}")

train_df.sample(10)
```

<div class="k-default-codeblock">
```
Total videos for training: 594
Total videos for testing: 224

```
</div>
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

<div class="k-default-codeblock">
```
.dataframe tbody tr th {
    vertical-align: top;
}

.dataframe thead th {
    text-align: right;
}
```
</div>
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>video_name</th>
      <th>tag</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>401</th>
      <td>v_ShavingBeard_g14_c06.avi</td>
      <td>ShavingBeard</td>
    </tr>
    <tr>
      <th>397</th>
      <td>v_ShavingBeard_g14_c02.avi</td>
      <td>ShavingBeard</td>
    </tr>
    <tr>
      <th>30</th>
      <td>v_CricketShot_g12_c03.avi</td>
      <td>CricketShot</td>
    </tr>
    <tr>
      <th>237</th>
      <td>v_PlayingCello_g25_c07.avi</td>
      <td>PlayingCello</td>
    </tr>
    <tr>
      <th>92</th>
      <td>v_CricketShot_g22_c01.avi</td>
      <td>CricketShot</td>
    </tr>
    <tr>
      <th>527</th>
      <td>v_TennisSwing_g15_c03.avi</td>
      <td>TennisSwing</td>
    </tr>
    <tr>
      <th>29</th>
      <td>v_CricketShot_g12_c02.avi</td>
      <td>CricketShot</td>
    </tr>
    <tr>
      <th>480</th>
      <td>v_TennisSwing_g08_c04.avi</td>
      <td>TennisSwing</td>
    </tr>
    <tr>
      <th>444</th>
      <td>v_ShavingBeard_g21_c02.avi</td>
      <td>ShavingBeard</td>
    </tr>
    <tr>
      <th>4</th>
      <td>v_CricketShot_g08_c05.avi</td>
      <td>CricketShot</td>
    </tr>
  </tbody>
</table>
</div>



One of the many challenges of training video classifiers is figuring out a way to feed
the videos to a network. [This blog post](https://blog.coast.ai/five-video-classification-methods-implemented-in-keras-and-tensorflow-99cad29cc0b5) discusses
five such methods. As a video is an ordered sequence of frames, we can extract the
frames, organize them, and then feed them to our network. But the number of frames may
differ which would not allow mini-batch learning. To account for all these factors,
we can do the following:

1. Capture the frames of a video.
2. Extract frames from the videos until a maximum frame count is reached.
3. In the case, where a video's frame count is lesser than the maximum frame count we
will pad the video with zeros.

Note that this workflow is identical to [problems involving texts sequences](https://developers.google.com/machine-learning/guides/text-classification/). Videos of the UCF101 dataset is [known](https://www.crcv.ucf.edu/papers/UCF101_CRCV-TR-12-01.pdf)
to not contain extreme variations in objects and actions across frames. Because of this,
it may be okay to only consider a few frames for the learning task. But this approach may
not generalize well to other video classification problems. We will be using
[OpenCV's `VideoCapture()` method](https://docs.opencv.org/master/dd/d43/tutorial_py_video_display.html) to read
frames from videos.


```python
# The following two methods are taken from this tutorial:
# https://www.tensorflow.org/hub/tutorials/action_recognition_with_tf_hub


def crop_center_square(frame):
    y, x = frame.shape[0:2]
    min_dim = min(y, x)
    start_x = (x // 2) - (min_dim // 2)
    start_y = (y // 2) - (min_dim // 2)
    return frame[start_y : start_y + min_dim, start_x : start_x + min_dim]


def load_video(path, max_frames=0, resize=(IMG_SIZE, IMG_SIZE)):
    cap = cv2.VideoCapture(path)
    frames = []
    try:
        while True:
            ret, frame = cap.read()
            if not ret:
                break
            frame = crop_center_square(frame)
            frame = cv2.resize(frame, resize)
            frame = frame[:, :, [2, 1, 0]]
            frames.append(frame)

            if len(frames) == max_frames:
                break
    finally:
        cap.release()
    return np.array(frames)

```

We can use a pre-trained network to extract meaningful features from the extracted
frames. The [`Applications`](https://keras.io/api/applications/) class of Keras provides
a number of state-of-the-art models pre-trained on the [ImageNet-1k dataset](http://image-net.org/).
We will be using the [InceptionV3 model](https://arxiv.org/abs/1512.00567) for this purpose.


```python

def build_feature_extractor():
    feature_extractor = keras.applications.InceptionV3(
        weights="imagenet",
        include_top=False,
        pooling="avg",
        input_shape=(IMG_SIZE, IMG_SIZE, 3),
    )
    preprocess_input = keras.applications.inception_v3.preprocess_input

    inputs = keras.Input((IMG_SIZE, IMG_SIZE, 3))
    preprocessed = preprocess_input(inputs)

    outputs = feature_extractor(preprocessed)
    return keras.Model(inputs, outputs, name="feature_extractor")


feature_extractor = build_feature_extractor()
```

The labels of the videos are strings and since neural networks can not process strings we
need to convert these labels into integers. Here we will use the
[`StringLookup`](https://keras.io/api/layers/preprocessing_layers/categorical/string_lookup) layer
encode the class labels as integers.


```python
label_processor = keras.layers.experimental.preprocessing.StringLookup(
    num_oov_indices=0, vocabulary=np.unique(train_df["tag"])
)
print(label_processor.get_vocabulary())
```

<div class="k-default-codeblock">
```
['', 'CricketShot', 'PlayingCello', 'Punch', 'ShavingBeard', 'TennisSwing']

```
</div>
Finally, we can put all the pieces together to create our data processing utility.


```python

def prepare_all_videos(df, root_dir):
    num_samples = len(df)
    video_paths = df["video_name"].values.tolist()
    labels = df["tag"].values
    labels = label_processor(labels[..., None]).numpy()

    # `frame_masks` and `frame_features` are what we will feed to our sequence model.
    # `frame_masks` will contain a bunch of booleans denoting if a timestep is
    # masked with padding or not.
    frame_masks = np.zeros(shape=(num_samples, MAX_SEQ_LENGTH), dtype="bool")
    frame_features = np.zeros(
        shape=(num_samples, MAX_SEQ_LENGTH, NUM_FEATURES), dtype="float32"
    )

    # For each video.
    for idx, path in enumerate(video_paths):
        # Gather all its frames and add a batch dimension.
        frames = load_video(os.path.join(root_dir, path))
        frames = frames[None, ...]

        # Initialize placeholders to store the masks and features of the current video.
        temp_frame_mask = np.zeros(shape=(1, MAX_SEQ_LENGTH,), dtype="bool")
        temp_frame_featutes = np.zeros(
            shape=(1, MAX_SEQ_LENGTH, NUM_FEATURES), dtype="float32"
        )

        # Extract features from the frames of the current video.
        for i, batch in enumerate(frames):
            video_length = batch.shape[1]
            length = min(MAX_SEQ_LENGTH, video_length)
            for j in range(length):
                temp_frame_featutes[i, j, :] = feature_extractor.predict(
                    batch[None, j, :]
                )
            temp_frame_mask[i, :length] = 1  # 1 = not masked, 0 = masked

        frame_features[idx,] = temp_frame_featutes.squeeze()
        frame_masks[idx,] = temp_frame_mask.squeeze()

    return (frame_features, frame_masks), labels


train_data, train_labels = prepare_all_videos(train_df, "train")
test_data, test_labels = prepare_all_videos(test_df, "test")

print(f"Frame features in train set: {train_data[0].shape}")
print(f"Frame masks in train set: {train_data[1].shape}")
```

<div class="k-default-codeblock">
```
Frame features in train set: (594, 20, 2048)
Frame masks in train set: (594, 20)

```
</div>
The above code block will take ~20 minutes to execute depending on the machine it's being
executed.

---
## The sequence model

Now, we can feed this data to a sequence model consisting of recurrent layers like `GRU`.


```python
# Utility for our sequence model.
def get_sequence_model():
    class_vocab = label_processor.get_vocabulary()

    frame_features_input = keras.Input((MAX_SEQ_LENGTH, NUM_FEATURES))
    mask_input = keras.Input((MAX_SEQ_LENGTH,), dtype="bool")

    # Refer to the following tutorial to understand the significance of using `mask`:
    # https://keras.io/api/layers/recurrent_layers/gru/
    x = keras.layers.GRU(16, return_sequences=True)(
        frame_features_input, mask=mask_input
    )
    x = keras.layers.GRU(8)(x)
    x = keras.layers.Dropout(0.4)(x)
    x = keras.layers.Dense(8, activation="relu")(x)
    output = keras.layers.Dense(len(class_vocab), activation="softmax")(x)

    rnn_model = keras.Model([frame_features_input, mask_input], output)

    rnn_model.compile(
        loss="sparse_categorical_crossentropy", optimizer="adam", metrics=["accuracy"]
    )
    return rnn_model


# Utility for running experiments.
def run_experiment():
    filepath = "/tmp/video_classifier"
    checkpoint = keras.callbacks.ModelCheckpoint(
        filepath, save_weights_only=True, save_best_only=True, verbose=1
    )

    seq_model = get_sequence_model()
    history = seq_model.fit(
        [train_data[0], train_data[1]],
        train_labels,
        validation_split=0.3,
        epochs=EPOCHS,
        callbacks=[checkpoint],
    )

    seq_model.load_weights(filepath)
    _, accuracy = seq_model.evaluate([test_data[0], test_data[1]], test_labels)
    print(f"Test accuracy: {round(accuracy * 100, 2)}%")

    return history, seq_model


_, sequence_model = run_experiment()
```

<div class="k-default-codeblock">
```
Epoch 1/10
13/13 [==============================] - 9s 194ms/step - loss: 1.8309 - accuracy: 0.1331 - val_loss: 1.7844 - val_accuracy: 0.0000e+00
```
</div>
    
<div class="k-default-codeblock">
```
Epoch 00001: val_loss improved from inf to 1.78437, saving model to /tmp/video_classifier
Epoch 2/10
13/13 [==============================] - 0s 16ms/step - loss: 1.6009 - accuracy: 0.3854 - val_loss: 1.8179 - val_accuracy: 0.0000e+00
```
</div>
    
<div class="k-default-codeblock">
```
Epoch 00002: val_loss did not improve from 1.78437
Epoch 3/10
13/13 [==============================] - 0s 16ms/step - loss: 1.4859 - accuracy: 0.4585 - val_loss: 1.8392 - val_accuracy: 0.0056
```
</div>
    
<div class="k-default-codeblock">
```
Epoch 00003: val_loss did not improve from 1.78437
Epoch 4/10
13/13 [==============================] - 0s 16ms/step - loss: 1.3842 - accuracy: 0.6205 - val_loss: 1.7819 - val_accuracy: 0.3296
```
</div>
    
<div class="k-default-codeblock">
```
Epoch 00004: val_loss improved from 1.78437 to 1.78192, saving model to /tmp/video_classifier
Epoch 5/10
13/13 [==============================] - 0s 16ms/step - loss: 1.3119 - accuracy: 0.7185 - val_loss: 1.7764 - val_accuracy: 0.3128
```
</div>
    
<div class="k-default-codeblock">
```
Epoch 00005: val_loss improved from 1.78192 to 1.77637, saving model to /tmp/video_classifier
Epoch 6/10
13/13 [==============================] - 0s 16ms/step - loss: 1.2164 - accuracy: 0.7711 - val_loss: 1.7850 - val_accuracy: 0.3240
```
</div>
    
<div class="k-default-codeblock">
```
Epoch 00006: val_loss did not improve from 1.77637
Epoch 7/10
13/13 [==============================] - 0s 16ms/step - loss: 1.1071 - accuracy: 0.8066 - val_loss: 1.7833 - val_accuracy: 0.3408
```
</div>
    
<div class="k-default-codeblock">
```
Epoch 00007: val_loss did not improve from 1.77637
Epoch 8/10
13/13 [==============================] - 0s 16ms/step - loss: 1.0061 - accuracy: 0.9113 - val_loss: 1.8160 - val_accuracy: 0.3240
```
</div>
    
<div class="k-default-codeblock">
```
Epoch 00008: val_loss did not improve from 1.77637
Epoch 9/10
13/13 [==============================] - 0s 15ms/step - loss: 0.9399 - accuracy: 0.8684 - val_loss: 1.8548 - val_accuracy: 0.3240
```
</div>
    
<div class="k-default-codeblock">
```
Epoch 00009: val_loss did not improve from 1.77637
Epoch 10/10
13/13 [==============================] - 0s 15ms/step - loss: 0.8665 - accuracy: 0.8892 - val_loss: 1.8832 - val_accuracy: 0.3408
```
</div>
    
<div class="k-default-codeblock">
```
Epoch 00010: val_loss did not improve from 1.77637
7/7 [==============================] - 1s 5ms/step - loss: 1.3853 - accuracy: 0.7545
Test accuracy: 75.45%

```
</div>
**Note**: To keep the runtime of this example relatively short, we just used a few
training examples. This number of training examples is low with respect to the sequence
model being used that has 99,909 trainable parameters. You are encouraged to sample more
data from the UCF101 dataset using [the notebook](https://colab.research.google.com/github/sayakpaul/Action-Recognition-in-TensorFlow/blob/main/Data_Preparation_UCF101.ipynb) mentioned above and train the same model.

---
## Inference


```python

def prepare_single_video(frames):
    frames = frames[None, ...]
    frame_mask = np.zeros(shape=(1, MAX_SEQ_LENGTH,), dtype="bool")
    frame_featutes = np.zeros(shape=(1, MAX_SEQ_LENGTH, NUM_FEATURES), dtype="float32")

    for i, batch in enumerate(frames):
        video_length = batch.shape[1]
        length = min(MAX_SEQ_LENGTH, video_length)
        for j in range(length):
            frame_featutes[i, j, :] = feature_extractor.predict(batch[None, j, :])
        frame_mask[i, :length] = 1  # 1 = not masked, 0 = masked

    return frame_featutes, frame_mask


def sequence_prediction(path):
    class_vocab = label_processor.get_vocabulary()

    frames = load_video(os.path.join("test", path))
    frame_features, frame_mask = prepare_single_video(frames)
    probabilities = sequence_model.predict([frame_features, frame_mask])[0]

    for i in np.argsort(probabilities)[::-1]:
        print(f"  {class_vocab[i]}: {probabilities[i] * 100:5.2f}%")
    return frames


# This utility is for visualization.
# Referenced from:
# https://www.tensorflow.org/hub/tutorials/action_recognition_with_tf_hub
def to_gif(images):
    converted_images = images.astype(np.uint8)
    imageio.mimsave("animation.gif", converted_images, fps=10)
    return embed.embed_file("animation.gif")


test_video = np.random.choice(test_df["video_name"].values.tolist())
print(f"Test video path: {test_video}")
test_frames = sequence_prediction(test_video)
to_gif(test_frames[:MAX_SEQ_LENGTH])
```

<div class="k-default-codeblock">
```
Test video path: v_ShavingBeard_g01_c02.avi
  ShavingBeard: 28.89%
  CricketShot: 22.33%
  TennisSwing: 20.33%
  Punch: 12.89%
  PlayingCello:  9.52%
  :  6.04%

```
</div>



---
## Next steps

* In this example, we made use of transfer learning for extracting meaningful features
from video frames. You could also fine-tune the pre-trained network to notice how that
affects the end results.
* For speed-accuracy trade-offs, you can try out other models present inside
`tf.keras.applications`.
* Try different combinations of `MAX_SEQ_LENGTH` to observe how that affects the
performance.
* Train on a higher number of classes and see if you are able to get good performance.
* Following [this tutorial](https://www.tensorflow.org/hub/tutorials/action_recognition_with_tf_hub), try a
[pre-trained action recognition model](https://arxiv.org/abs/1705.07750) from DeepMind.
* Rolling-averaging can be useful technique for video classification and it can be
combined with a standard image classification model to infer on videos. [This tutorial](https://www.pyimagesearch.com/2019/07/15/video-classification-with-keras-and-deep-learning/)
will help understand how to use rolling-averaging with an image classifier.
* When there are variations in between the frames of a video not all the frames might be
equally important to decide its category. In those situations, putting a
[self-attention layer](https://www.tensorflow.org/api_docs/python/tf/keras/layers/Attention) in the
sequence model will likely yield better results.
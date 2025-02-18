# The Image Classification Dataset
:label:`sec_fashion_mnist`

(~~The MNIST dataset is one of the widely used dataset for image classification, while it's too simple as a benchmark dataset. We will use the similar, but more complex Fashion-MNIST dataset~~)

One of the widely used dataset for image classification is the  MNIST dataset :cite:`LeCun.Bottou.Bengio.ea.1998`.
While it had a good run as a benchmark dataset,
even simple models by today's standards achieve classification accuracy over 95%,
making it unsuitable for distinguishing between stronger models and weaker ones.
Today, MNIST serves as more of sanity checks than as a benchmark.
To up the ante just a bit, we will focus our discussion in the coming sections
on the qualitatively similar, but comparatively complex Fashion-MNIST
dataset :cite:`Xiao.Rasul.Vollgraf.2017`, which was released in 2017.

```{.python .input}
%matplotlib inline
from d2l import mxnet as d2l
from mxnet import gluon
import time

d2l.use_svg_display()
```

```{.python .input}
#@tab pytorch
%matplotlib inline
import time
from d2l import torch as d2l
import torch
import torchvision
from torchvision import transforms

d2l.use_svg_display()
```

```{.python .input}
#@tab tensorflow
%matplotlib inline
import time
from d2l import tensorflow as d2l
import tensorflow as tf

d2l.use_svg_display()
```

## Reading the Dataset

We can [**download and read the Fashion-MNIST dataset into memory via the build-in functions in the framework.**]

```{.python .input}
class FashionMNIST(d2l.DataModule):  #@save
    def __init__(self, batch_size=32, resize=(28, 28)):
        super().__init__()
        self.save_hyperparameters()
        self.train = gluon.data.vision.FashionMNIST(train=True)
        self.val = gluon.data.vision.FashionMNIST(train=False)
```

```{.python .input}
#@tab pytorch
class FashionMNIST(d2l.DataModule):  #@save
    def __init__(self, batch_size=32, resize=(28, 28)):
        super().__init__()
        self.save_hyperparameters()
        trans = transforms.Compose([transforms.Resize(resize),
                                    transforms.ToTensor()])
        self.train = torchvision.datasets.FashionMNIST(
            root=self.root, train=True, transform=trans, download=True)
        self.val = torchvision.datasets.FashionMNIST(
            root=self.root, train=False, transform=trans, download=True)
```

```{.python .input}
#@tab tensorflow
class FashionMNIST(d2l.DataModule):  #@save
    def __init__(self, batch_size=32, resize=(28, 28)):
        super().__init__()
        self.save_hyperparameters()
        self.train, self.val = tf.keras.datasets.fashion_mnist.load_data()
```

Fashion-MNIST consists of images from 10 categories, each represented
by 6000 images in the training dataset and by 1000 in the test dataset.
A *test dataset* (or *test set*) is used for evaluating  model performance and not for training.
Consequently the training set and the test set
contain 60000 and 10000 images, respectively.

```{.python .input}
#@tab mxnet, pytorch
data = FashionMNIST()
len(data.train), len(data.val)
```

```{.python .input}
#@tab tensorflow
data = FashionMNIST()
len(data.train[0]), len(data.val[0])
```

The height and width of each input image are both 28 pixels.
Note that the dataset consists of grayscale images, whose number of channels is 1.
For brevity, throughout this book
we store the shape of any image with height $h$ width $w$ pixels as $h \times w$ or ($h$, $w$).

```{.python .input}
#@tab all
data.train[0][0].shape
```

[~~Two utility functions to visualize the dataset~~]

The images in Fashion-MNIST are associated with the following categories:
t-shirt, trousers, pullover, dress, coat, sandal, shirt, sneaker, bag, and ankle boot.
The following function converts between numeric label indices and their names in text.

```{.python .input}
#@tab all
@d2l.add_to_class(FashionMNIST)  #@save
def text_labels(self, indices):
    """Return text labels."""
    labels = ['t-shirt', 'trouser', 'pullover', 'dress', 'coat',
              'sandal', 'shirt', 'sneaker', 'bag', 'ankle boot']
    return [labels[int(i)] for i in indices]
```

We can now create a function to visualize these examples.

```{.python .input}
#@tab all
def show_images(imgs, num_rows, num_cols, titles=None, scale=1.5):  #@save
    """Plot a list of images."""
    raise NotImplementedError
```

Here are [**the images and their corresponding labels**] (in text)
for the first few examples in the training dataset.

```{.python .input}
X, y = data.train[:18]
d2l.show_images(X.squeeze(axis=-1), 2, 9, titles=data.text_labels(y));
```

```{.python .input}
#@tab pytorch
X = [data.train[i][0].squeeze(0) for i in range(18)]
y = [data.train[i][1] for i in range(18)]
d2l.show_images(X, 2, 9, titles=data.text_labels(y));
```

```{.python .input}
#@tab tensorflow
X = tf.constant(data.train[0][:18])
y = tf.constant(data.train[1][:18])
d2l.show_images(X, 2, 9, titles=data.text_labels(y));
```

## Reading a Minibatch

To make our life easier when reading from the training and test sets,
we use the built-in data iterator rather than creating one from scratch.
Recall that at each iteration, a data iterator
[**reads a minibatch of data with size `batch_size` each time.**]
We also randomly shuffle the examples for the training data iterator.

```{.python .input}
@d2l.add_to_class(FashionMNIST)  #@save
def train_dataloader(self):
    trans = gluon.data.vision.transforms.ToTensor()
    return gluon.data.DataLoader(
        self.train.transform_first(trans), self.batch_size,
        shuffle=True, num_workers=self.num_workers)

@d2l.add_to_class(FashionMNIST)  #@save
def val_dataloader(self):
    trans = gluon.data.vision.transforms.ToTensor()
    return gluon.data.DataLoader(
        self.val.transform_first(trans), self.batch_size,
        shuffle=False, num_workers=self.num_workers)

```

```{.python .input}
#@tab pytorch
@d2l.add_to_class(FashionMNIST)  #@save
def train_dataloader(self):
    return torch.utils.data.DataLoader(
        self.train, self.batch_size, shuffle=True,
        num_workers=self.num_workers)

@d2l.add_to_class(FashionMNIST)  #@save
def val_dataloader(self):
    return torch.utils.data.DataLoader(
        self.val, self.batch_size, shuffle=False,
        num_workers=self.num_workers)
```

```{.python .input}
#@tab tensorflow
@d2l.add_to_class(FashionMNIST)  #@save
def train_dataloader(self):
    return tf.data.Dataset.from_tensor_slices(
        self.train).batch(self.batch_size).shuffle(len(self.train[0]))

@d2l.add_to_class(FashionMNIST)  #@save
def val_dataloader(self):
    return tf.data.Dataset.from_tensor_slices(
        self.val).batch(self.batch_size)
```

Let us look at the time it takes to read the training data.

```{.python .input}
#@tab all
tic = time.time()
for X, y in data.train_dataloader():
    continue
f'{time.time() - tic:.2f} sec'
```

We are now ready to work with the Fashion-MNIST dataset in the sections that follow.

## Summary

* Fashion-MNIST is an apparel classification dataset consisting of images representing 10 categories. We will use this dataset in subsequent sections and chapters to evaluate various classification algorithms.
* We store the shape of any image with height $h$ width $w$ pixels as $h \times w$ or ($h$, $w$).
* Data iterators are a key component for efficient performance. Rely on well-implemented data iterators that exploit high-performance computing to avoid slowing down your training loop.


## Exercises

1. Does reducing the `batch_size` (for instance, to 1) affect the reading performance?
1. The data iterator performance is important. Do you think the current implementation is fast enough? Explore various options to improve it.
1. Check out the framework's online API documentation. Which other datasets are available?

:begin_tab:`mxnet`
[Discussions](https://discuss.d2l.ai/t/48)
:end_tab:

:begin_tab:`pytorch`
[Discussions](https://discuss.d2l.ai/t/49)
:end_tab:

:begin_tab:`tensorflow`
[Discussions](https://discuss.d2l.ai/t/224)
:end_tab:

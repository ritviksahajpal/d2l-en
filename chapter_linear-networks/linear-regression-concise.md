# Concise Implementation of Linear Regression
:label:`sec_linear_concise`

Broad and intense interest in deep learning for the past several years
has inspired companies, academics, and hobbyists
to develop a variety of mature open source frameworks
for automating the repetitive work of implementing
gradient-based learning algorithms.
In :numref:`sec_linear_scratch`, we relied only on
(i) tensors for data storage and linear algebra;
and (ii) auto differentiation for calculating gradients.
In practice, because data iterators, loss functions, optimizers,
and neural network layers
are so common, modern libraries implement these components for us as well.

In this section, (**we will show you how to implement
the linear regression model**) from :numref:`sec_linear_scratch`
(**concisely by using high-level APIs**) of deep learning frameworks.


## Generating the Dataset

To start, we will generate the same dataset as in :numref:`sec_linear_scratch`.

```{.python .input}
from d2l import mxnet as d2l
from mxnet import autograd, gluon, np, npx
npx.set_np()
```

```{.python .input}
#@tab pytorch
from d2l import torch as d2l
import numpy as np
import torch
```

```{.python .input}
#@tab tensorflow
from d2l import tensorflow as d2l
import numpy as np
import tensorflow as tf
```

## Reading the Dataset

Rather than rolling our own iterator,
we can [**call upon the existing API in a framework to read data.**]
We pass in `features` and `labels` as arguments and specify `batch_size`
when instantiating a data iterator object.
Besides, the boolean value `is_train`
indicates whether or not
we want the data iterator object to shuffle the data
on each epoch (pass through the dataset).

```{.python .input}
@d2l.add_to_class(d2l.SyntheticRegressionData)  #@save
def train_dataloader(self):
    dataset = gluon.data.ArrayDataset(self.X, self.y)
    return gluon.data.DataLoader(dataset, self.batch_size, shuffle=True)
```

```{.python .input}
#@tab pytorch
@d2l.add_to_class(d2l.SyntheticRegressionData)  #@save
def train_dataloader(self):
    dataset = torch.utils.data.TensorDataset(self.X, self.y)
    return torch.utils.data.DataLoader(dataset, self.batch_size, shuffle=True)
```

```{.python .input}
#@tab tensorflow
@d2l.add_to_class(d2l.SyntheticRegressionData)  #@save
def train_dataloader(self):
    return tf.data.Dataset.from_tensor_slices(
        (self.X, self.y)).shuffle(buffer_size=1000).batch(self.batch_size)
```

```{.python .input}
#@tab all
data = d2l.SyntheticRegressionData(w=d2l.tensor([2, -3.4]), b=4.2)
```

Now we can use `data_iter` in much the same way as we called
the `data_iter` function in :numref:`sec_linear_scratch`.
To verify that it is working, we can read and print
the first minibatch of examples.
Comparing with :numref:`sec_linear_scratch`,
here we use `iter` to construct a Python iterator and use `next` to obtain the first item from the iterator.

```{.python .input}
#@tab all
next(iter(data.train_dataloader()))
```

## Defining the Model

When we implemented linear regression from scratch
in :numref:`sec_linear_scratch`,
we defined our model parameters explicitly
and coded up the calculations to produce output
using basic linear algebra operations.
You *should* know how to do this.
But once your models get more complex,
and once you have to do this nearly every day,
you will be glad for the assistance.
The situation is similar to coding up your own blog from scratch.
Doing it once or twice is rewarding and instructive,
but you would be a lousy web developer
if every time you needed a blog you spent a month
reinventing the wheel.

For standard operations, we can [**use a framework's predefined layers,**]
which allow us to focus especially
on the layers used to construct the model
rather than having to focus on the implementation.
We will first define a model variable `net`,
which will refer to an instance of the `Sequential` class.
The `Sequential` class defines a container
for several layers that will be chained together.
Given input data, a `Sequential` instance passes it through
the first layer, in turn passing the output
as the second layer's input and so forth.
In the following example, our model consists of only one layer,
so we do not really need `Sequential`.
But since nearly all of our future models
will involve multiple layers,
we will use it anyway just to familiarize you
with the most standard workflow.

Recall the architecture of a single-layer network as shown in :numref:`fig_single_neuron`.
The layer is said to be *fully connected*
because each of its inputs is connected to each of its outputs
by means of a matrix-vector multiplication.

:begin_tab:`mxnet`
In Gluon, the fully connected layer is defined in the `Dense` class.
Since we only want to generate a single scalar output,
we set that number to 1.

It is worth noting that, for convenience,
Gluon does not require us to specify
the input shape for each layer.
So here, we do not need to tell Gluon
how many inputs go into this linear layer.
When we first try to pass data through our model,
e.g., when we execute `net(X)` later,
Gluon will automatically infer the number of inputs to each layer.
We will describe how this works in more detail later.
:end_tab:

:begin_tab:`pytorch`
In PyTorch, the fully connected layer is defined in the `Linear` class. Note that we passed two arguments into `nn.Linear`. The first one specifies the input feature dimension, which is 2, and the second one is the output feature dimension, which is a single scalar and therefore 1.
:end_tab:

:begin_tab:`tensorflow`
In Keras, the fully connected layer is defined in the `Dense` class. Since we only want to generate a single scalar output, we set that number to 1.

It is worth noting that, for convenience,
Keras does not require us to specify
the input shape for each layer.
So here, we do not need to tell Keras
how many inputs go into this linear layer.
When we first try to pass data through our model,
e.g., when we execute `net(X)` later,
Keras will automatically infer the number of inputs to each layer.
We will describe how this works in more detail later.
:end_tab:

```{.python .input}
# `nn` is an abbreviation for neural networks
from mxnet.gluon import nn

class LinearRegression(d2l.Module):  #@save
    def __init__(self, num_inputs, num_outputs, lr):
        super().__init__()
        self.save_hyperparameters()
        self.net = nn.Dense(num_outputs)
        self.net.initialize()
```

```{.python .input}
#@tab pytorch
# `nn` is an abbreviation for neural networks
from torch import nn

class LinearRegression(d2l.Module):  #@save
    def __init__(self, num_inputs, num_outputs, lr):
        super().__init__()
        self.save_hyperparameters()
        self.net = nn.Linear(num_inputs, num_outputs)
```

```{.python .input}
#@tab tensorflow
# `keras` is the high-level API for TensorFlow

class LinearRegression(d2l.Module):  #@save
    def __init__(self, num_inputs, num_outputs, lr):
        super().__init__()
        self.save_hyperparameters()
        self.net = tf.keras.layers.Dense(num_outputs)
```

## Initializing Model Parameters

Before using `net`, we need to (**initialize the model parameters,**)
such as the weights and bias in the linear regression model.
Deep learning frameworks often have a predefined way to initialize the parameters.
Here we specify that each weight parameter
should be randomly sampled from a normal distribution
with mean 0 and standard deviation 0.01.
The bias parameter will be initialized to zero.

:begin_tab:`mxnet`
We will import the `initializer` module from MXNet.
This module provides various methods for model parameter initialization.
Gluon makes `init` available as a shortcut (abbreviation)
to access the `initializer` package.
We only specify how to initialize the weight by calling `init.Normal(sigma=0.01)`.
Bias parameters are initialized to zero by default.
:end_tab:

:begin_tab:`pytorch`
As we have specified the input and output dimensions when constructing `nn.Linear`,
now we can access the parameters directly to specify their initial values.
We first locate the layer by `net[0]`, which is the first layer in the network,
and then use the `weight.data` and `bias.data` methods to access the parameters.
Next we use the replace methods `normal_` and `fill_` to overwrite parameter values.
:end_tab:

:begin_tab:`tensorflow`
The `initializers` module in TensorFlow provides various methods for model parameter initialization. The easiest way to specify the initialization method in Keras is when creating the layer by specifying `kernel_initializer`. Here we recreate `net` again.
:end_tab:

```{.python .input}
#from mxnet import init
#net.initialize(init.Normal(sigma=0.01))
```

```{.python .input}
#@tab tensorflow
#initializer = tf.initializers.RandomNormal(stddev=0.01)
#net = tf.keras.Sequential()
#net.add(tf.keras.layers.Dense(1, kernel_initializer=initializer))
```

:begin_tab:`mxnet`
The code above may look straightforward but you should note
that something strange is happening here.
We are initializing parameters for a network
even though Gluon does not yet know
how many dimensions the input will have!
It might be 2 as in our example or it might be 2000.
Gluon lets us get away with this because behind the scene,
the initialization is actually *deferred*.
The real initialization will take place only
when we for the first time attempt to pass data through the network.
Just be careful to remember that since the parameters
have not been initialized yet,
we cannot access or manipulate them.
:end_tab:

:begin_tab:`pytorch`

:end_tab:

:begin_tab:`tensorflow`
The code above may look straightforward but you should note
that something strange is happening here.
We are initializing parameters for a network
even though Keras does not yet know
how many dimensions the input will have!
It might be 2 as in our example or it might be 2000.
Keras lets us get away with this because behind the scenes,
the initialization is actually *deferred*.
The real initialization will take place only
when we for the first time attempt to pass data through the network.
Just be careful to remember that since the parameters
have not been initialized yet,
we cannot access or manipulate them.
:end_tab:

## Defining the Loss Function

:begin_tab:`mxnet`
In Gluon, the `loss` module defines various loss functions.
In this example, we will use the Gluon
implementation of squared loss (`L2Loss`).
:end_tab:

:begin_tab:`pytorch`
[**The `MSELoss` class computes the mean squared error, also known as squared $L_2$ norm.**]
By default it returns the average loss over examples.
:end_tab:

:begin_tab:`tensorflow`
The `MeanSquaredError` class computes the mean squared error, also known as squared $L_2$ norm.
By default it returns the average loss over examples.
:end_tab:

```{.python .input}
#@tab mxnet, pytorch
@d2l.add_to_class(LinearRegression)
def forward(self, X):
    """The linear regression model."""
    return self.net(X)
```

```{.python .input}
@d2l.add_to_class(LinearRegression)
def training_step(self, batch, batch_idx):
    X, y = batch
    loss = gluon.loss.L2Loss()
    l = loss(self(X), y).mean()
    self.board.draw({'step':batch_idx, 'loss':l}, every_n=10)
    return l
```

```{.python .input}
#@tab pytorch
@d2l.add_to_class(LinearRegression)
def training_step(self, batch, batch_idx):
    X, y = batch
    loss = nn.MSELoss()
    l = loss(self(X), y)
    self.board.draw({'step':batch_idx, 'loss':l}, every_n=10)
    return l
```

```{.python .input}
#@tab tensorflow
@d2l.add_to_class(LinearRegression)
def call(self, X):
    """The linear regression model."""
    return self.net(X)

@d2l.add_to_class(LinearRegression)
def training_step(self, batch, batch_idx):
    X, y = batch
    loss = tf.keras.losses.MeanSquaredError()
    l = loss(self(X), y)
    self.board.draw({'step':batch_idx, 'loss':l}, every_n=10)
    return l
```

## Defining the Optimization Algorithm

:begin_tab:`mxnet`
Minibatch stochastic gradient descent is a standard tool
for optimizing neural networks
and thus Gluon supports it alongside a number of
variations on this algorithm through its `Trainer` class.
When we instantiate `Trainer`,
we will specify the parameters to optimize over
(obtainable from our model `net` via `net.collect_params()`),
the optimization algorithm we wish to use (`sgd`),
and a dictionary of hyperparameters
required by our optimization algorithm.
Minibatch stochastic gradient descent just requires that
we set the value `learning_rate`, which is set to 0.03 here.
:end_tab:

:begin_tab:`pytorch`
Minibatch stochastic gradient descent is a standard tool
for optimizing neural networks
and thus PyTorch supports it alongside a number of
variations on this algorithm in the `optim` module.
When we (**instantiate an `SGD` instance,**)
we will specify the parameters to optimize over
(obtainable from our net via `net.parameters()`), with a dictionary of hyperparameters
required by our optimization algorithm.
Minibatch stochastic gradient descent just requires that
we set the value `lr`, which is set to 0.03 here.
:end_tab:

:begin_tab:`tensorflow`
Minibatch stochastic gradient descent is a standard tool
for optimizing neural networks
and thus Keras supports it alongside a number of
variations on this algorithm in the `optimizers` module.
Minibatch stochastic gradient descent just requires that
we set the value `learning_rate`, which is set to 0.03 here.
:end_tab:

```{.python .input}
@d2l.add_to_class(LinearRegression)
def configure_optimizers(self):
    return gluon.Trainer(self.collect_params(),
                         'sgd', {'learning_rate': self.lr})
```

```{.python .input}
#@tab pytorch
@d2l.add_to_class(LinearRegression)
def configure_optimizers(self):
    return torch.optim.SGD(self.parameters(), self.lr)
```

```{.python .input}
#@tab tensorflow
@d2l.add_to_class(LinearRegression)
def configure_optimizers(self):
    return tf.keras.optimizers.SGD(self.lr)
```

## Training

You might have noticed that expressing our model through
high-level APIs of a deep learning framework
requires comparatively few lines of code.
We did not have to individually allocate parameters,
define our loss function, or implement minibatch stochastic gradient descent.
Once we start working with much more complex models,
advantages of high-level APIs will grow considerably.
However, once we have all the basic pieces in place,
[**the training loop itself is strikingly similar
to what we did when implementing everything from scratch.**]

To refresh your memory: for some number of epochs,
we will make a complete pass over the dataset (`train_data`),
iteratively grabbing one minibatch of inputs
and the corresponding ground-truth labels.
For each minibatch, we go through the following ritual:

* Generate predictions by calling `net(X)` and calculate the loss `l` (the forward propagation).
* Calculate gradients by running the backpropagation.
* Update the model parameters by invoking our optimizer.

For good measure, we compute the loss after each epoch and print it to monitor progress.

```{.python .input}
#@tab all
model = LinearRegression(2, 1, lr=0.03)
trainer = d2l.Trainer(3)
trainer.fit(model, data)
```

Below, we [**compare the model parameters learned by training on finite data
and the actual parameters**] that generated our dataset.
To access parameters,
we first access the layer that we need from `net`
and then access that layer's weights and bias.
As in our from-scratch implementation,
note that our estimated parameters are
close to their ground-truth counterparts.

```{.python .input}
w = model.net.weight.data()
b = model.net.bias.data()
```

```{.python .input}
#@tab pytorch
w = model.net.weight.data
b = model.net.bias.data
```

```{.python .input}
#@tab tensorflow
w = model.get_weights()[0]
b = model.get_weights()[1]
```

```{.python .input}
#@tab all
print(f'error in estimating w: {data.w - d2l.reshape(w, data.w.shape)}')
print(f'error in estimating b: {data.b - b}')
```

## Summary

:begin_tab:`mxnet`
* Using Gluon, we can implement models much more concisely.
* In Gluon, the `data` module provides tools for data processing, the `nn` module defines a large number of neural network layers, and the `loss` module defines many common loss functions.
* MXNet's module `initializer` provides various methods for model parameter initialization.
* Dimensionality and storage are automatically inferred, but be careful not to attempt to access parameters before they have been initialized.
:end_tab:

:begin_tab:`pytorch`
* Using PyTorch's high-level APIs, we can implement models much more concisely.
* In PyTorch, the `data` module provides tools for data processing, the `nn` module defines a large number of neural network layers and common loss functions.
* We can initialize the parameters by replacing their values with methods ending with `_`.
:end_tab:

:begin_tab:`tensorflow`
* Using TensorFlow's high-level APIs, we can implement models much more concisely.
* In TensorFlow, the `data` module provides tools for data processing, the `keras` module defines a large number of neural network layers and common loss functions.
* TensorFlow's module `initializers` provides various methods for model parameter initialization.
* Dimensionality and storage are automatically inferred (but be careful not to attempt to access parameters before they have been initialized).
:end_tab:

## Exercises

:begin_tab:`mxnet`
1. If we replace `l = loss(output, y)` with `l = loss(output, y).mean()`, we need to change `trainer.step(batch_size)` to `trainer.step(1)` for the code to behave identically. Why?
1. Review the MXNet documentation to see what loss functions and initialization methods are provided in the modules `gluon.loss` and `init`. Replace the loss by Huber's loss.
1. How do you access the gradient of `dense.weight`?

[Discussions](https://discuss.d2l.ai/t/44)
:end_tab:

:begin_tab:`pytorch`
1. If we replace `nn.MSELoss(reduction='sum')` with `nn.MSELoss()`, how can we change the learning rate for the code to behave identically. Why?
1. Review the PyTorch documentation to see what loss functions and initialization methods are provided. Replace the loss by Huber's loss.
1. How do you access the gradient of `net[0].weight`?

[Discussions](https://discuss.d2l.ai/t/45)
:end_tab:

:begin_tab:`tensorflow`
1. Review the TensorFlow documentation to see what loss functions and initialization methods are provided. Replace the loss by Huber's loss.

[Discussions](https://discuss.d2l.ai/t/204)
:end_tab:

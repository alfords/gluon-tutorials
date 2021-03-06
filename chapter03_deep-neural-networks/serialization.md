# Serialization - saving, loading and checkpointing

At this point we've already covered quite a lot of ground. 
We know how to manipulate data and labels.
We know how to construct flexible models capable of expressing plausible hypotheses.
We know how to fit those models to our dataset.
We've know of loss functions to use for classification and for regression,
and we know how to minimize those losses with respect to our models' parameters. 
We even know how to write our own neural network layers in ``gluon``.

But even with all this knowledge, we're not ready to build a real machine learning system.
That's because we haven't yet covered how to save and load models. 
In reality, we often  train a model on one device
and then want to run it to make predictions on many devices simultaneously.
In order for our models to persist beyond the execution of a single Python script, 
we need mechanisms to save and load NDArrays, ``gluon`` Parameters, and models themselves.

```{.python .input  n=1}
from __future__ import print_function
import mxnet as mx
from mxnet import nd, autograd
from mxnet import gluon
ctx = mx.gpu()
```

## Saving and loading NDArrays

To start, let's show how you can save and load a list of NDArrays for future use. Note that while it's possible to use a general Python serialization package like ``Pickle``, it's not optimized for use with NDArrays and will be unnecessarily slow. We prefer to use ``ndarray.save`` and ``ndarray.load``.

```{.python .input  n=2}
X = nd.ones((100, 100))
Y = nd.zeros((100, 100))
import os
os.makedirs('checkpoints', exist_ok=True)
filename = "checkpoints/test1.params"
nd.save(filename, [X, Y])
```

It's just as easy to load a saved NDArray.

```{.python .input  n=3}
A, B = nd.load(filename)
print(A)
print(B)
```

We can also save a dictionary where the keys are strings and the values are NDArrays.

```{.python .input  n=4}
mydict = {"X": X, "Y": Y}
filename = "checkpoints/test2.params"
nd.save(filename, mydict)
```

```{.python .input  n=5}
C = nd.load(filename)
print(C)
```

## Saving and loading the parameters of ``gluon`` models

Recall from [our first look at the plumbing behind ``gluon`` blocks](P03.5-C01-plumbing.ipynb]) 
that ``gluon`` wraps the NDArrays corresponding to model parameters in ``Parameter`` objects. 
We'll often want to store and load an entire models parameters without 
having to individually extract or load the NDarrays from the Parameters via ParameterDicts in each block.

Fortunately, ``gluon`` blocks make our lives very easy by providing a ``.save_params()`` and ``.load_params()`` methods. To see them in work, let's just spin up a simple MLP.

```{.python .input  n=6}
num_hidden = 256
num_outputs = 1
net = gluon.nn.Sequential()
with net.name_scope():
    net.add(gluon.nn.Dense(num_hidden, activation="relu"))
    net.add(gluon.nn.Dense(num_hidden, activation="relu"))
    net.add(gluon.nn.Dense(num_outputs))
```

Now, let's initialize the parameters by attaching an initializer and actually passing in a datapoint to induce shape inference.

```{.python .input  n=7}
net.collect_params().initialize(mx.init.Normal(sigma=1.), ctx=ctx)
net(nd.ones((1, 100), ctx=ctx))
```

So this randomly initialized model maps a 100-dimensional vector of all ones to the number 381.35.
Let's save the parameters, instantiate a new network, load them in and make sure that we get the same result.

```{.python .input  n=8}
filename = "checkpoints/testnet.params"
net.save_params(filename)
net2 = gluon.nn.Sequential()
with net2.name_scope():
    net2.add(gluon.nn.Dense(num_hidden, activation="relu"))
    net2.add(gluon.nn.Dense(num_hidden, activation="relu"))
    net2.add(gluon.nn.Dense(num_outputs))
net2.load_params(filename, ctx=ctx)
net2(nd.ones((1, 100), ctx=ctx))
```

Great! Now we're ready to save our work. 
The practice of saving models is sometimes called *checkpointing*
and it's especially important for a number of reasons.
1. We can preserve and syndicate models that are trained once.
2. Some models perform best (as determined on validation data) at some epoch in the middle of training. If we checkpoint the model after each epoch, we can later select the best epoch.
3. We might want to ask questions about our trained model that we didn't think of when we first wrote the scripts for our experiments. Having the parameters lying around allows us to examine our past work without having to train from scratch.
4. Sometimes people might want to run our models who don't know how to execute training themselves or can't access a suitable dataset for training. Checkpointing gives us a way to share our work with others.

<!-- ## Serializing models themselves

[PLACEHOLDER] -->

## Next
[Convolutional neural networks from scratch](../chapter04_convolutional-neural-networks/cnn-scratch.ipynb)

For whinges or inquiries, [open an issue on  GitHub.](https://github.com/zackchase/mxnet-the-straight-dope)

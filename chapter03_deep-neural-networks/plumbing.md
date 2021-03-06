# Plumbing: A look under the hood of ``gluon``

In the previous tutorials, we taught you about linear regression and softmax regression. We explained how these models work in principle, showed you how to implement them from scratch, and presented a compact implementation using ``gluon``.
We explained *how to do things* in ``gluon`` but didn't really explain *how they work*. We relied on ``nn.Sequential``, syntactically convenient shorthand for ``nn.Block`` but didn't peek under the hood.  And while each notebook presented a working, trained model, we didn't show you how to inspect its parameters, save and load models, etc. In this chapter, we'll take a break from modeling to explore the gory details of ``mxnet.gluon``.

## Load up the data
First, let's get the preliminaries out of the way.

```{.python .input}
from __future__ import print_function
import mxnet as mx
import numpy as np
from mxnet import nd, autograd, gluon
from mxnet.gluon import nn, Block

###########################
#  Speficy the context we'll be using
###########################
ctx = mx.cpu()

###########################
#  Load up our dataset
###########################
batch_size = 64
def transform(data, label):
    return data.astype(np.float32)/255, label.astype(np.float32)
train_data = mx.gluon.data.DataLoader(mx.gluon.data.vision.MNIST(train=True, transform=transform),
                                      batch_size, shuffle=True)
test_data = mx.gluon.data.DataLoader(mx.gluon.data.vision.MNIST(train=False, transform=transform),
                                     batch_size, shuffle=False)
```

## Composing networks with ``gluon.Block``
Now you might remember that up until now, we've defined neural networks (for, example a multilayer perceptron) like this:

```{.python .input}
net1 = gluon.nn.Sequential()
with net1.name_scope():
    net1.add(gluon.nn.Dense(128, activation="relu"))
    net1.add(gluon.nn.Dense(64, activation="relu"))
    net1.add(gluon.nn.Dense(10))
```

This is a convenient shorthand 
that allows us to express a neural network compactly. 
When we want to build simple networks, 
this saves us a lot of time. 
But both (i) to understand how ``nn.Sequential`` works, 
and (ii) to compose more complex architectures, 
you'll want to understand ``gluon.Block``.

Let's take a look at the same model would be expressed with ``gluon.Block``.

```{.python .input}
class MLP(Block):
    def __init__(self, **kwargs):
        super(MLP, self).__init__(**kwargs)
        with self.name_scope():
            self.dense0 = nn.Dense(128)
            self.dense1 = nn.Dense(64)
            self.dense2 = nn.Dense(10)
        
    def forward(self, x):
        x = nd.relu(self.dense0(x))
        x = nd.relu(self.dense1(x))
        return self.dense2(x)
```

Now that we've defined a class for MLPs, we can go ahead and instantiate one:

```{.python .input}
net2 = MLP()
```

And initialize its parameters:

```{.python .input}
net2.initialize(ctx=ctx)
```

At this point we can pass data through the network by calling it like a function, just as we have in the previous tutorials.

```{.python .input}
for data, _ in train_data:
    data = data.as_in_context(ctx)
    break
net2(data[0:1])
```

## Calling ``Block`` as a function

Notice that ``MLP`` is a class and thus its instantiation, ``net2``, is an object. If you're a casual Python user, you might be surprised to see that we can *call an object as a function*. This is a syntactic convenience owing to Python's `__call__` method. Basically, ``gluon.Block.__call__(x)`` is defined so that ``net(data)`` behaves identically to ``net.forward(data)``. Since passing data through models is so fundamental and common, you'll be glad to save these 8 characters many times per day.

## So what is a ``Block``?

In ``gluon``, a ``Block`` is a generic component in a neural network. The entire network is a ``Block``, each layer is a ``Block``, and we can even have repeating sequences of layers that form an intermediate ``Block``.

This might sounds confusing, so let's make it simple. Each neural network has to do the following things:
1. Store parameters
2. Accept inputs
3. Produce outputs (the forward pass)
4. Take derivatives (the backward pass)

This can be said for the network as a whole, but it can also be said of each individual layer. A single fully-connected layer is parameterized by weight matrix and bias vector, produces outputs from inputs, and can given the derivative of some objective with respect to its outputs, can calculate the derivative with respect to its inputs. 

Fortunately, MXNet can take derivatives automatically. So we only have to define the forward pass (``forward(self, x)``). Then, using ``mxnet.autograd``, ``gluon`` can handle the backward pass. This is quite a powerful interface. For example we could define the forward pass for some component to take multiple inputs, and to combine them in arbitrary ways. We can even compose the ``forward()`` function such that it throws together a different architecture on the fly depending on some conditions that we could specify in Python. As long as the result is an NDArray, we're in the clear.  

## What's the deal with ``name_scope()``?
The next thing you might have noticed is that we added all of our layers inside a ``with net1.name_scope():`` block. This coerces ``gluon`` to give each parameter an appropriate name, indicating which model it belongs to, e.g. ``sequential8_dense2_weight``. Keeping these names straight makes our lives much easier once we start writing more complex code where we might be working with multiple models and saving and loading the parameters of each. It helps us to make sure that we associate each weight with the right model.

## Demystifying ``nn.Sequential``
So Sequential is basically a way of throwing together a Block on the fly. Let's revisit the ``Sequential`` version of our multilayer perceptron.

```{.python .input}
net1 = gluon.nn.Sequential()
with net1.name_scope():
    net1.add(gluon.nn.Dense(128, activation="relu"))
    net1.add(gluon.nn.Dense(64, activation="relu"))
    net1.add(gluon.nn.Dense(10))
```

In just 5 lines and 183 characters, we defined a multilayer perceptron with three fully-connected layers, each parametrized by weight matrix and bias term. We also specified the ReLU activation function for the hidden layers. 

Sequential itself subclasses ``Block`` and maintains a list of ``_children``. Then, every time we call ``net1.add(...)`` our net simply registers a new child. We can actually pass in an arbitrary ``Block``, even layers that we write ourselves. 

When we call ``forward`` on a ``Sequential``, it executes the following code:

```{.python .input}
def forward(self, x):
    for block in self._children:
        x = block(x)
    return x
```

Basically, it calls each child on the output of the previous one, returning the final output at the end of the chain.

## Shape inference
One of the first things you might notice is that for each layer, we only specified the number of nodes output, we never specified how many input nodes! You might wonder, how does ``gluon`` know that the first weight matrix should be $784 \times 128$ and not $42 \times 128$. In fact it doesn't. We can see this by accessing the network's parameters.

```{.python .input}
print(net1.collect_params())
```

Take a look at the shapes of the weight matrices: (128,0), (64, 0), (10, 0). What does it mean to have zero dimension in a matrix? This is ``gluon``'s way of marking that the shape of these matrices is not yet known. The shape will be inferred on the fly once the network is provided with some input.

So when we initialize our parameters, you might wonder, what precisely is happening?

```{.python .input}
net1.collect_params().initialize(mx.init.Xavier(magnitude=2.24), ctx=ctx)
```

In this situation, ``gluon`` is not actually initializing any parameters! Instead, it's making a note of which initializer to associate with each parameter, even though its shape is not yet known. The parameters are instantiated and the initializer is called once we provide the network with some input.

```{.python .input}
net1(data)
print(net1.collect_params())
```

This shape inference can be extremely useful at times. For example, when working with convnets, it can be quite a pain to calculate the shape of various hidden layers. It depends on both the kernel size, the number of filters, the stride, and the precise padding scheme used which can vary in subtle ways from library to library.

## Specifying shape manually

If we want to specify the shape manually, that's always an option. We accomplish this by using the ``in_units`` argument when adding each layer.

```{.python .input}
net2 = gluon.nn.Sequential()
with net2.name_scope():
    net2.add(gluon.nn.Dense(128, in_units=784, activation="relu"))
    net2.add(gluon.nn.Dense(64, in_units=128, activation="relu"))
    net2.add(gluon.nn.Dense(10, in_units=64))
```

Note that the parameters from this network can be initialized before we see any real data.

```{.python .input}
net2.collect_params().initialize(mx.init.Xavier(magnitude=2.24), ctx=ctx)
print(net2.collect_params())
```

## Next
[Writing custom layers with ``gluon.Block``](../chapter03_deep-neural-networks/custom-layer.ipynb)

For whinges or inquiries, [open an issue on  GitHub.](https://github.com/zackchase/mxnet-the-straight-dope)

---
layout: "page"
title: Adjusting Keras training loops (BETA VERSION)
subtitle: There is still something to correct
show-avatar: false
---

<script type="text/x-mathjax-config">
  MathJax.Hub.Config({
    extensions: ["tex2jax.js"],
    jax: ["input/TeX", "output/HTML-CSS"],
    tex2jax: {
      inlineMath: [ ['$','$'], ["\\(","\\)"] ],
      displayMath: [ ['$$','$$'], ["\\[","\\]"] ],
      processEscapes: true
    },
    "HTML-CSS": { availableFonts: ["TeX"] }
  });
</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>

In this mini project I would like to show how keras training loops can be adjusted for custom cases. Adjusting training loops can be handy if you would like to implement own layers for a neural network or an own loss function.

So, here I will try to partially implement a very nice idea of the paper [Neural Cleanse: Identifying and Mitigating Backdoor Attacks in Neural Networks](https://sites.cs.ucsb.edu/~bolunwang/assets/docs/backdoor-sp19.pdf). The prenseted implementation is based on [Bolun Wang](https://github.com/bolunwang/backdoor)'s implementation, Bolun Wang is one of the authors of the above paper.

The explanation of Keras' `get_update` and `function` functions is based on excelente [Javier](https://towardsdatascience.com/keras-custom-training-loop-59ce779d60fb)'s article.

In the first section, I will formally express the above idea. In the second part, I will explain and show how the idea can be then implemented. Lastly, I will give a summary.

### First section
Suppose a neural network $\mathcal{M}$ that classifies traffic signs, and a dataset $X = (x_1, \ldots, x_n)$, $Y = (y_1, \ldots, y_n)$ where $n$ represents number of samples, $x_i$ is a RGB image of a traffic sing of size $(32, 32, 3)$ and $y_i \in \[0, \ldots, 42\]$ is represents the class of $x_i$. So, we have in total $43$ different traffic signs.

Now, we would like to add some constant noise (also called _patern_) $p$ of size $(32, 32, 3)$ to an input $x_i$ so that our model $\mathcal{M}$ classifies the input with high probability as a predefined class. So, patrern $p$ causes that that all inputs are going to be likely classified into one class. For instance, we would that all samples should be classified as class $y = 33$

The questions are:
- how can we find such a pattern $p$?
- what do we mean by _add_ $p$ to an image? 

To answer the above questions, we need to introduce a mask $m$ of size $(32, 32, 3)$.

For adding the pattern to an image, define:

$$
    \begin{align}
        Trigger(x_i|~p,m) = x_i \cdot (1-m) + m \cdot p,
    \end{align}
$$

where all operations are defined as element-wise operations. The mask $m$ defines the intensity of the pattern $p$ in the image $x_i$.

To find a a pattern that does the job consider optimization:

$$
    \begin{align}
        \min_{p, m} ~ \sum_{i=1}^{n}\mathcal{L}(y, \mathcal{M}(Trigger(x_i|~p,m))),
    \end{align}
$$


where $\mathcal{M}(Trigger(x_i \vert ~p,m)) \in (0,1)^{43}$ and $\mathcal{L}$ is the classical cross entropy loss function.

### Second section
Now, to implement the above formulation and optimization we define own Keras layer class use Keras' `get_update` and `function` functions.

To implement own Keras layer I recommend [Keras Documentation](https://keras.io/layers/writing-your-own-keras-layers/). But basically, we have to overwrite `build` and `call` from the superclass `Layer`. 

The function `build` defines the weights mask and the patern that should be during training optimized. The function `call` defines the $Trigger$ function. The following code does the job.

```python
from tensorflow.keras.layers import Layer
import numpy as np
from  tensorflow.keras import backend as K

class Trigger(Layer):
  def __init__(self, **kwargs):
    super(Trigger, self).__init__(**kwargs)

  def build(self, input_shape):
    print(input_shape)
    self.mask1 = np.zeros(input_shape[1:])
    self.mask1 = K.variable(self.mask1, name = "mask")
    
    self.pattern = np.zeros(input_shape[1:])
    self.pattern = K.variable(self.pattern, name = "pattern")
    super(Trigger,self).build(input_shape)

  def call(self, x):
    #return x+self.mask1 
    return x*(K.ones_like(self.mask1)- self.mask1) + self.pattern*self.mask1
```

Now, we need to add the trigger layer to our model $\mathcal{m}$. Suppose $\mathcal{m}$ is stored in Python as a sequential Keras model `model_m`. 

```python
from src.Trigger_layer import Trigger
from tensorflow.keras.models import Sequential
trigger = Sequential()
trigger_layer = Trigger(input_shape=(32,32,3))

trigger.add(trigger_layer)

model_m2 = tf.keras.Sequential([
  trigger,
  model_m
])
```
So, `model_m2` represents the composition of the original model and the trigger, i.e. $\mathcal{M}(Trigger(x_i|~p,m))$.

The other task is to implement using Keras the optimization defined in the _first section_.

Firstly, define placeholders for the training data:
```python
input_tensor = K.placeholder(model_m2.input_shape)
y_true_tensor = K.placeholder(model_m2.output_shape)
```
`input_tensor` is here take samples $x_i$ and `y_true_tensor` is to take always one predefined class, e.g. $y = 33$.

Secondly, define the loss function:
```python
output_tensor = model_m2(input_tensor)

loss_acc = categorical_accuracy(output_tensor, y_true_tensor)
loss_ce = categorical_crossentropy(output_tensor, y_true_tensor)

loss = loss_ce
```

Great, we are almost there! So far, we defined individual operations that need to be done to run the optimization. Now, we put all the operations into a single computational graph:

```python
optimizer = Adam()
updates = optimizer.get_updates(params = [model_m2.trainable_weights[0], model_m2.trainable_weights[1]] , loss = loss)
```

To be honest, we instantiated two graphs:
1. Graph1 with input = [x, y_true], output = [loss]. This graph represents the so called forward pass of our network `model_m2`.

2. Graph2 with input = [loss, weights], output = [updated weights]. In this graph the backward pass is performed.

To run the two graphs we use `function`:
```python
train = K.function(inputs = [input_tensor, y_true_tensor], outputs = [loss, loss_acc], updates = updates)
```
Now, `train` is a function that performs forward pass given `input_tensor` and `y_true_tensor`, it outputs loss and accuracy. Further, the backward pass is also performed updating the mask and pattern stored in `model_m2.trainable_weights[0]`, `model_m2.trainable_weights[1]`.

Finally we can start our optimization procedure:
```python
#epochs = 3
epochs = 10000

# the pattern should cause that all samples are classified as y_target
y_target = 33
loss = [] 
acc = []
for epoch in range(epochs):
  Y_target = to_categorical([y_target]*BATCH_SIZE, NUM_CLASSES)
  X_batch, _ = data_generator.next()
  if X_batch.shape[0] != Y_target.shape[0]:
    Y_target = to_categorical([y_target]*X_batch.shape[0], NUM_CLASSES)

  losses, acces = train([X_batch, Y_target])
  acc.append(np.mean(acces))
  loss.append(np.mean(losses))
```

After running the above optimization procedure we could get:

<p align="center">
<img src="/img/acc_loss.JPG" alt="geo" width="500" height="500"/>
</p>

So, our optimization procedure was successfull. This means all traffic signs from the training data are going be with high probability classified as class 33 if the trigger is added.

In the following we can see possible images and the pattern and images with the added pattern:

<p align="center">
<img src="/img/possible_output.JPG" alt="geo" width="500" height="500"/>
</p>

If you are interested for the complete code, then check out my [github repo](https://github.com/CingisAlexander/trigger_generation).

I hope, I was able to explain you the power and simplicity of custom Keras training loops. And you will be able to apply it to your own projects!
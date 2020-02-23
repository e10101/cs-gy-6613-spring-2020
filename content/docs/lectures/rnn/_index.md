---
title: Lecture 4b - Sequences and Recurrent Neural Networks (RNN)
draft: true
weight: 70
---

# Sequences and Recurrent Neural Networks (RNN)

## Sequences

Data streams are everywhere in various applications. For example, weather station sensor data arrive in streams indexed by time,  in financial trading data - one can think of many others. We are interested to fit sequenced data with a model and in order to do so we need a hypothesis set that we can draw from that is rich enough for the task at hand. 

_Dynamical systems_ are such rich models. In the following we use $t$ as the index variable and the notation $s_{1:\tau}$ means the sequence from 1 to $\tau$, without this implying any time semantics for the index $t$. In a dynamical system,  its _recurrent_ state evolution can be represented as:

$$\mathbf{s}\_t = f_t(\mathbf{s}\_{t-1}, \mathbf{a}_t ; \bm \theta_t)$$

where $\bm s$ is the evolving state,  $\bm a$ is an external action or control and $\bm \theta$ is a set of parameters that specify the state evolution model $f$.  This innocent looking equation can capture quite a lot of complexity. 

1.  The _state space_ which is the set of states can depend on $t$. 
2.  The _action space_ similarly can depend on $t$
3.  Finally, the function that maps previous states and actions to a new state can also depend on $t$

So the dynamical system above has indeed offer a very profound modeling flexibility. 

## RNN Architecture
The RNN architecture is a constrained implementation of the above dynamical system 

$$\mathbf{h}\_t = f(\mathbf{h}\_{t-1}, \mathbf{x}_t ; \bm \theta)$$

RNNs implement the _same_ function (parametrized by $\bm \theta$) across the sequence $1:\tau$. Effectively there is no dependency on $t$ of the parameters $\bm \theta$ and this means that the network _shares_ parameters across the sequence. We have seen parameter sharing in CNNs as well but if you recall the sharing was over the relatively small span of the filter. But the most striking difference between CNNs and RNNs is in recursion itself. The state is latent and is denoted with $\bm h$ to match the notation we used earlier for the hidden layers. 

![rnn-recurrence](images/rnn-recurrence.png#center)
*Recursive state representation in RNNs*

The weights $\bm h$ in CNNs were not a function of previous weights and this means that they cannot remember previous hidden states in the classification or regression task they try to solve. This is perhaps the most distinguishing element of the RNN architecture - its ability to remember via the hidden state who is dimensioned according to the task at hand. There is a way using sliding windows to allow DNNs to remember past inputs as shown in the figure below for an NLP application. 

![dnn-sequential-processing](images/dnn-sequential-processing.png#center)
*DNNs can create models from sequential data (such as the language modeling use case shown here). At each step $t$ the network with a sliding window span of $\tau=3$ that acts as memory, will concatenate the word embeddings and use a hidden layer $\bm h$ to predict the the next element in the sequence.  However, notice that (a) the span is limited and fixed (b) words such as "the ground" will appear in multiple sliding windows forcing the network to learn two different patterns for this constituent ("in the ground", "the ground there").*

There are many RNN architectures and in this course will suffice to go over just a handful to understand what they offer in terms of their representational capacity. One significant factor that separates the architectures is the way they perform the hidden state calculation at each $t$. This is shown in the next figure.


![hidden-state-types](images/hidden-state-types.png#center)
*Differentiating Architectures (a) DNN, (b) Simple RNN, (c) LTSM, (d) GRU*

### Simple RNN 
![rnn-hidden-recurrence](images/rnn-hidden-recurrence.png#center)

*Simple RNN with recurrences between hidden units. This architecture can compute any computable function and therefore is a [Universal Turing Machine](http://alvyray.com/CreativeCommons/BizCardUniversalTuringMachine_v2.3.pdf). Your laptops and smartphones are descendants of UTM.* 

Notice how the path from input $\bm x_{t-1}$ affects the label $\bm y_{t}$ and also the conditional independence between $\bm y$ given $\bm x$. Please note that this is not a computational graph rather one way to represent the hidden state transfer between recurrences.

#### Forward Propagation 

This network maps the input sequence to a sequence of the same length and implements the following forward pass:

$$\bm a_t = \bm W \bm h _{t-1} + \bm U \bm x_t + \bm b$$

$$\bm h_t = \tanh(\bm a_t)$$

$$\bm o_t = \bm V \bm h_t + \bm c$$

$$\hat \bm y_t = \mathtt{softmax}(\bm o_t)$$

$$L(\bm x_1, \dots , \bm x_{\tau}, \bm y_1, \dots , \bm y_{\tau}) = D_{KL}[\hat p_{data}(\bm y | \bm x) || p_{model}(\bm y | \bm x; \bm w)]$$

$$= - E_{\bm y | \bm x ≋ \hat{p}_{data}} \log p_{model}(\bm y | \bm x ; \bm w)  = - \sum_t \log p_{model}(y_t | \bm x_1, \dots, \bm x_t ; \bm w)$$ 

Notice that RNNs can model very generic distributions  $\log p_{model}(\bm x, \bm y ; \bm w)$. The simple RNN architecture above, effectively models the posterior distribution $\log p_{model}(\bm y | \bm x ; \bm w)$  and based on a conditional independence assumption it factorizes into $\sum_t \log p_{model}(y_t | \bm x_1, \dots, \bm x_t ; \bm w)$. 

Note that by connecting the $\bm y_{t-1}$ to $\bm h_t$ via a matrix e.g. $\bm R$ we can avoid this simplifying assumption and be able to model an arbitrary distribution $\log p_{model}(\bm y | \bm x ; \bm w)$. In other words just like in the other DNN architectures, connectivity directly affects the representational capacity of the hypothesis set. 

In many instances we have problems where it only matters the label $y_\tau$ at the end of the sequence. Lets say that you are classifying spoken words or video inside the cabin of a car to detect the psychological state of the driver. The same architecture shown above can also represent such problems - the only difference is the only the $\bm o_\tau$, $L_\tau$ and $y_\tau$ will be considered. 

Lets see an example to understand better the forward propagation equations.

![example-sentence](images/example-sentence.png#center)
*Example sentence as input to the RNN*

In the figure above you have a hypothetical document (a sentence) that is broken into what in natural language processing called _tokens_. Lets say that a token is a word in this case. In the simpler case where we need a classification of the whole document, given that $\tau=6$, we are going to receive at t=1, the first token $\bm x_1$ and with an input hidden state  $\bm h_0 = 0$ we will calculate the forward equations for $\bm h_1$, ignoring the output $\bm o_1$ and repeat the unrolling when the next input $\bm x_2$ comes in until we reach the end of sentence token $\bm x_6$ which in this case will calculate the output and loss 

$$- \log p_{model} (y_6|\bm x_1, \dots , \bm x_6; \bm  w)$$ 

where $\bm w = \\{ \bm W, \bm U, \bm V, \bm b, \bm c \\}$. 


#### Back-Propagation Through Time (BPTT)
Lets now see how the backward propagation would work. 

![rnn-BPTT](images/rnn-BPTT.png#center)
*Understanding RNN memory through BPTT procedure*

Backpropagation is similar to that of feed-forward (FF) networks simply because the unrolled architecture resembles a FF one. But there is an important difference and we explain this using the above computational graph for the unrolled recurrences $t$ and $t-1$. During computation of the variable $\bm h_t$ we use the value of the variable $\bm h_{t-1}$ calculated in the previous recurrence. So when we apply the chain rule in the backward phase of BP, for all nodes that involve the such variables with recurrent dependencies, the end result is that _non local_ gradients from previous backpropagation steps ($t$ in the figure) appear. This is effectively why we say that simple RNNs feature _memory_. This is in contrast to the FF network case where during BP only local to each gate gradients where involved as we have seen in the the [DNN chapter]({{<ref "../dnn/backprop-dnn">}}). 

The key point to notice in the backpropagation in recurrence $t-1$ is the junction between $\tanh$ and $\bm V \bm h_{t-1}$. This junction brings in the gradient $\nabla_{\bm h_{t-1}}L_t$ from the backpropagation of the $\bm W h_{t-1}$ node in recurrence $t$ and just because its a junction, it is added to the backpropagated gradient from above in the current recurrence $t-1$ i.e.

$$\nabla_{\bm h_{t-1}}L_{t-1} += \nabla_{\bm h_{t-1}}L_t $$ 

Ian Goodfellow's book section 10.2.2 provides the exact equations - please note that you need to know the intuition behind computational graphs for RNNs. 

As you can understand the presence of such paths that connect the hidden states extends all the way from the $t=\tau$ to $t=1$ making the gradient, when $\tau$ is large, smaller and smaller. This is called the _vanishing gradient_ problem and is discussed in section 10.7 of Ian Goodfellow's book.  

### The Long Short-Term Memory (LSTM) Architecture


### GRU

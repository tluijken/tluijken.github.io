---
title: Creating a neural network
author: Thomas Luijken
date: 2020-11-29 14:10:00 +0200
categories: [Machine Learning, AI, Neural Network]
tags: [neutal network]
hidden: true;
---

## Introduction
These days, Machine Learning and Artificial Intelligence are playing a major role in our software. Big tech companies like Microsoft and Google made API's available to us, which allows us to easily embed machine learning into or software, without diving deep into the nerdy stuff. But, there are developers like me, who like the nerdy stuff. I want to know, in the basics how this stuff works. Start small, and understand the process of machine learning, rather than just consuming a public API and shouting out loud 'we use AI!' as a commercial trademark.

![Image](/assets/img/neural-network/neural_network_brain.png)

In this article I'll explain how to build a neural network from scratch, and configure it to perform a simple XOR like operation. I won't go into too much detail on the types of machine learning, but I'll provide resources as much as possible. There is so much information regarding this on the internet, written by people who have more knowledge regarding this topic than I have.

### Short introduction to neural networks
Machine learning can be done using neural networks. A neural network in software can be seen as the neural network in our brain. Our brain consists of [**neurons**](https://en.wikipedia.org/wiki/Neuron), which are connected through [**Synapsys**](https://en.wikipedia.org/wiki/Synapse). A group of connected neurons can be called a [**neural network**](https://en.wikipedia.org/wiki/Neural_network).

Whenever a [**pulse**](https://en.wikipedia.org/wiki/Action_potential) travels through the neural network, it's value is altered by the weight of each neuron. The signal altered by all the various neuron weights, will lead to the processed output result. [**This article**](https://www.explainthatstuff.com/introduction-to-neural-networks.html) explains the process pretty simple. The weight of each neuron can also be influenced by a [**bias**](https://www.baeldung.com/cs/neural-networks-bias), changing the processed result in the output nodes.

### Types of machine learning
Before we focus on artificial neural networks, it's important to state that there are 3 main types of machine learning.
1. **Supervised learning**, which is commonly used for classification. We have a predefined set of input values, and a predefined set of expected output results. We'll feed the network with the specified inputs, and check if the output that is produced, matches the expected ouput. Usually, 80% of the dataset is used to train the network, and the other 20% is used to validate the trained network.
2. **Unsupervised learning**, which is commonly used for finding unknown patterns throughout a dataset. There is no expected output. We'll just keep the network processing the dataset untill it finds patterns.
3. **Hybrid learning**, which is a combination of the 2 above. The network is partially trained, to perform unsupervised learning after the supervised learning process.

*This article will focus on **Supervised learning** from now on.*

### Artificial neural networks
Artificial neural networks work in pretty much the same way as our biological neural network. A neural network consist of many neurons, which are grouped into layers. A neural network at least has 1 input layer, 1 hidden layer, and 1 output layer. Depending on the complexity of the dataset we want it to learn, more hidden layers can be added.

![neuralnetwork](/assets/img/neural-network/1_YgJ6SYO7byjfCmt5uV0PmA.png)

The number of nodes (neurons) in the input layer, is usually defined by the number of input variables we have in our dataset. The number of output nodes is defined by the number of outputs we're expecting.

We have input nodes, which will take the normalized inputs. Let's say, we have 4 charactaristics:
1. Age
2. Gender
3. Income
4. Religion

Based on these 4 charactaristics, we want to determine which political party each person is most likely to choose during the elections.

We'll feed our neural network with a pre-defined set of input values, which will be pulsed through the neural network's hidden layer(s). When the pulse reaches the output nodes, we will check if the output value is correct, or not. The bigger our dataset is, the better our network can be trained.

Our dataset can look something like this:

AGE | INCOME  | GENDER | RELIGION   | POLITICAL PARTY
--- | ------- | ------ | ---------- | ---------------
31  | 51.000  | Male   | Catholic   | Republican
54  | 64.000  | Male   | Atheist    | Democratic
48  | 43.000  | Male   | Protestant | Democratic
24  | 24.000  | Male   | Catholic   | Republican
35  | 46.000  | Male   | Protestant | Republican
66  | 100.000 | Male   | Protestant | Democratic

etc......

The political party will be the output node in this case.
So our network will have one input layer, containing 4 input nodes, and 1 output layer, containing 1 output node.

#### Back propagation
When training our network, we will validate the value in the output node with the expected outcome. When it's not correct, we will (randomly) alter the weights of the nodes in the hidden layers, and try to run the same process again untill this is correct. We could do this totally at random, untill we have the correct set of weights, to produce the expected output. However, this process can be done a bit more sophisticated. With **back propagation** we'll tell the network *how far* the output was off. If the expected outcome is far off, we'll adjust the weights with bigger numbers. If the output is sligtly off, we'll adjust the weights with smaller numbers. This way, once the network is going in the right direction, it will refine it's training process each time with smaller steps.

## What are we're going to create?



## Show me the code!
The code I mentioned above can be found in [**this github repository**](https://github.com/tluijken/neural-network-demo)

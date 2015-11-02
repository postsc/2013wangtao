## Backpropagation算法的Python实现

标签 ： 算法

``` Python
#! /usr/bin/env python
#-*-coding: utf-8-*-#

### Note: This is a 3-layer perceptron, including input layer, hidden layer and
### 	output layer.

### Author: Lor (a.k.a ButcherCat);
### Date: 2014-10-23;
### E-mail: wt_lor@163.com (Maybe ButcherCat@lortech.com);
### Location: Room 403, Building 9, Shandong University of Technology, Zibo, Shandong

import math
import random

set_train = []
input_neurons = int(raw_input("Enter neuron number of input layer:->"))
hidden_neurons = int(raw_input("Enter neuron number of hidden layer:->"))
output_neurons = int(raw_input("Enter neuron number of output layer:->"))
elemnum = int(raw_input("Enter the number of element in training set:->"))

## Initialize the real output vector 
output = []
for i in range(0,output_neurons):
	output.append(0);

## Initialize weight vector w between hidden layer and output layer
## The value at the subscript 0 of w is the bias
## The weight of bias is included in this vector(between hidden layer and output layer).
w = []
for i in range(0,output_neurons):
	temp = []
	for j in range(0,hidden_neurons+1):
		temp.append(random.random() / 10)
	w.append(temp)

## Initialize weight vector v between input layer and hidden layer
## The value at the subscript 0 of v is the bias
## The weight of bias is included in this vector(between input layer and hidden layer).
v = []
for i in range(0,hidden_neurons):
	temp = []
	for j in range(0,input_neurons+1):
		temp.append(random.random() / 10)
	v.append(temp)

## Initialize [l]earing [f]actor
lf = float(raw_input("Enter the learning factor yita(0.0 ~ 1.0):->"))

## Get data from traning set and save them to local structure
for item in range(0,elemnum):
	print "\tNo. %d element :->" % (item + 1)
	inlist = []
	outlist = []
	inlist.append(-1.0)
##	outlist.append(-1.0)
	print "\t\tEnter input part of element in training set:"
	for inloop in range(0,input_neurons):
		print "The No. %d input" %(inloop+1)	
		inlist.append(float(raw_input()))
	print "\t\tEnter output part of element in training set:"
	for outloop in range(0,output_neurons):
		print "The No. %d output" %(outloop+1)
		outlist.append(float(raw_input()))
	bp_elem = {"Input":inlist, "Output_d":outlist, "Output_r":output, "Flag":0}
	set_train.append(bp_elem)

## Define sigmoid function to calculate the value of Sigmoid
def sigmoid(x):
	ret = 1 / (1 + math.exp(-x))
	return ret

## Function deri_sigmoid() is the derivative of Sigmoid function.
def deri_sigmoid(x):
	ret = (1 - sigmoid(x)) * sigmoid(x)
	return ret

## Define a function called cal_net() to calculate the net.
## Note: The 2nd dimension of weight_v should share the same
## length with that of input_v.
def cal_net(weight_v,input_v):
	r = []
	for item in weight_v:
		s = 0
		for i in range(0,len(input_v)):
			s += item[i] * input_v[i]
		r.append(s)
	return r

cond = True
precision = 0.1
while cond:
	error = 0
	for count in set_train:
		net_hlayer = cal_net(v,count["Input"])	## List net_hlayer means net of hidden layer
		houtput = []	## List houtput means output of hidden layer.
		houtput.append(-1.0)
		## Calcualte the output of hidden layer
		for item in net_hlayer:
			tmp = sigmoid(item)
			houtput.append(tmp)
		net_olayer = cal_net(w,houtput)
		## Calcuate the output of output layer and save them to the List output
		for item in net_olayer:
			tmp = sigmoid(item)
			count["Output_r"][net_olayer.index(item)] = tmp
		## Correction of weight vector between hidden layer and output layer
		for item in w:
			for i in range(0,len(item)):
				delta_k = (count["Output_d"][i] - count["Output_r"][i]) * count["Output_r"][i] * (1-count["Output_r"][i])
				item[i] = lf * delta_k * houtput[i]
		for item in v:
			for i in range(0,len(item)):
				s = 0
				for j in range(0,output_neurons):
					for k in range(0,hidden_neurons+1):
						s += (w[j][k] / (lf * houtput[k])) * w[j][k]
				delta_h = s * houtput[k] * (1 - houtput[k])
				item[i] += lf * delta_h * count["Input"][i]
		
	if error < precision:
		cond = False
print "Bias between input layer and hidden layaer:->"
for item in v:
        print item[0]
print "Bias between hidden layer and output layer:->"
for item in w:
        print item[0]

print "Weight vector between input layer and hidden layer:->"
for item in v:
        temp = []
        for i in range(1,len(item)):
                temp.append(item[i])
        print temp
print "Weight vector between hidden layer and output layer:->"
for item in w:
        temp = []
        for i in range(1,len(item)):
                temp.append(item[i])
        print temp

```





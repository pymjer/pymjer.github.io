---
title:      TensorFlow中的函数
layout:     post
subtitle:   "TensorFlow中的函数记录"
date:       2016-09-22
author:     "Pymjer"
header-img: "img/post-bg-unix-linux.jpg"
tags: ["机器学习"]
---

TensorFlow1.0升级的函数
======================
- tf.pack => tf.stack
- tf.concat(concat_dim, value) => tf.concat(value,concat_dim)

tf.reduce_sum 函数
===================
对tensor求和

	For example:
	# 'x' is [[1, 1, 1]
	#         [1, 1, 1]]
	tf.reduce_sum(x) ==> 6
	tf.reduce_sum(x, 0) ==> [2, 2, 2]
	tf.reduce_sum(x, 1) ==> [3, 3]
	tf.reduce_sum(x, 1, keep_dims=True) ==> [[3], [3]]
	tf.reduce_sum(x, [0, 1]) ==> 6

tf.concat(concat_dim, value, name = 'concat')
==================
解释：这个函数的作用是沿着concat_dim维度，去重新串联value，组成一个新的tensor。

使用例子：

	#!/usr/bin/env python
	# -*- coding: utf-8 -*-

	import tensorflow as tf 
	import numpy as np 

	sess = tf.Session()
	t1 = tf.constant([[1, 2, 3], [4, 5, 6]])
	t2 = tf.constant([[7, 8, 9], [10, 11, 12]])
	d1 = tf.concat(0, [t1, t2])
	d2 = tf.concat(1, [t1, t2])
	print sess.run(d1)
	print sess.run(tf.shape(d1))
	print sess.run(d2)
	print sess.run(tf.shape(d2))

	# output
	[[ 1  2  3]
	 [ 4  5  6]
	 [ 7  8  9]
	 [10 11 12]]

	[[ 1  2  3  7  8  9]
	 [ 4  5  6 10 11 12]]

	# tips
	从直观上来看，我们取的concat_dim的那一维的元素个数肯定会增加。比如，上述例子中的d1的第0维增加了，而且d1.shape[0] = t1.shape[0]+t2.shape[0]。

输入参数：

- concat_dim: 一个零维度的Tensor，数据类型是int32。
- values: 一个Tensor列表，或者一个单独的Tensor。
- name:（可选）为这个操作取一个名字。

输出参数：一个重新串联之后的Tensor。

> 在TF1.0 后，该函数的api变了，concat_dim是第二个参数

tf.argmax(input, axis=None, name=None, dimension=None)
======================
返回tensor中最大值的索引
定义中的axis与numpy中的axis是一致的，下面通过代码进行解释

	import numpy as np
	import tensorflow as tf

	sess = tf.session()
	m = sess.run(tf.truncated_normal((5,10), stddev = 0.1) )
	print type(m)
	print m
	# m是一个5行10列的矩阵，类型为numpy.ndarray
	#使用tensorflow中的tf.argmax()
	col_max = sess.run(tf.argmax(m, 0) )  #当axis=0时返回每一列的最大值的位置索引print col_max

	row_max = sess.run(tf.argmax(m, 1) )  #当axis=1时返回每一行中的最大值的位置索引print row_max

	array([2, 3, 0, 3, 0, 0, 0, 0, 3, 4])
	array([5, 0, 0, 8, 9])

	#使用numpy中的numpy.argmax
	row_max = m.argmax(0)
	print row_max

	col_max = m.argmax(1)
	print col_max

	array([2, 3, 0, 3, 0, 0, 0, 0, 3, 4])
	array([5, 0, 0, 8, 9])

可以看到tf.argmax()与numpy.argmax()方法的用法是一致的

	* axis = 0的时候返回每一列最大值的位置索引
	* axis = 1的时候返回每一行最大值的位置索引
	* axis = 2、3、4...，即为多维张量时，同理推断

tf.truncated_normal(shape, mean=0.0, stddev=1.0, dtype=tf.float32, seed=None, name=None)
=====================
从一个正态分布片段中输出随机数值,生成的值会遵循一个指定了平均值和标准差的正态分布，只保留两个标准差以内的值，超出的值会被弃掉重新生成。

	参数：
	shape: 一个一维整数张量 或 一个Python数组。 这个值决定输出张量的形状。
	mean: 一个零维张量或 类型属于dtype的Python值. 这个值决定正态分布片段的平均值
	stddev: 一个零维张量或 类型属于dtype的Python值. 这个值决定正态分布片段的标准差。
	dtype: 输出的类型.
	seed: 一个Python整数. 被用来为正态分布创建一个随机种子. 详情可见set_random_seed for behavior.
	name: 操作的名字 (可选参数).


tf.expand_dims(input, dim, name = None)
=====================
解释：这个函数的作用是向input中插入维度是1的张量。
我们可以指定插入的位置dim，dim的索引从0开始，dim的值也可以是负数，从尾部开始插入，符合 python 的语法。
这个操作是非常有用的。举个例子，如果你有一张图片，数据维度是[height, width, channels]，你想要加入“批量”这个信息，那么你可以这样操作expand_dims(images, 0)，那么该图片的维度就变成了[1, height, width, channels]。
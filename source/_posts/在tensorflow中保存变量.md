---
title: 在tensorflow中保存变量
date: 2018-06-19 17:11:05
tags:[tensorflow,tf.train.Saver,Attempting to use uninitialized value]
categories:
- tensorflow
---

在TensorFlow中保存变量是训练必须的一个步骤，也十分简单。参考官方文档

保存变量代码

```python
# Create some variables.
v1 = tf.get_variable("v1", shape=[3], initializer = tf.zeros_initializer)
v2 = tf.get_variable("v2", shape=[5], initializer = tf.zeros_initializer)

inc_v1 = v1.assign(v1+1)
dec_v2 = v2.assign(v2-1)

# Add an op to initialize the variables.
init_op = tf.global_variables_initializer()

# Add ops to save and restore all the variables.
saver = tf.train.Saver()

# Later, launch the model, initialize the variables, do some work, and save the
# variables to disk.
with tf.Session() as sess:
  sess.run(init_op)
  # Do some work with the model.
  inc_v1.op.run()
  dec_v2.op.run()
  # Save the variables to disk.
  save_path = saver.save(sess, "/tmp/model.ckpt")
  print("Model saved in path: %s" % save_path)
```

读取代码

```python
tf.reset_default_graph()


# Create some variables.
v1 = tf.get_variable("v1", shape=[3])
v2 = tf.get_variable("v2", shape=[5])

# Add ops to save and restore all the variables.
saver = tf.train.Saver()

# Later, launch the model, use the saver to restore variables from disk, and
# do some work with the model.
with tf.Session() as sess:
  # Restore variables from disk.
  saver.restore(sess, "/tmp/model.ckpt")
  print("Model restored.")
  # Check the values of the variables
  print("v1 : %s" % v1.eval())
  print("v2 : %s" % v2.eval())
```

简单易用，但是我们在使用的过程中出现了“Attempting to use uninitialized value”的错误，经排查发现，我们使用的某些变量定义在函数中，而在声明saver之前并没有调用上述函数，所以就出现了以上错误，我们可以将读取代码中第9行的代码插入5行和6行之间，就会发现`v2.eval()`无法执行了，这是因为在声明saver时它会从当前的图中寻找需要恢复的变量，在其之后声明的变量将不会包含在这里面，使用的时候要格外注意。
---
title: 使用训练好的word2vec词典时需要注意的问题
date: 2019-01-16 18:20:11
tags: [TensorFlow,word2vec,larger than 2GB]
categories:
- TensorFlow
---

在模型中使用训练好的word2vec向量时，我们通常使用一下代码

```python
self.reader = tf.train.NewCheckpointReader(self.embedding_cpt_path)
self.embedding_W = self.reader.get_tensor('w_in') #w_in是训练好的词向量变量
```

但是当该词向量字典过大时，我们就会遇到以下错误

```
ValueError: Cannot create a tensor proto whose content is larger than 2GB
```

这是因为除变量外，像常量等很多都不允许超过2GB（因为要序列化到pb），所以当你要加载一个2GB以上的数据时，将它声明为变量。

```python
self.embedding_W = tf.Variable(tf.constant(0.0, shape=[word_size, embedding_size]),
                            trainable=False, name="e_w")
self.embedding_saver = tf.train.Saver({"w_in": self.embedding_W})
sess = tf.Session()
self.embedding_saver.restore(sess, "checkpoint_filename.ckpt")  #在全局初始化后进行
```

###### 参考资料

- https://stackoverflow.com/questions/35687678/using-a-pre-trained-word-embedding-word2vec-or-glove-in-tensorflow
- https://stackoverflow.com/questions/51244489/valueerror-cannot-create-a-tensor-proto-whose-content-is-larger-than-2gb


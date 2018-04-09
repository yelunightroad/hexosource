---
title: TensorFlow中tfRecord的使用以及测试
date: 2018-04-09 17:57:45
tags: [TensorFlow,tfRecord,tf.contrib.learn.run_n]
categories:
- TensorFlow
---

tfRecord是TensorFlow官方推荐的标准数据格式，对于存储字典类型的数据比较友好，所以撰写此文记录一下使用过程中的一点经验。

## tfRecord结构

tfRecord中存储的记录由tf.train.Example定义，其实就是一个protocol buffer文件，定义如下：

```protobuf
message Example {
 Features features = 1;
};

message Features{
 map<string,Feature> featrue = 1;
};

message Feature{
    oneof kind{
        BytesList bytes_list = 1;
        FloatList float_list = 2;
        Int64List int64_list = 3;
    }
};
```

可以看到其主要包含了一个字符串格式的key，以及字符串、浮点数或者整数类型的列表构成的value

##  代码示例

```python
import tensorflow as tf
import os
keys=[[1.0,2.0],[2.0,3.0],[1,2]]
keys2=[[1.0,2.0],[2.0,3.0],[3.0]]

strtest='aaa'
sess=tf.InteractiveSession()
sess.run(tf.global_variables_initializer())
```

基础变量，没什么可解释的

```python
def make_example(key):
    example = tf.train.Example(features=tf.train.Features(
        feature={
            'ft':tf.train.Feature(float_list=tf.train.FloatList(value=key)),
            'st':tf.train.Feature(bytes_list=tf.train.BytesList(value=[strtest]))
        }
    ))
    return example

filename="tmp.tfrecords"
if os.path.exists(filename):
    os.remove(filename)
writer = tf.python_io.TFRecordWriter(filename)
for key in keys:
    ex = make_example(key)
    writer.write(ex.SerializeToString())
writer.close()

reader = tf.TFRecordReader()
filename_queue = tf.train.string_input_producer(["tmp.tfrecords"],num_epochs=2)
_,serialized_example =reader.read(filename_queue)

batch = tf.train.batch(tensors=[serialized_example],batch_size=3)
```

在以上代码中我们展示了如何定义一个example并将其写入文件，同事19行开始我们也对如何读取文件做了简单示例，现在读取部分也可以使用dataset，更加符合官方潮流，例如用一下代码代替19-23行

```python
dataset = tf.data.TFRecordDataset(filenames=filenames, buffer_size=200).batch(3)
record_iter = dataset.make_one_shot_iterator()
```

下面进入重头戏，如何解析tfRecord的内容

```python
features={
    "ft":tf.VarLenFeature(tf.float32),
    "st":tf.VarLenFeature(tf.string)
}
key_parsed = tf.parse_example(batch,features)

sess.run(tf.initialize_local_variables())
print tf.contrib.learn.run_n(key_parsed) 
#tf.train.get_or_create_global_step()
#print tf.contrib.training.train(key_parsed,'../')
```

输出如下

```
[{'ft': SparseTensorValue(indices=array([[0, 0],
       [0, 1],
       [1, 0],
       [1, 1],
       [2, 0],
       [2, 1]]), values=array([1., 2., 2., 3., 1., 2.], dtype=float32), dense_shape=array([3, 2])), 'st': SparseTensorValue(indices=array([[0, 0],
       [1, 0],
       [2, 0]]), values=array(['aaa', 'aaa', 'aaa'], dtype=object), dense_shape=array([3, 1]))}]
```



其实非常简单，定义一个同样的feature，然后用相应的方法获取就可以了，注意代码的第8行的方法已经废弃了，TensorFlow相关人员表示没有开发其替代函数的打算，想实现相应的效果可以使用第9、10行的代码，基本可以替代，但是做实验的话由于train方法其实是在训练，还是8行的函数简单些，在删除前我们暂且继续使用。

在以上代码中我们使用的是VarlenFeature，它更通用一些，另外比较常见的还有FixedLenFeature，使用代码如下

```
features={
    "ft":tf.FixedLenFeature(shape=[2],dtype=tf.float32,default_value=[2.0,3.0])
}

key_parsed = tf.parse_example(batch,features)

print tf.contrib.learn.run_n(key_parsed)
```

输出结果为

```
[{'ft': array([[1., 2.],
       [2., 3.],
       [1., 2.]], dtype=float32)}]
```

这里需要注意的是它解析的数据必须是定长的，且数据格式与shape参数必须一致，否则就会报错。另外它返回的是一个tensor不同于VarLenFeature返回的是sparseTensor。这里default_value的意义是你查询的字段不存在时赋予默认值，比如你现在查询的是_ft_字段，如果存储时只存储了_st_那么就会使用默认值赋值，所以default_value，的shape必须与shape参数一致。

有了以上两个参数基本可以解决我们的数据读写问题，如果还不够，可以了解一下SparseFeature，它可以以一种类似SparseTensor的方式帮你定义数据

```
`serialized`:
  [
    features {
      feature { key: "val" value { float_list { value: [ 0.5, -1.0 ] } } }
      feature { key: "ix" value { int64_list { value: [ 3, 20 ] } } }
    },
    features {
      feature { key: "val" value { float_list { value: [ 0.0 ] } } }
      feature { key: "ix" value { int64_list { value: [ 42 ] } } }
    }
  ]
  #And arguments
  example_names: ["input0", "input1"],
  features: {
      "sparse": SparseFeature(
          index_key="ix", value_key="val", dtype=tf.float32, size=100),
  }
  #Then the output is a dictionary:
  {
    "sparse": SparseTensor(
        indices=[[0, 3], [0, 20], [1, 42]],
        values=[0.5, -1.0, 0.0]
        dense_shape=[2, 100]),
  }
```

## 其它相关函数

除了以上用到的parse_example方法，还有一些其它方法我们可以简单了解。

### tf.parse_single_example

该方法与tf.parse_example方法基本相同，只不过少了batch函数，单独解析

### tf.parse_single_sequence_example

说到这个方法我们就得提一下tf.train.SequenceExample，望文生义，它与tf.train.Example方法的不同之处在于它多了一个边长的list部分，定义如下

```
message SequenceExample {
  Features context = 1;
  FeatureLists feature_lists = 2;
};
```

我们以源码中电影打分示例来了解一下SequenceExample

```
// context: {
//   feature: {
//     key  : "locale"
//     value: {
//       bytes_list: {
//         value: [ "pt_BR" ]
//       }
//     }
//   }
//   feature: {
//     key  : "age"
//     value: {
//       float_list: {
//         value: [ 19.0 ]
//       }
//     }
//   }
//   feature: {
//     key  : "favorites"
//     value: {
//       bytes_list: {
//         value: [ "Majesty Rose", "Savannah Outen", "One Direction" ]
//       }
//     }
//   }
// }
// feature_lists: {
//   feature_list: {
//     key  : "movie_ratings"
//     value: {
//       feature: {
//         float_list: {
//           value: [ 4.5 ]
//         }
//       }
//       feature: {
//         float_list: {
//           value: [ 5.0 ]
//         }
//       }
//     }
//   }
//   feature_list: {
//     key  : "movie_names"
//     value: {
//       feature: {
//         bytes_list: {
//           value: [ "The Shawshank Redemption" ]
//         }
//       }
//       feature: {
//         bytes_list: {
//           value: [ "Fight Club" ]
//         }
//       }
//     }
//   }
//   feature_list: {
//     key  : "actors"
//     value: {
//       feature: {
//         bytes_list: {
//           value: [ "Tim Robbins", "Morgan Freeman" ]
//         }
//       }
//       feature: {
//         bytes_list: {
//           value: [ "Brad Pitt", "Edward Norton", "Helena Bonham Carter" ]
//         }
//       }
//     }
//   }
// }
//
```

这是一个用户的电影打分记录，context部分是这个用户不随时间变化的固定属性，而feature_list中就是他观看过的电影，电影的演员，以及对电影打分的记录，是可以一直记录下去的。

官方总结的原则如下：

```
// Context:
//   - All conformant context features K must obey the same conventions as
//     a conformant Example's features (see above).
// Feature lists:
//   - A FeatureList L may be missing in an example; it is up to the
//     parser configuration to determine if this is allowed or considered
//     an empty list (zero length).
//   - If a FeatureList L exists, it may be empty (zero length).
//   - If a FeatureList L is non-empty, all features within the FeatureList
//     must have the same data type T. Even across SequenceExamples, the type T
//     of the FeatureList identified by the same key must be the same. An entry
//     without any values may serve as an empty feature.
//   - If a FeatureList L is non-empty, it is up to the parser configuration
//     to determine if all features within the FeatureList must
//     have the same size.  The same holds for this FeatureList across multiple
//     examples.
```

没有什么复杂的，只是讨论了数据缺失的可能性，另外需要注意list的长度是否固定取决于解析时的设置，例如使用FixedLenSequenceFeature时就会要求等长。

而tf.parse_single_sequence_example就是解析此种数据格式的，与parse_example大同小异，这里就不展开了，也可以从参考资料中找到示例代码

## 参考资料

[Tensorflow高阶读写教程](https://blog.csdn.net/u010223750/article/details/70482498)

[TensorFlow源码](https://github.com/tensorflow/tensorflow/blob/r1.7/tensorflow/python/ops/parsing_ops.py)
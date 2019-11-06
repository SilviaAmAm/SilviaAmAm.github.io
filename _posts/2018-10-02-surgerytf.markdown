---
layout: post
title:  "Surgery of TensorFlow (v1.10) graphs"
date:   2018-10-02 18:41:11 +0100
categories: TensorFlow
mathjax: true
---

<p align="center"><img src="/images/surgerytf_images/surgery_tf.png" width="250"></p>

This short post shows how to replace some operations in a TensorFlow graph that has been reloaded from a `.pb` file. This is actually quite simple to do, but all the tutorials that I found when I needed to do this were for older versions of TensorFlow, so it took me longer than it should have. 

First of all, let's create a toy graph and save it to a `.pb` file.

{% highlight python %}
import tensorflow as tf
import numpy as np

# Matrix of inputs to feed to the graph
feed_a = np.ones(shape=(2, 4))

# Creating the graph for the operation a*b + c
a = tf.placeholder(shape=[2, 4], dtype=tf.float32, name="Input_a")
b = tf.ones(shape=[4, 3], name='b')

m = tf.matmul(a, b, name="matmul")

c = tf.ones(shape=[2, 3], name="c")

out = tf.add(m, c, name="out")

# Running the graph in a session and saving it
with tf.Session() as sess:
    result = sess.run(out, feed_dict={a: feed_a})
    print(result)

    graph = tf.get_default_graph()

    a_tf = graph.get_tensor_by_name("Input_a:0")

    out_tf = graph.get_tensor_by_name("out:0")
    tf.saved_model.simple_save(sess, export_dir="saved_model", inputs={"Input_a:0": a_tf}, outputs={"out:0":out_tf})

{% endhighlight %}

This graph is composed of matrix `a` of shape `(2, 4)` which gets multiplied by matrix `b` of shape (4, 3), so that the resulting matrix `m` has shape `(2, 3)`. Matrix `m` is then added to matrix `c` of shape `(2, 3)` to give the final result.

Then, the graph is saved and one has to specify what are the input and output tensors of the graph. In this case they are `a` and `out`.

Now, we want to reload the graph but we would like `a` and `b` to be matrices of shape `(2, 6)` and `(6, 3)` instead of `(4, 3)` and `(4, 3)`.

So we create two new tensors `new_a` and `new_b` with the desired shapes and multiply them together with an operation that we call `matmul_new`.

Then, inside the session we reload the graph that we previously saved. They key part is to add this argument when the graph is reloaded:

```
input_map={"matmul:0": new_m}
```

which replaces the old `matmul` operation with the new one. Now, when we run the tensor with name `out:0` from the graph we need to feed a matrix of shape `(2,6)` instead of `(4,6)`, because the placeholder has been changed!

{% highlight python %}
import tensorflow as tf
import numpy as np

new_feed_a = np.ones(shape=(2, 6))

new_a = tf.placeholder(shape=[2, 6], dtype=tf.float32, name="Input_a_new")
new_b = tf.ones(shape=[6, 3], name='b_new')
new_m = tf.matmul(new_a, new_b, name="matmul_new")

with tf.Session() as sess:

    tf.saved_model.loader.load(sess, [tf.saved_model.tag_constants.SERVING], "saved_model", input_map={"matmul:0": new_m})

    new_out = tf.get_default_graph().get_tensor_by_name("out:0")
    new_result = sess.run(new_out, feed_dict={new_a: new_feed_a})

print(new_result)
{% endhighlight %}

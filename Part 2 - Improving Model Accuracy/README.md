# fizzbuzz_deep_neural_network
## Improving FizzBuzz Approximation

In the previous experiment, the FizzBuzz function was approximated from 1 to 100. Here, training data will scale up to the range of 1,001 to 1,000,000 and testing data to the range of 1 to 1,000. In addition, I will introduce additional hidden layers for a deeper network and a mode of regulatization known as dropout. During session, I will apply random shuffling and batch learning of the training data.

## Required tools

    import tensorflow as tf
    import numpy as np

## Minor Revisions

I am encoding the X (input) values as 20-bit binary to support 1 million training elements:

    # encoding values of X
    def binary_encode_20b_array(a):
        encoded_a = list()
        for i, elem in enumerate(a):
            encoded_a.append(binary_encode_20b(elem))
            progress = i * 100. / len(a)
            if progress % 1 == 0:
                print("binary encoding", str(progress), "%")
        return np.array(encoded_a)

    def binary_encode_20b(val):
        bin_str = format(val, '020b')
        return np.array([int(bit) for bit in bin_str])

The original FizzBuzz function is now a part of the one-hot encoding process:

    # encoding values of Y
    def one_hot_encode_array(a):
        encoded_a = list()
        for i, elem in enumerate(a):
            encoded_a.append(one_hot_encode(int(elem)))
            progress = i * 100. / len(a)
            if progress % 1 == 0:
                print("one-hot encoding", str(progress), "%")
        return np.array(encoded_a)

    def one_hot_encode(val):
        if val % 15 == 0:  # FizzBuzz
            return np.array([1, 0, 0, 0])
        elif val % 3 == 0:  # Fizz
            return np.array([0, 1, 0, 0])
        elif val % 5 == 0:  # Buzz
            return np.array([0, 0, 1, 0])
        else:  # None of the above
            return np.array([0, 0, 0, 1])

For example, if `[0.03 -0.4 -0.4  0.4]` is returned, the program knows not to print any of "Fizz", "Buzz", or "FizzBuzz":

    # decoding values of Y
    def one_hot_decode_array(x, y):
        decoded_a = list()
        for index, elem in enumerate(y):
            decoded_a.append(one_hot_decode(x[index], elem))
        return np.array(decoded_a)

    def one_hot_decode(x, val):
        return ['FizzBuzz', 'Fizz', 'Buzz', str(x)][np.argmax(val)]

## Initializing Data
This is how I am dividing up the training and testing data:

    # train with data that will not be tested
    test_x_start = 1
    test_x_end = 1000
    train_x_start = 1001
    train_x_end = 1000000

    test_x_raw = np.arange(test_x_start, test_x_end + 1)
    test_x = binary_encode_20b_array(test_x_raw).reshape([-1, 20])
    test_y_raw = np.arange(test_x_start, test_x_end + 1)
    test_y = one_hot_encode_array(test_y_raw)

    train_x_raw = np.arange(train_x_start, train_x_end + 1)
    train_x = binary_encode_20b_array(train_x_raw).reshape([-1, 20])
    train_y_raw = np.arange(train_x_start, train_x_end + 1)
    train_y = one_hot_encode_array(train_y_raw)

The model trains using values between 1,001 and 1,000,000 and tests using values between 1 and 1,000.

## Neural Network Model

Dimensions of my model architecture are the following:

    # define params
    input_dim = 20
    output_dim = 4
    h1_dim = 128
    h2_dim = 64
    h3_dim = 32

Therefore, there are three hidden layers:

    # build graph
    X = tf.placeholder(tf.float32, [None, input_dim])
    Y = tf.placeholder(tf.float32, [None, output_dim])

    h1_w = tf.Variable(tf.random_normal([input_dim, h1_dim], stddev=0.1))
    h1_b = tf.Variable(tf.constant(0.1, shape=[h1_dim]))
    h1_z = tf.nn.relu(tf.matmul(X, h1_w) + h1_b)

    h2_w = tf.Variable(tf.random_normal([h1_dim, h2_dim], stddev=0.1))
    h2_b = tf.Variable(tf.constant(0.1, shape=[h2_dim]))
    h2_z = tf.nn.relu(tf.matmul(h1_z, h2_w) + h2_b)

    h3_w = tf.Variable(tf.random_normal([h2_dim, h3_dim], stddev=0.1))
    h3_b = tf.Variable(tf.constant(0.1, shape=[h3_dim]))
    h3_z = tf.nn.relu(tf.matmul(h2_z, h3_w) + h3_b)

Add dropout to the output of hidden layer #3:

    keep_prob = tf.placeholder(tf.float32)
    h3_dropout = tf.nn.dropout(h3_z, keep_prob)

Define the output layer as a function of dropout probability and the output of hidden layer #3:

    fc_w = tf.Variable(tf.random_normal([h3_dim, output_dim], stddev=0.1))
    fc_b = tf.Variable(tf.constant(0.1, shape=[output_dim]))
    Z = tf.matmul(h3_dropout, fc_w) + fc_b

I am using the AdamOptimizer to minimize the cross entropy of the observed vs expected distribution:

    # define cost
    cross_entropy = tf.reduce_mean(tf.nn.softmax_cross_entropy_with_logits(labels=Y, logits=Z))

    # define op
    train_step = tf.train.AdamOptimizer(0.0005).minimize(cross_entropy)

    # define accuracy
    correct_prediction = tf.equal(tf.argmax(Z, 1), tf.argmax(Y, 1))
    correct_prediction = tf.cast(correct_prediction, tf.float32)
    accuracy = tf.reduce_mean(correct_prediction)

I find that a learning rate of 0.0005 is suitable for this problem.

## Running the Model

Define training conditions:

    n_epochs = 10000
    n_batch = 2048

For all batchs of each epoch, train using data that is shuffled each epoch, in batches of size n_batch:

    with tf.Session() as sess:
        sess.run(tf.global_variables_initializer())

        for i in range(n_epochs):
            # shuffle training data
            shuffle = np.random.permutation(range(len(train_x)))
            train_x, train_y = train_x[shuffle], train_y[shuffle]

            # batch train
            for start in range(0, len(train_x), n_batch):
                end = start + n_batch
                sess.run(train_step, feed_dict={X: train_x[start:end], Y: train_y[start:end], keep_prob:0.5})

            # get accuracy
            train_accuracy = sess.run(accuracy, feed_dict={X: train_x, Y: train_y, keep_prob:1.0})
            test_accuracy = sess.run(accuracy, feed_dict={X: test_x, Y: test_y, keep_prob:1.0})
            print("Epoch:", i, "train accuracy:", train_accuracy, "test accuracy:", test_accuracy)

            if train_accuracy == 1.0 and test_accuracy == 1.0:
                break

        # get output of test data
        output = sess.run(Z, feed_dict={X: test_x, keep_prob:1.0})

Note that probability of dropout is `1.0-keep_prob`, which in this case is 50% during training.

## Results

    Epoch: 0 train accuracy: 0.533333 test accuracy: 0.533
    Epoch: 1 train accuracy: 0.533333 test accuracy: 0.533
    Epoch: 2 train accuracy: 0.533333 test accuracy: 0.533
    Epoch: 3 train accuracy: 0.533333 test accuracy: 0.533
    Epoch: 4 train accuracy: 0.533333 test accuracy: 0.533

    ...

    Epoch: 233 train accuracy: 0.989274 test accuracy: 0.998
    Epoch: 234 train accuracy: 0.98877 test accuracy: 0.998
    Epoch: 235 train accuracy: 0.985099 test accuracy: 0.993
    Epoch: 236 train accuracy: 0.989605 test accuracy: 0.998
    Epoch: 237 train accuracy: 0.986278 test accuracy: 1.0

    ...

    Epoch: 1293 train accuracy: 1.0 test accuracy: 1.0
    Epoch: 1294 train accuracy: 1.0 test accuracy: 1.0
    Epoch: 1295 train accuracy: 1.0 test accuracy: 1.0
    Epoch: 1296 train accuracy: 1.0 test accuracy: 1.0
    Epoch: 1297 train accuracy: 1.0 test accuracy: 1.0

By the 1293rd epoch, both train and test performance reach 100% accuracy.
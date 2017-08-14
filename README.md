# Aspect-Based Sentiment Analysis using Tree-Structured LSTMs

In this code base, we implement a [Constituency Tree-LSTM model](https://nlp.stanford.edu/pubs/tai-socher-manning-acl2015.pdf) for [sentence level aspect based sentiment analysis (ABSA)](http://alt.qcri.org/semeval2016/task5/index.php?id=data-and-tools). The training/validation dataset for the model consists of annotated sentences from a domain with a predefined, fixed set of aspects. The annotation for a sentence comprises a list of aspect, polarity tuples. For illustration,  we list a few annotated sentences from the [Laptop review trial dataset](http://alt.qcri.org/semeval2014/task4/data/uploads/laptops-trial.xml):

Sentence: The So called laptop runs to Slow and I hate it! → Annotation: {(LAPTOP#OPERATION_PERFORMANCE, negative), (LAPTOP#GENERAL, negative)}

Sentence: Do not buy it! → Annotation: {LAPTOP#GENERAL, negative}

Sentence: It is the worst laptop ever → Annotation: {LAPTOP#GENERAL, negative}

The model need to be trained over these annotated sentences so that it can, for a new sentence,
1. output the list of aspects present in it 
2. predict the sentiment polarity (negative, positive, mildly negative/positive) associated with it.

## Laptop Review Dataset & Preprocessing

For model training and validation, we use the [Laptop review data set](http://metashare.ilsp.gr:8080/repository/browse/semeval-2014-absa-train-data-v20-annotation-guidelines/683b709298b811e3a0e2842b2b6a04d7c7a19307f18a4940beef6a6143f937f0/). This dataset comprises 3048 sentences extracted from customer reviews of laptops and spans 154 aspects. Out of these, the 17 most frequently occurring aspects account for ~80% of the aspect labels in the dataset. So as a pre-processing step, we collapse the remaining aspects into a miscellaneous category ‘Other’ to bound model complexity and avoid over-fitting. Therefore, in the revised task, the model has to predict the presence/absence of 18 aspects in a sentence.

We use the [Stanford NLP parser](https://nlp.stanford.edu/software/lex-parser.shtml) to generate binary parse trees for each sentence in the training set as part of an offline pre-processing step. The aspect-polarity annotations are encoded as an 18 character string attached to the root node. The position in the 18 character string signifies the aspect, and the character – ‘1’ for positive sentiment, '2' for mildly positive/negative, '3' for negative -  encodes the aspect's polarity with '0' implying the aspect is missing . For instance, 

(100000000000000000 ( ( Not) ( ( bad))))

has one aspect – overall performance – which is encoded by the first character in the string. Since there are no other aspects, the remaining positions are encoded by ‘0’. The labelled instances are stored in the train folder in the train, dev and text files.

## Implementation Details
The [code](LSTM_Tree-v2.ipynb) itself is a modified version of [Tree-Structured LSTM](https://github.com/tensorflow/fold/blob/master/tensorflow_fold/g3doc/sentiment.ipynb). It departs from the original code-base in the following ways: 

1.	Multi-aspect labels appended only to the root node: For ABSA, all annotations are with respect to the entire sentence and therefore attached to the root node. The remaining nodes of the tree do not contribute to the loss.

2. Computing loss from two tasks: ASBA comprises two tasks – aspect detection and polarity prediction for the present aspects. So it makes sense to compute a separate loss for each of the tasks and sum them in the final layer of the neural network. For task 1 loss, we compute the loss over 18 softmax units and sum them. Each softmax unit in this layer predicts the presence/absence of the corresponding aspect. For task 2, we compute loss over another set of 18 softmax units and sum them. Here, each softmax unit predicts the sentiment class (positive, negative, mildly positive/mildly negative) for the corresponding aspect. Note that for each of the 18 aspectss, loss is computed only if the aspect is present otherwise it is taken to be 0. The final loss is the weighted sum of these two losses. A key advantage of this approach is that we can adjust the weights depending on which evaluation metric – F1 score for aspect detection or accuracy score for sentiment polarity prediction – matters more to you. 

3.	Non-training of the word vectors: Since the dataset is limited in size, we avoid backpropagating the error into the word2vecs for each word in leaf node of the parse tree.


## Requirements
We use the [Tensorflow Fold](https://github.com/tensorflow/fold) library to facilitate dynamic batching of trees and train more efficiently. Batching allows for a more stable estimate of the gradients which in turn permits training initializaton with a larger learning rate. Combined with faster sweeps through epochs, dynamic batching reduces overall training time. Note that TensorFlow by itself doesn’t allow for batching of variable sized inputs such as trees.

To use TensorFlow Fold, we used Tenforflow 1.0.0  GPU version on Ubuntu 16.04 with Python 3.6.2. We recommend training over a GPU for speed-up.

We use 300 dimensional word2vecs pre-trained on the [Google News](https://github.com/mmihaltz/word2vec-GoogleNews-vectors) corpus as inputs to the leaf nodes. In the current implementation, we store this file in the root folder. After creating a vocabulary over the training, dev and test sets, a word2vec embedding matrix is then created using the Google word2vec file. The out of vocabulary rate for the Latptop data-set is ~ 5%. 

Other key requirements/dependancies are:

-Gensim for handling Google News word2vecs

-Numpy


## Results and Evaluation
As part of validation, we track the F1 score for [SamEval](http://alt.qcri.org/semeval2016/task5/index.php?id=data-and-tools) subtask 1 slot 1 and the accuracy score for subtask 1 slot 3. For our model, both the scores are very competitive with the results of the leading teams in SamEval’15 and SamEval’16.

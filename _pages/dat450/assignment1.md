---
layout: page
title: 'DAT450/DIT247: Programming Assignment 1: Introduction to language modeling'
permalink: /courses/dat450/assignment1/
description:
nav: false
nav_order: 4
---

# DAT450/DIT247: Programming Assignment 1: Introduction to language modeling

*Language modeling* is the foundation that recent advances in NLP technlogies build on. In essence, language modeling means that we learn how to imitate the language that we observe in the wild. More formally, we want to train a system that models the statistical distribution of natural language. Solving this task is exactly what the famous commercial large language models do (with some additional post-hoc tweaking to make the systems more interactive and avoid generating provocative outputs).

In the course, we will cover a variety of technical solutions to this fundamental task (e.g. recurrent models and Transformers). In this first assignment of the course, we are going to build a neural network-based language model using simple techniques that should be familiar to anyone with an experience of neural networks. In essence, our solution is going to be similar to the one described by [Bengio et al. (2003)](https://www.jmlr.org/papers/volume3/bengio03a/bengio03a.pdf).

### Pedagogical purposes of this assignment
- Introducing the task of language modeling
- Getting experience of preprocessing text
- Understanding the concept of word embeddings
- Refreshing basic skills in how to set up and train a neural network

### Requirements

Please submit your solution in [Canvas](https://chalmers.instructure.com/courses/31739/assignments/98342). **Submission deadline**: November 11.

Submit a notebook containing your solution to the programming tasks described below. This is a pure programming assignment and you do not have to write a technical report or explain details of your solution in the notebook: there will be a separate individual assignment where you will answer some conceptual questions about what you have been doing here.

## Step 0: Preliminaries

If you are working on your own machine, make sure that the following libraries are installed:
- [NLTK](https://www.nltk.org/install.html) or [SpaCy](https://spacy.io/usage) for tokenization,
- [PyTorch](https://pytorch.org/get-started/locally/) for building and training the models,
- Optional: [Matplotlib](https://matplotlib.org/stable/users/getting_started/) and [scikit-learn](https://scikit-learn.org/stable/install.html) for the embedding visualization in the last step.
If you are using a Colab notebook, these libraries are already installed.

For the third part of the assignment, you will need to understand some basic concepts of PyTorch such as tensors, models, optimizers, loss functions and how to write the training loop. There are plenty of tutorials available, for instance on the [PyTorch website](https://pytorch.org/tutorials/). From the Applied Machine Learning course, there is also an [example notebook](https://www.cse.chalmers.se/~richajo/dit866/lectures/l7/Implementing%20classifiers%20with%20PyTorch.html) that shows how to train a basic classifier in PyTorch. (But note that if you take code from this notebook, several technical details have to change since our input data and prediction task are different!)

## Step 1: Preprocessing the text

Download and extract [this archive](https://www.cse.chalmers.se/~richajo/diverse/lmdemo.zip), which contains three text files. The files have been created from Wikipedia articles converted into raw text, with all Wiki markup removed. (We'll actually just use the training and validation sets, and you can ignore the test file.)

You will need a *tokenizer* that splits English text into separate words (tokens). In this assignment, you will just use an existing tokenizer. Popular NLP libraries such as SpaCy and NLTK come with built-in tokenizers. We recommend NLTK in this assignment since it is somewhat faster than SpaCy and somewhat easier to use.

<details>
<summary><b>Hint</b>: How to use NLTK's English tokenizer.</summary>

<div style="margin-left: 10px; border-radius: 4px; background: #ddfff0; border: 1px solid black; padding: 5px;">Import the function <code>word_tokenize</code> from the <code>nltk</code> library. If you are running this on your own machine, you will first need to install NLTK with <code>pip</code> or <code>conda</code>. In Colab, NLTK is already installed.

For instance, <code>word_tokenize("Let's test!!")</code> should give the result <code>["Let", "'s", "test", "!", "!"]</code>
</div>
</details>

Each nonempty line in the text files correspond to one paragraph in Wikipedia. Apply the tokenizer to all paragraphs in the training and validation datasets. Convert all words into lowercase.

**Sanity check**: after this step, your training set should consist of around 147,000 paragraphs and the validation set around 18,000 paragraphs. (The exact number depends on what tokenizer you selected.)

## Step 2: Encoding the text as integers

### Building the vocabulary

Create a utility (a function or a class) that goes through the training text and creates a *vocabulary*: a mapping from token strings to integers.

In addition, the vocabulary should contain 3 special symbols:
- a symbol for previously unseen or low-frequency tokens,
- a symbol we will put at the beginning of each paragraph,
- a symbol we will put at the end of each paragraph.

The total size of the vocabulary (including the 3 symbols) should be at most `max_voc_size`, which is is a user-specified hyperparameter. If the number of unique tokens in the text is greater than `max_voc_size`, then use the most frequent ones.

<details>
<summary><b>Hint</b>: A <a href="https://docs.python.org/3/library/collections.html#collections.Counter"><code>Counter</code></a> can be convenient when computing the frequencies.</summary>
<div style="margin-left: 10px; border-radius: 4px; background: #ddfff0; border: 1px solid black; padding: 5px;">A <code>Counter</code> is like a regular Python dictionary, with some additional functionality for computing frequencies. For instance, you can go through each paragraph and call <a href="https://docs.python.org/3/library/collections.html#collections.Counter.update"><code>update</code></a>. After building the <code>Counter</code> on your dataset, <a href="https://docs.python.org/3/library/collections.html#collections.Counter.most_common"><code>most_common</code></a> gives the most frequent items.</div>
</details>

Also create some tool that allows you to go back from the integer to the original word token. This will only be used in the final part of the assignment, where we look at model outputs and word embedding neighbors.

**Example**: you might end up with something like this:
<pre>
str_to_int = { 'BEGINNING':0, 'END':1, 'UNKNOWN':2, 'the':3, 'and':4, ... }

int_to_str = { 0:'BEGINNING', 1:'END', 2:'UNKNOWN', 3:'the', 4:'and', ... }
</pre>

**Sanity check**: after creating the vocabulary, make sure that
- the size of your vocabulary is not greater than the max vocabulary size you specified,
- the 3 special symbols exist in the vocabulary and that they don't coincide with any real words,
- some highly frequent example words (e.g. "the", "and") are included in the vocabulary but that some rare words (e.g. "cuboidal", "epiglottis") are not.
- if you take some test word, you can map it to an integer and then back to the original test word using the inverse mapping.

### Encoding the texts and creating training instances

The model we are going to train will predict the next token given the previous *N* tokens. We will now create the examples we will use for training and evaluation by extracting word sequences from the provided texts.

First, make an educated guess about the size of the context window *N* you are going to use. (You can go back later on and try out other values.) Any value of *N* greater than 0 should work for the purposes of this assignment. Intuitively, small context windows (e.g. *N*=1) makes the model more stupid but a bit more efficient in terms of time and memory.

Go through the training and validation data and extract all sequences of *N*+1 tokens and map them to the corresponding integer values. Remember to use the special symbols when necessary:
- the "unseen" symbol for tokens not in your vocabulary,
- *N* "beginning" symbols before each paragraph,
- an "end" symbol after each paragraph.

Store all these sequences in lists.

**Example**: If our training text consists of the three tokens *Wonderful news !*, and we use a context window size of 1, we would extract the training instances `[['BEGINNING', 'wonderful'], ['wonderful', 'news'], ['news', '!'], ['!', 'END']]`. After integer encoding, we might have something like `[[0, 2], [2, 3], [3, 4], [4, 1]]`, depending on how our encoding works.

**Sanity check**: after these steps, you should have around 12 million training instances and 1.5 million validation instances.

## Step 3: Developing a language model

### Creating training batches

When using neural networks, we typically process several instances at the time when training the model and when running the trained model.
Make sure that you have a way to put your training instances in batches of a fixed size. It is enough to go through the training instances with a `for` loop, taking steps of length `batch_size`,  but most solutions would probably use a `DataLoader` here.
Put each batch in a PyTorch tensor (e.g. by calling `torch.as_tensor`).

<details>
<summary><b>Hint</b>: More information about <a href="https://pytorch.org/tutorials/beginner/basics/data_tutorial.html"><code>DataLoader</code></a>.</summary>
<div style="margin-left: 10px; border-radius: 4px; background: #ddfff0; border: 1px solid black; padding: 5px;">
PyTorch provides a utility called <a href="https://pytorch.org/tutorials/beginner/basics/data_tutorial.html"><code>DataLoader</code></a> to help us create batches. It can work on a variety of underlying data structures, but in this assignment, we'll just apply it to the list you prepared previously.
<pre>
dl = DataLoader(your_list, batch_size=..., shuffle=..., collate_fn=torch.as_tensor)
</pre>
The arguments here are as follows:
<ul>
<li><code>batch_size</code>: the number of instances in each batch.</li>
<li><code>shuffle</code>: whether or not we rearrange the instances randomly. It is common to shuffle instances while training.</li>
<li><code>collate_fn</code>: a function that defines how each batch is created. In our case, we just want to put each batch in a tensor.</li>
</ul>
When you have created a <code>DataLoader</code>, you can iterate through the dataset batch by batch:
<pre>for batch in dl:
   ... do something with each batch ...</pre>
</div>
</details>

**Sanity check**: Make sure that your batches are PyTorch tensors of shape (*B*, *N*+1) where *B* is the batch size and *N* the number of context tokens. (Depending on your batch size, the last batch in the training set might be smaller than *B*.) Each batch should contain integers, not floating-point numbers.

### Setting up the neural network structure

Set up a neural network inspired by the neural language model proposed by [Bengio et al. (2003)](https://www.jmlr.org/papers/volume3/bengio03a/bengio03a.pdf). The main components are:
- an *embedding layer* that maps token integers to floating-point vectors,
- *intermediate layers* that map between input and output representations,
- an *output layer* that computes (the logits of) a probability distribution over the vocabulary.

You are free to experiment with the design of the intermediate layers and you don't have to follow the exact structure used in the paper.

<details>
<summary><b>Hint</b>: Setting up a neural network in PyTorch.</summary>
<div style="margin-left: 10px; border-radius: 4px; background: #ddfff0; border: 1px solid black; padding: 5px;">There are a few different ways that we can write code to set up a neural network in PyTorch.

If your model has the traditional structure of stacked layers, then the most concise way to declare the model is to use <code>nn.Sequential</code>:
<pre>
model = nn.Sequential(
  layer1,
  layer2,
  ...
  layerN)
</pre>
You can use any type of layers here. In our case, you'll typically start with a <code>nn.Embedding</code> layer, followed by some intermediate layers (e.g. <code>nn.Linear</code> followed by some activation such as <code>nn.ReLU</code>), and then a linear output layer.

A more general solution is to declare your network as a class that inherits from <code>nn.Module</code>. You will then have to declare your model components in <code>__init__</code> and define the forward computation in `forward`:
<pre>
class MyNetwork(nn.Module):
  def __init__(self, hyperparameters):
    super().__init__()
    self.layer1 = ... some layer ...
    self.layer2 = ... some layer ...
    ...

  def forward(self, inputs):
    step1 = self.layer1(inputs)
    step2 = self.layer2(step1)
    ...
    return something
</pre>

The second coding style, while more verbose, has the advantage that it is easier to debug: for instance, it is easy to check the shapes of intermediate computations. It is also more flexible and allows you to go beyond the constraints of a traditional layered setup.</div>
</details>

<details>
<summary><b>Hint</b>: It can be useful to put a <a href="https://pytorch.org/docs/stable/generated/torch.nn.Flatten.html"><code>nn.Flatten</code></a> after the embedding layer.</summary>
<div style="margin-left: 10px; border-radius: 4px; background: #ddfff0; border: 1px solid black; padding: 5px;"><a href="https://pytorch.org/docs/stable/generated/torch.nn.Flatten.html"><code>nn.Flatten</code></a> is a convenient tool that you can put after the embedding layer to get the right tensor shapes. Let's say we have a batch of <em>B</em> inputs, each of which is a context window of size <em>N</em>, so our input tensor has the shape (<em>B</em>, <em>N</em>). The output from the embedding layer will then have the shape (<em>B</em>, <em>N</em>, <em>D</em>) where <em>D</em> is the embedding dimensionality. If you use a <code>nn.Flatten</code>, we go back to a two-dimensional tensor of shape (<em>B</em>, <em>N</em>*<em>D</em>). That is, we can see this as a step that concatenates the embeddings of the tokens in the context window.</div>
</details>

**Sanity check**: carry out the following steps:
- Create an integer tensor of shape 1x*N* where *N* is the size of the context window. It doesn't matter what the integers are except that they should be less than the vocabulary size. (Alternatively, take one instance from your training set.)
- Apply the model to this input tensor. It shouldn't crash here.
- Make sure that the shape of the returned output tensor is 1x*V* where *V* is the size of the vocabulary. This output corresponds to the logits of the next-token probability distribution, but it is useless at this point because we haven't yet trained the model.

### Training the model

Before training, we need two final pieces:

- A *loss* that we want to minimize. In our case, this is going to be the cross-entropy loss, implemented in PyTorch as <a href="https://pytorch.org/docs/stable/generated/torch.nn.CrossEntropyLoss.html"><code>nn.CrossEntropyLoss</code></a>. Minimizing this loss corresponds to minimizing the negative log probability of the observed tokens.
- An *optimizer* that updates model parameters, based on the gradients with respect to the model parameters. The optimizer typically implements some variant of [stochastic gradient descent](https://en.wikipedia.org/wiki/Stochastic_gradient_descent). An example of a popular optimizer is <a href="https://pytorch.org/docs/stable/generated/torch.optim.AdamW.html"><code>AdamW</code></a>.

Now, we are ready to train the neural network on the training set. Using the loss, the optimizer, and the training batches, write a *training loop* that iterates through the batches and updates the model incrementally to minimize the loss.

If you followed our previous implementation advice, your training batches will
be integer tensors of shape (*B*, *N*+1) where *B* is the batch size and *N* is the context window size; the first *N* columns correspond to the context windows and the last column the tokens we are predicting.
So while you are training, you will apply the model to the first *N* columns of the batch and compute the loss with respect to the last column.

<details>
<summary><b>Hint</b>: A typical PyTorch training loop.</summary>
<div style="margin-left: 10px; border-radius: 4px; background: #ddfff0; border: 1px solid black; padding: 5px;">
<pre>
for each training epoch:
    for each batch B in the training set:
        FORWARD PASS:
	X = first N columns in B
	Y = last column in B
	put X and Y on the GPU
        apply the model to X
	compute the loss for the model output and Y
        BACKWARD PASS (updating the model parameters):
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()	
</pre>
</div>
</details>

<details>
<summary><b>Hint</b>: Some advice on development.</summary>
<div style="margin-left: 10px; border-radius: 4px; background: #ddfff0; border: 1px solid black; padding: 5px;">
While developing the code, work with very small datasets until you know it doesn't crash, and then use the full training set. Monitor the cross-entropy loss (and/or the perplexity) over the training: if the loss does not decrease while you are training, there is probably an error. For instance, if the learning rate is set to a value that is too large, the loss values may be unstable or increase.
</div>
</details>

## Step 4: Evaluation and analysis

### Predicting the next word

Take some example context window and use the model to predict the next word.
- Apply the model to the integer-encoded context window. As usual, this gives you (the logits of) a probability distribution over your vocabulary.
- Use <a href="https://pytorch.org/docs/stable/generated/torch.argmax.html"><code>argmax</code></a> to find the index of the highest-scoring item, or <a href="https://pytorch.org/docs/stable/generated/torch.topk.html"><code>topk</code></a> to find the indices and scores of the *k* highest-scoring items.
- Apply the inverse vocabulary encoder (that you created in Step 2) so that you can understand what words the model thinks are the most likely in this context.

### Quantitative evaluation

The most common way to evaluate language models quantitatively is the [perplexity](https://huggingface.co/docs/transformers/perplexity) score on a test dataset. The better the model is at predicting the actually occurring words, the lower the perplexity. This quantity is formally defined as follows:

$$\text{perplexity} = 2^{-\frac{1}{m}\sum_{i=1}^m \log_2 P(w_i | c_i)}$$

In this formula, *m* is the number of words in the dataset, *P* is the probability assigned by our model, <em>w<sub>i</sub></em> and <em>c<sub>i</sub></em> the word and context window at each position.

Compute the perplexity of your model on the validation set. The exact value will depend on various implementation choices you have made, how much of the training data you have been able to use, etc. Roughly speaking, if you get perplexity scores around 700 or more, there are probably problems. Carefully implemented and well-trained models will probably have perplexity scores in the range of 200&ndash;300.

<details>
<summary><b>Hint</b>: An easy way to compute the perplexity in PyTorch.</summary>
<div style="margin-left: 10px; border-radius: 4px; background: #ddfff0; border: 1px solid black; padding: 5px;">
As you can see in the formula, the perplexity is an exponential function applied to the mean of the negative log probability of each token.
You are probably already computing the <em>cross-entropy loss</em> as part of your training loop, and this actually computes what you need here.

The perplexity is traditionally defined in terms of logarithms of base 2. However, we will get the same result regardless of what logarithmic base we use. So it is OK to use the natural logarithms and exponential functions, as long as we are consistent: this means that we can compute the perplexity by applying <code>exp</code> to the mean of the cross-entropy loss over your batches in the validation set.
</div>
</details>

If you have time for exploration, investigate the effect of the context window size *N* (and possibly other hyperparameters such as embedding dimensionality) on the model's perplexity.

### Inspecting the word embeddings

It is common to say that neural networks are "black boxes" and that we cannot fully understand their internal mechanics, especially as they grow larger and structurally more complex. The research area of model interpretability aims to develop methods to help us reason about the high-level functions the models implement.

In this assignment, we will briefly investigate the [embeddings](https://en.wikipedia.org/wiki/Word_embedding) that your model learned while you trained it.
If we have successfully trained a word embedding model, an embedding vector stores a crude representation of "word meaning", so we can reason about the learned meaning representations by investigating the geometry of the vector space of word embeddings.
The most common way to do this is to look at nearest neighbors in the vector space: intuitively, if we look at some example word, its neighbors should correspond to words that have a similar meaning.

Select some example words (e.g. `"sweden"`) and look at their nearest neighbors in the vector space of word embeddings. Does it seem that the nearest neighbors make sense?

<details>
<summary><b>Hint</b>: Example code for computing nearest neighbors.</summary>
<div style="margin-left: 10px; border-radius: 4px; background: #ddfff0; border: 1px solid black; padding: 5px;">
The following code shows how to compute the nearest neighbors in the embedding space of a given word. Depending on your implementation, you may need to change some details. Here, <code>emb</code> is the <code>nn.Embedding</code> module of your language model, while <code>voc</code> and <code>inv_voc</code> are the string-to-integer and integer-to-string mappings you created in Step 2.
<pre>
def nearest_neighbors(emb, voc, inv_voc, word, n_neighbors=5):

    # Look up the embedding for the test word.
    test_emb = emb.weight[voc[word]]
    
    # We'll use a cosine similarity function to find the most similar words.
    sim_func = nn.CosineSimilarity(dim=1)
    cosine_scores = sim_func(test_emb, emb.weight)
    
    # Find the positions of the highest cosine values.
    near_nbr = cosine_scores.topk(n_neighbors+1)
    topk_cos = near_nbr.values[1:]
    topk_indices = near_nbr.indices[1:]
    # NB: the first word in the top-k list is the query word itself!
    # That's why we skip the first position in the code above.
    
    # Finally, map word indices back to strings, and put the result in a list.
    return [ (inv_voc[ix.item()], cos.item()) for ix, cos in zip(topk_indices, topk_cos) ]
</pre>
</div>
</details>

Optionally, you may visualize some word embeddings in a two-dimensional plot.
<details>
<summary><b>Hint</b>: Example code for PCA-based embedding scatterplot.</summary>
<div style="margin-left: 10px; border-radius: 4px; background: #ddfff0; border: 1px solid black; padding: 5px;">
<pre>
from sklearn.decomposition import TruncatedSVD
import matplotlib.pyplot as plt
def plot_embeddings_pca(emb, inv_voc, words):
    vectors = np.vstack([emb.weight[inv_voc[w]].cpu().detach().numpy() for w in words])
    vectors -= vectors.mean(axis=0)
    twodim = TruncatedSVD(n_components=2).fit_transform(vectors)
    plt.figure(figsize=(5,5))
    plt.scatter(twodim[:,0], twodim[:,1], edgecolors='k', c='r')
    for word, (x,y) in zip(words, twodim):
        plt.text(x+0.02, y, word)
    plt.axis('off')

plot_embeddings_pca(model[0], prepr, ['sweden', 'denmark', 'europe', 'africa', 'london', 'stockholm', 'large', 'small', 'great', 'black', '3', '7', '10', 'seven', 'three', 'ten', '1984', '2005', '2010'])
</pre>
</div>
</details>
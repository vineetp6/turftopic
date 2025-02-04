Turftopic is a topic modeling library which intends to simplify and streamline the usage of contextually sensitive topic models.
We provide stable, minimal and scalable implementations of several types of models along with extensive documentation,
so that you can make an informed choice about which model suits you best in the light of a given task or research question.

## Installation

Turftopic can be installed from PyPI.

```bash
pip install turftopic
```

If you intend to use CTMs, make sure to install the package with Pyro as an optional dependency.

```bash
pip install turftopic[pyro-ppl]
```

## Basic Usage

Turftopic's models follow the scikit-learn API conventions, and as such they are quite easy to use if you are familiar with
scikit-learn workflows.

Here's an example of how you use KeyNMF, one of our models on the 20Newsgroups dataset from scikit-learn.

```python
from sklearn.datasets import fetch_20newsgroups

newsgroups = fetch_20newsgroups(
    subset="all",
    remove=("headers", "footers", "quotes"),
)
corpus = newsgroups.data
```

Turftopic also comes with interpretation tools that make it easy to display and understand your results.

```python
from turftopic import KeyNMF

model = KeyNMF(20).fit(corpus)
model.print_topics()
```

<center>

| Topic ID | Top 10 Words                                                                                    |
| -------- | ----------------------------------------------------------------------------------------------- |
|        0 | armenians, armenian, armenia, turks, turkish, genocide, azerbaijan, soviet, turkey, azerbaijani |
|        1 | sale, price, shipping, offer, sell, prices, interested, 00, games, selling                      |
|        2 | christians, christian, bible, christianity, church, god, scripture, faith, jesus, sin           |
|        3 | encryption, chip, clipper, nsa, security, secure, privacy, encrypted, crypto, cryptography      |
|         | ....                                |

</center>


## Important Attributes

In Turftopic all models have a vectorizer and an encoder component, which you can specify when initializing a model.

1. The __vectorizer__ is used to turn documents into Bag-of-Words representations and learning the vocabulary. The default used in the package is sklearn's `CountVectorizer`.
1. The __encoder__ is used to encode documents, and optionally the vocabulary into contextual representations. This will most frequently be a Sentence Transformer. The default in Turftopic is `all-MiniLM-L6-v2`, a very lightweight English model.

You can use any of the built-in encoders in Turftopic to encode your documents, or any sentence transformer from the HuggingFace Hub.
This allows you to use embeddings of different quality and computational efficiency for different purposes.

Here's a model that uses E5 large as the embedding model, and only learns words that occur in at least 20 documents.

```python
from turftopic import KeyNMF
from sklearn.feature_extraction.text import CountVectorizer

model = KeyNMF(10, encoder="intfloat/e5-large-v2", vectorizer=CountVectorizer(min_df=20))
```

You can also use external models for encoding, here's an example with OpenAI's embedding models:

```python
from turftopic import SemanticSignalSeparation
from turftopic.encoders import OpenAIEmbeddings

model = SemanticSignalSeparation(10, encoder=OpenAIEmbeddings("text-embedding-3-large"))
```

If you intend to, you can also use n-grams as features instead of words:

```python
from turftopic import GMM
from sklearn.feature_extraction.text import CountVectorizer

model = GMM(10, vectorizer=CountVectorizer(ngram_range=(2,4)))
```

## Fitting Models

All models in Turftopic have a `fit()` method, that takes a textual corpus in the form of an iterable of strings.

Beware that the iterable has to be reusable, as models have to do multiple passes over the corpus.

```python
corpus: list[str] = ["this is a a document", "this is yet another document", ...]

model.fit(corpus)
```

#### Performance tips

In order to cut down on costs/computational load when fitting multiple models in a row, you might want to encode the documents before fitting a model.
Encoding the corpus is the heaviest part of the process and you can spare yourself a lot of time by only doing it once.

Some models have to encode the vocabulary as well, this cannot be done before inference, as the models learn the vocabulary itself from the corpus.

The fit method of all models takes and `embeddings` argument, that allows you to pass a precooked embedding matrix along to fitting.

One thing to watch out for is that you have to pass the embedding model along to the model that was used for encoding the corpus.
This is again, to ensure that the vocabulary gets encoded with the same embedding model as the documents.

Here's a snippet of correct usage:

```python
import numpy as np
from sentence_transformers import SentenceTransformer
from turftopic import GMM, ClusteringTopicModel

encoder = SentenceTransformer("intfloat/e5-large-v2")

corpus: list[str] = ["this is a a document", "this is yet another document", ...]
embeddings = np.asarray(encoder.encode(corpus))

gmm = GMM(10, encoder=encoder).fit(corpus, embeddings=embeddings)

clustering = ClusteringTopicModel(encoder=encoder).fit(corpus, embeddings=embeddings)
```

## Inference

In order to get the importance of each topic for the documents in the corpus, you might want to use `fit_transform()` instead of `fit()`

```python
document_topic_matrix = model.fit_transform(corpus)
```

This will give you a matrix, where every row is a document and every column represents the importance of a given topic.

You can infer topical content for new documents with a fitted model using the `transform()` method (beware that this only works with inductive methods):

```python
document_topic_matrix = model.transform(new_documents, embeddings=None)
```

 > Note that using `fit()` and `transform()` in succession is not the same as using `fit_transform()` and the later should be preferred under all circumstances.
 > For one, not all models have a `transform()` method, but `fit_transform()` is also way more efficient, as documents don't have to be encoded twice.
 > Some models have additional optimizations going on when using `fit_transform()`, and the `fit()` method typically uses `fit_transform()` in the background.


## Interpreting Models

Turftopic comes with a number of pretty printing utilities for interpreting the models.

To see the highest the most important words for each topic, use the `print_topics()` method.

```python
model.print_topics()
```

<center>

| Topic ID | Top 10 Words                                                                                    |
| -------- | ----------------------------------------------------------------------------------------------- |
|        0 | armenians, armenian, armenia, turks, turkish, genocide, azerbaijan, soviet, turkey, azerbaijani |
|        1 | sale, price, shipping, offer, sell, prices, interested, 00, games, selling                      |
|        2 | christians, christian, bible, christianity, church, god, scripture, faith, jesus, sin           |
|        3 | encryption, chip, clipper, nsa, security, secure, privacy, encrypted, crypto, cryptography      |
|         | ....                                |


</center>

```python
# Print highest ranking documents for topic 0
model.print_highest_ranking_documents(0, corpus, document_topic_matrix)
```

<center>

| Document                                                                                             | Score |
| -----------------------------------------------------------------------------------------------------| ----- |
| Poor 'Poly'. I see you're preparing the groundwork for yet another retreat from your...              |  0.40 |
| Then you must be living in an alternate universe. Where were they? An Appeal to Mankind During the... |  0.40 |
| It is 'Serdar', 'kocaoglan'. Just love it. Well, it could be your head wasn't screwed on just right... |  0.39 |

</center>

```python
model.print_topic_distribution(
    "I think guns should definitely banned from all public institutions, such as schools."
)
```

<center>

| Topic name                                | Score |
| ----------------------------------------- | ----- |
| 7_gun_guns_firearms_weapons               |  0.05 |
| 17_mail_address_email_send                |  0.00 |
| 3_encryption_chip_clipper_nsa             |  0.00 |
| 19_baseball_pitching_pitcher_hitter       |  0.00 |
| 11_graphics_software_program_3d           |  0.00 |

</center>

### Visualization

Turftopic does not come with built-in visualization utilities, but the next release of [topicwizard](https://github.com/x-tabdeveloping/topicwizard) is going to be compatible with all models from Turftopic.

```bash
pip install topic-wizard
```

By far the easiest way to visualize your models for interpretation is to launch the topicwizard web app.

```python
import topicwizard

topicwizard.visualize(model=model, corpus=corpus)
```

<figure>
  <img src="https://x-tabdeveloping.github.io/topicwizard/_images/screenshot_topics.png" width="70%" style="margin-left: auto;margin-right: auto;">
  <figcaption>Screenshot of the topicwizard Web Application</figcaption>
</figure>


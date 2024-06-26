---
layout: post
title: "Who Like Lindsey Stirling to See at Ravinia 2024, According to a Language Model"
---

# Motivation

This project is motivated by a simple question: who should I go see at the Ravinia Festival in 2024 to have a performance closest to the musical style of Lindsey Stirling?

To answer this question, I collected data from the 2024 Ravinia schedule available online. Sadly, there was no easy way to web scrape this data, so I simply created an Excel spreadsheet containing columns for the artist, the description of the artist, and the date of the performance. I then trained a quick and dirty `doc2vec` model on the artist description to obtain word and document vectors, then used the model to calculate the most similar artist to Lindsey Stirling based on a description of her from Wikipedia.

# Results

The results are surprisingly passable for my simple plug-and-play work.

* Test accuracy looks fine -- I randomly sample an artist who is a classical performer, and the most similar train document is also a classical performer; the least similar train document is a rock performer
* The most similar performance to Lindsey Stirling is a chamber performance from the Ravinia Steans Music Institute, featuring kickass violinist [Midori](https://en.wikipedia.org/wiki/Midori_(violinist)), which seems about right.
* To see this performance, this means that I should go to the Festival on July 7.


```python
from pathlib import Path
import yaml
import pandas as pd
from gensim.utils import simple_preprocess
from gensim.models.doc2vec import TaggedDocument, Doc2Vec
from sklearn.model_selection import train_test_split
import joblib
import numpy as np
import matplotlib.pyplot as plt
```

# Preprocess the data


```python
data_dir = Path('input/')
with open('hand/config.yaml') as f:
    config = yaml.safe_load(f)

config
output_dir = Path('output/' + config['date_output'] + '/')

df_ravinia = pd.read_excel(data_dir / 'ravinia_data_2024.xlsx', engine='openpyxl')

# Preprocessing
df_ravinia['descrip'] = df_ravinia['descrip'].astype(str)
df_ravinia['descrip'] = df_ravinia['descrip'].replace('nan', '')
df_ravinia = df_ravinia[df_ravinia['descrip'] != '']

df_ravinia['date'] = pd.to_datetime(df_ravinia['date'], format='%Y-%m-%d')
```


```python
df_ravinia.head(10)
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>band</th>
      <th>date</th>
      <th>descrip</th>
      <th>genre</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>The Return of The Flock featuring Jerry Goodman</td>
      <td>2024-06-07</td>
      <td>Columbia and Mercury recording artists The Flo...</td>
      <td>jazz</td>
    </tr>
    <tr>
      <th>1</th>
      <td>James Taylor &amp; His All-Star Band</td>
      <td>2024-06-08</td>
      <td>As a recording and touring artist, James Taylo...</td>
      <td>rock</td>
    </tr>
    <tr>
      <th>2</th>
      <td>James Taylor &amp; His All-Star Band</td>
      <td>2024-06-09</td>
      <td>As a recording and touring artist, James Taylo...</td>
      <td>rock</td>
    </tr>
    <tr>
      <th>5</th>
      <td>Robert Plant &amp; Alison Krauss</td>
      <td>2024-06-12</td>
      <td>After closing the 14-year gap between two albu...</td>
      <td>blues</td>
    </tr>
    <tr>
      <th>6</th>
      <td>Kronos Quartet: Five Decades</td>
      <td>2024-06-13</td>
      <td>One of the most influential chamber groups of ...</td>
      <td>classical</td>
    </tr>
    <tr>
      <th>7</th>
      <td>Hauser</td>
      <td>2024-06-14</td>
      <td>Hauser, the superstar cellist whose rise to fa...</td>
      <td>classical</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Michael Franti &amp; Spearhead</td>
      <td>2024-06-15</td>
      <td>Michael Franti, the noted American singer-song...</td>
      <td>hip hop</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Trevor Hall</td>
      <td>2024-06-15</td>
      <td>Singer-songwriter and guitarist Trevor Hall, w...</td>
      <td>folk</td>
    </tr>
    <tr>
      <th>13</th>
      <td>Trombone Shorty</td>
      <td>2024-06-19</td>
      <td>Trombone Shorty, also known as Troy Andrews, i...</td>
      <td>jazz</td>
    </tr>
    <tr>
      <th>14</th>
      <td>Big Boi</td>
      <td>2024-06-19</td>
      <td>Rapper, producer, and actor Big Boi, known bes...</td>
      <td>rap</td>
    </tr>
  </tbody>
</table>
</div>




```python
df_ravinia.shape
```




    (91, 4)



Only keep the first example of each band to avoid train/test leakage


```python
df_ravinia = df_ravinia.groupby('band').first().reset_index()
```

# Split into train + test


```python
df_train, df_test = train_test_split(df_ravinia, random_state=42)
```

Train requires the tokens and document ID


```python
def process_and_tokenize_series(df_pandas,
                                col_to_process,
                                tokens_only=True):
    for i, doc in enumerate(df_pandas[col_to_process]):
        tokens = simple_preprocess(doc) # seems to remove stop words
        if tokens_only:
            yield tokens
        else:
            yield TaggedDocument(tokens, [i])

train = list(process_and_tokenize_series(df_train, col_to_process='descrip', tokens_only=False))
test = list(process_and_tokenize_series(df_test, col_to_process='descrip'))
```

# Instantiate the model + train


```python
vector_size = 100
min_count = 2
epochs = 1500

model = Doc2Vec(
    vector_size=vector_size,
    min_count=min_count,
    epochs=epochs
)
```


```python
model.build_vocab(train)
```


```python
model.train(train, total_examples=model.corpus_count, epochs=model.epochs)
filename_model = 'ravinia_model.sav'
joblib.dump(model, output_dir / filename_model)
```




    ['output\\2024_03_25\\ravinia_model.sav']



# Test


```python
test_doc_idx = 0
test_doc = test[test_doc_idx]
print(test_doc)
test_doc_vec = model.infer_vector(test_doc)

sims = model.dv.most_similar([test_doc_vec], topn=len(train))

print('most similar: ')
print(train[sims[0][0]].words)

print('least similar:')
print(train[sims[len(train) - 1][0]].words)
```

    ['the', 'knights', 'an', 'orchestral', 'collective', 'led', 'by', 'an', 'open', 'minded', 'spirit', 'of', 'camaraderie', 'and', 'collaboration', 'join', 'pianist', 'aaron', 'diehl', 'an', 'artist', 'quietly', 're', 'defining', 'the', 'lines', 'between', 'jazz', 'and', 'classical', 'onstage', 'at', 'the', 'martin', 'theatre', 'the', 'group', 'performs', 'excerpts', 'from', 'jazz', 'titan', 'mary', 'lou', 'williams', 'zodiac', 'suite', 'on', 'the', 'tail', 'of', 'recording', 'this', 'joyous', 'enchanting', 'creation', 'the', 'guardian', 'that', 'publication', 'continued', 'that', 'the', 'moods', 'crammed', 'into', 'each', 'sign', 'three', 'minutes', 'are', 'wonder', 'the', 'playing', 'and', 'on', 'pisces', 'operatic', 'singing', 'inspired', 'triumph', 'the', 'orchestra', 'presents', 'the', 'ravinia', 'premiere', 'of', 'louise', 'farrenc', 'finale', 'allegro', 'from', 'symphony', 'no', 'in', 'minor', 'and', 'concludes', 'the', 'program', 'with', 'beethoven', 'symphony', 'no', 'pastoral']
    most similar:
    ['the', 'evening', 'is', 'filled', 'entirely', 'with', 'ravinia', 'premieres', 'as', 'world', 'class', 'clarinetist', 'kinan', 'azmeh', 'and', 'genre', 'defying', 'pianist', 'dinuk', 'wijeratne', 'join', 'chamber', 'orchestra', 'far', 'cry', 'at', 'the', 'martin', 'theatre', 'the', 'program', 'opens', 'with', 'syrian', 'american', 'composer', 'kareem', 'roustom', 'dabke', 'for', 'string', 'orchestra', 'piece', 'exploring', 'folk', 'dance', 'from', 'palestine', 'syria', 'and', 'lebanon', 'that', 'is', 'typically', 'performed', 'at', 'joyous', 'occasions', 'the', 'musicians', 'continue', 'with', 'two', 'pieces', 'illustrating', 'the', 'clarinetist', 'and', 'pianist', 'rich', 'history', 'of', 'collaboration', 'azmeh', 'ibn', 'arabi', 'postlude', 'and', 'wijeratne', 'clarinet', 'concerto', 'czech', 'composer', 'leoš', 'janácek', 'is', 'also', 'in', 'the', 'spotlight', 'as', 'the', 'concert', 'wraps', 'with', 'his', 'idyll', 'for', 'string', 'orchestra']
    least similar:
    ['founded', 'in', 'australia', 'in', 'powerhouse', 'rock', 'group', 'crowded', 'house', 'catalogue', 'of', 'hits', 'includes', 'don', 'dream', 'it', 'over', 'weather', 'with', 'you', 'better', 'be', 'home', 'soon', 'and', 'fall', 'at', 'your', 'feet', 'vocalist', 'guitarist', 'and', 'songwriter', 'neil', 'finn', 'has', 'led', 'the', 'band', 'through', 'multiple', 'line', 'ups', 'and', 'artistic', 'iterations', 'and', 'has', 'consistently', 'proven', 'his', 'knack', 'for', 'crafting', 'high', 'quality', 'songs', 'that', 'combine', 'irresistible', 'melodies', 'with', 'meticulous', 'lyrical', 'detail', 'allmusic', 'the', 'band', 'appears', 'for', 'the', 'first', 'time', 'ever', 'at', 'ravinia', 'just', 'out', 'of', 'the', 'studio', 'with', 'their', 'new', 'single', 'oh', 'hi']


As an unsupervised model, `doc2vec` has no easy way to calculate test accuracy. One proxy is making sure that the genre of a randomly sampled document from test lines up with the genre of the most similar document from train, and that the genre is not close to the least similar document from train.


```python
print(df_test.iloc[test_doc_idx]['genre'])
print(df_train.iloc[sims[0][0]]['genre'])
print(df_train.iloc[sims[len(train) - 1][0]]['genre'])
```

    classical
    classical
    rock



```python
# bio from Wikipedia
stirling_doc = simple_preprocess('Lindsey Stirling (born September 21, 1986) is an American violinist, songwriter, and dancer. She presents choreographed violin performances, in live and music videos found on her official YouTube channel, which she created in 2007. Stirling performs a variety of music styles, from classical to pop and rock to electronic dance music. Aside from original work, her discography contains covers of songs by other musicians such as Johann Sebastian Bach, Ludwig van Beethoven, Wolfgang Amadeus Mozart, and Antonio Vivaldi and various soundtracks.')
print(stirling_doc)
stirling_doc_vec = model.infer_vector(stirling_doc)

sims_stirling = model.dv.most_similar([stirling_doc_vec], topn=len(train))

print('most similar: ')
print(train[sims_stirling[0][0]].words)

print('least similar:')
print(train[sims_stirling[len(train) - 1][0]].words)
```

    ['lindsey', 'stirling', 'born', 'september', 'is', 'an', 'american', 'violinist', 'songwriter', 'and', 'dancer', 'she', 'presents', 'choreographed', 'violin', 'performances', 'in', 'live', 'and', 'music', 'videos', 'found', 'on', 'her', 'official', 'youtube', 'channel', 'which', 'she', 'created', 'in', 'stirling', 'performs', 'variety', 'of', 'music', 'styles', 'from', 'classical', 'to', 'pop', 'and', 'rock', 'to', 'electronic', 'dance', 'music', 'aside', 'from', 'original', 'work', 'her', 'discography', 'contains', 'covers', 'of', 'songs', 'by', 'other', 'musicians', 'such', 'as', 'johann', 'sebastian', 'bach', 'ludwig', 'van', 'beethoven', 'wolfgang', 'amadeus', 'mozart', 'and', 'antonio', 'vivaldi', 'and', 'various', 'soundtracks']


    most similar:
    ['faculty', 'members', 'of', 'the', 'ravinia', 'steans', 'music', 'institute', 'piano', 'strings', 'program', 'come', 'together', 'for', 'an', 'afternoon', 'of', 'chamber', 'music', 'with', 'violinist', 'midori', 'taking', 'the', 'stage', 'for', 'her', 'first', 'season', 'as', 'the', 'program', 'artistic', 'director', 'she', 'is', 'joined', 'by', 'violinist', 'mihaela', 'martin', 'violist', 'kim', 'kashkashian', 'cellists', 'frans', 'helmerson', 'and', 'clive', 'greensmith', 'and', 'pianist', 'marc', 'andré', 'hamelin', 'on', 'the', 'program', 'is', 'ludwig', 'van', 'beethoven', 'string', 'trio', 'no', 'timo', 'andres', 'piano', 'trio', 'commissioned', 'by', 'rsmi', 'in', 'carlos', 'simon', 'where', 'two', 'or', 'three', 'are', 'gathered', 'and', 'robert', 'schumann', 'piano', 'quintet', 'in', 'flat', 'major']
    least similar:
    ['appearing', 'in', 'acclaimed', 'broadway', 'productions', 'of', 'the', 'sound', 'of', 'music', 'the', 'book', 'of', 'mormon', 'and', 'parade', 'singer', 'and', 'actor', 'ben', 'platt', 'first', 'rose', 'to', 'prominence', 'in', 'the', 'title', 'role', 'of', 'dear', 'evan', 'hansen', 'that', 'role', 'earned', 'him', 'the', 'tony', 'award', 'for', 'best', 'actor', 'in', 'leading', 'role', 'in', 'musical', 'making', 'him', 'the', 'youngest', 'ever', 'recipient', 'at', 'that', 'time', 'he', 'has', 'recorded', 'two', 'studio', 'albums', 'sing', 'to', 'me', 'instead', 'and', 'reverie', 'and', 'appears', 'this', 'summer', 'following', 'the', 'release', 'of', 'his', 'latest', 'single', 'with', 'sara', 'bareilles', 'grow', 'as', 'we', 'go']



```python
df_ravinia[df_ravinia['descrip'].str.contains('Midori')]
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>band</th>
      <th>date</th>
      <th>descrip</th>
      <th>genre</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>59</th>
      <td>Midori</td>
      <td>2024-07-07</td>
      <td>Faculty members of the Ravinia Steans Music In...</td>
      <td>classical</td>
    </tr>
  </tbody>
</table>
</div>



# t-SNE

I create a t-SNE model to visualize the word vectors. t-SNE is a dimensionality reduction technique that maps high-dimensional data to a two-dimensional plane.

The results look okay but not the best. For example, some genres cluster together on the bottom (funk/folk/hip hop). However, there don't seem to be clear patterns in general, which may be due to the small size of the corpus. This small size is also why I chose a lower perplexity (4, which is unusual for t-SNE).


```python
# Create a t-SNE model for the word vectors -- stored in model[i]
wv_idx = list(model.wv.key_to_index.values())
wv_list = np.array([model[i] for i in wv_idx])
wv_labels = list(model.wv.key_to_index.keys())

counts = {}
for token in model.wv.key_to_index.keys():
    counts[token] = model.wv.get_vecattr(token, 'count')
```


```python
from sklearn.manifold import TSNE

tsne_model = TSNE(perplexity=4, n_components=2)
tsne_values = tsne_model.fit_transform(wv_list)
```


```python
xs = []
ys = []

for i in range(0, len(tsne_values)):
    xs.append(tsne_values[i][0])
    ys.append(tsne_values[i][1])

fig, ax = plt.subplots(figsize=(25, 25))
ax.scatter(xs, ys, c='#69b3a2')

for label, x, y in zip(wv_labels, xs, ys):
    ax.annotate(label, xy=(x, y))

fig.savefig(output_dir / 'ravinia_2024_tsne.png')
```

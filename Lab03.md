# Lab session #3: Extracting and Analyzing data from the Cloud


This Lab assignment is built on top of the previous ones to discuss a basis for extracting appealing terms from a dataset of tweets while we “keep the connection open” and gather all the upcoming tweets about a particular event.

* [Task 3.1: Realtime tweets API of Twitter](#Tasks31)
* [Task 3.2: Analyzing tweets - counting terms](#Tasks32)  
* [Task 3.3: Case study](#Tasks33)  
* [Task 3.4: Student proposal](#Tasks34)  

#  Tasks for Lab session #3

<a name="Tasks31"/>

## Task 3.1: Real-time tweets API of Twitter
In case that we want to “keep the connection open”, and gather all the upcoming tweets about a particular event, the [Real-time tweets API](https://developer.twitter.com/en/docs/tweets/filter-realtime/overview) is what we need.

The Real-time tweets APIs gives developers low latency access to Twitter’s global stream of Tweet data. Proper implementation of a streaming client pushes messages indicating that Tweets and other events have happened.

Connecting to the real-time tweets API requires keeping a persistent HTTP connection open. In many cases, this involves thinking about your application differently than if you were interacting with the REST API. Visit the [Real-time tweets API](https://developer.twitter.com/en/docs/tweets/filter-realtime/overview) for more details about the differences between Realtime tweets and REST APIs.

The real-time tweets API is one of the preferred ways of receiving a massive amount of data without exceeding the rate limits. If you intend to conduct singular searches, read user profile information, or post Tweets, consider using the REST APIs instead.

We need to extend the `StreamListener()` class to customize the way we process the incoming data. We are going to base our explanation on a working example (from [Marco Bonzanini](https://marcobonzanini.com/2015/03/02/mining-twitter-data-with-python-part-1/)) that gathers all the new tweets with the "Artificial Intelligence" content:

```python
import os
import tweepy
from tweepy import OAuthHandler
from tweepy import Stream
from tweepy.streaming import StreamListener

class MyListener(StreamListener):

    def on_data(self, data):
        try:
            with open('ArtificialIntelligenceTweets.json', 'a') as f:
                f.write(data)
                return True
        except BaseException as e:
            print("Error on_data: %s" % str(e))
        return True

    def on_error(self, status):
        print(status)
        return True

consumer_key = os.environ['CONSUMER_KEY']
consumer_secret = os.environ['CONSUMER_SECRET']
access_token = os.environ['ACCESS_TOKEN']
access_secret = os.environ['ACCESS_SECRET']

auth = OAuthHandler(consumer_key, consumer_secret)
auth.set_access_token(access_token, access_secret)

twitter_stream = Stream(auth, MyListener())
twitter_stream.filter(track=['ArtificialIntelligence'])
```

The core of the streaming logic is implemented in the `MyListener` class, which extends `StreamListener` and overrides two methods: `on_data()` and `on_error()`.

These are handlers that are triggered when new data is coming through, or the API throws an error. If the problem is that we have been rate limited by the Twitter API, we need to wait before we can use the service again.

When data becomes available `on_data()` method is invoked. That function stores the data inside the  `ArtificialIntelligenceTweets.json` file. Each line of that file corresponds to a single tweet, in the JSON format. You can use the command `wc -l ArtificialIntelligenceTweets.json` from a Unix shell to check how many tweets you’ve gathered.

Before continuing this hands-on, make sure that you have correctly generated the `.json` file.

Once your program works correctly, you can try finding another term of your interest.

<a name="Tasks32"/>

## Task 3.2:  Analyzing tweets - Counting terms

We can leave the previous program running in the background fetching tweets while we write `TwitterAnalyzer.py`, on a different Python file, our first exploratory analysis: a simple word count. We can, therefore, observe which are the most commonly used terms in the dataset.

Let's then go and read the file with all tweets to be sure that everything is correct:

```python
import json

with open('ArtificialIntelligenceTweets.json','r') as json_file:
         for line in json_file:
             tweet = json.loads(line)
             print(tweet["text"])
```

We are now ready to start to tokenize all these tweets:

```python
import json

with open('ArtificialIntelligenceTweets.json', 'r') as f:
    line = f.readline()
    tweet = json.loads(line)
    print(json.dumps(tweet, indent=4))
```
Now, if we want to process all our tweets, previously saved on file:

```python
with open('ArtificialIntelligenceTweets.json', 'r') as f:
    for line in f.readlines():
        tweet = json.loads(line)
        tokens = preprocess(tweet['text'])
        print(tokens)
```
Remember that `preprocess` has been defined in the previous Lab session in order to capture Twitter-specific aspects of the text, such as #hashtags, @-mentions and URLs.:

```python
import re

emoticons_str = r"""
    (?:
        [:=;] # Eyes
        [oO\-]? # Nose (optional)
        [D\)\]\(\]/\\OpP] # Mouth
    )"""

regex_str = [
    emoticons_str,
    r'<[^>]+>', # HTML tags
    r'(?:@[\w_]+)', # @-mentions
    r"(?:\#+[\w_]+[\w\'_\-]*[\w_]+)", # hash-tags
    r'http[s]?://(?:[a-z]|[0-9]|[$-_@.&+]|[!*\(\),]|(?:%[0-9a-f][0-9a-f]))+', # URLs

    r'(?:(?:\d+,?)+(?:\.?\d+)?)', # numbers
    r"(?:[a-z][a-z'\-_]+[a-z])", # words with - and '
    r'(?:[\w_]+)', # other words
    r'(?:\S)' # anything else
]

tokens_re = re.compile(r'('+'|'.join(regex_str)+')', re.VERBOSE | re.IGNORECASE)
emoticon_re = re.compile(r'^'+emoticons_str+'$', re.VERBOSE | re.IGNORECASE)

def tokenize(s):
    return tokens_re.findall(s)

def preprocess(s, lowercase=False):
    tokens = tokenize(s)
    if lowercase:
        tokens = [token if emoticon_re.search(token) else token.lower() for token in tokens]
    return tokens
 ```
In order to keep track of the frequencies while we process the tweets, we can use `collections.Counter()` which internally is a dictionary (term: count) with some useful methods like `most_common()`:

 ```python
import operator
import json
from collections import Counter

fname = 'ArtificialIntelligenceTweets.json'
with open(fname, 'r') as f:
    count_all = Counter()
    for line in f.readlines():
        tweet = json.loads(line)
        # Create a list with all the terms
        terms_all = [term for term in preprocess(tweet['text'])]
        # Update the counter
        count_all.update(terms_all)
    print(count_all.most_common(5))

```
As you can see, the above code produces words (or tokens) that are stop words. Given the nature of our data and our tokenization, we should also be careful with all the punctuation marks and with terms like `RT` (used for re-tweets) and `via` (used to mention the original author), which are not in the default stop-word list.

```python
import nltk
from nltk.corpus import stopwords
nltk.download("stopwords") # download the stopword corpus on our computer
import string

punctuation = list(string.punctuation)
stop = stopwords.words('english') + punctuation + ['rt', 'via', 'RT']

```

We can now substitute the variable `terms_all` in the first example with something like:

```python
import operator
import json
from collections import Counter

fname = 'ArtificialIntelligenceTweets.json'
with open(fname, 'r') as f:
    count_all = Counter()
    for line in f.readlines():
        tweet = json.loads(line)
        # Create a list with all the terms
        terms_stop = [term for term in preprocess(tweet['text']) if term not in stop]
        count_all.update(terms_stop)
    for word, index in count_all.most_common(5):
        print ('%s : %s' % (word, index))
```

Besides stop-word removal, we can further customize the list of terms/tokens of our interest.
For instance, if we want to count *hashtags* only by using:

```python
terms_hash = [term for term in preprocess(tweet['text'])
              if term.startswith('#')]
```
In the case that we are interested in only counting terms, no hashtags and no mentions:

```python
terms_only = [term for term in preprocess(tweet['text'])
              if term not in stop and
              not term.startswith(('#', '@'))]
```
> Mind the double brackets (( )) at `startswith()`. It takes a tuple (not a list).

Although we do not consider it in this Lab session, there are other convenient NLTK functions. For instance, to put things in context, some analysis examines sequences of two terms. In this case, we can use `bigrams()` function that takes a list of tokens and produces a list of tuples using adjacent tokens.

Create a program at `TwitterAnalyzer.py` that reads **only once** the `.json` file generated by the listener process and prompts:

- a list of the top ten most frequent tokens
- a list of the top ten most frequent hashtags
- a list of the top ten most frequent terms, skipping mentions and hashtags.

**Q32: copy and paste the output of the program to `README.md`**


<a name="Tasks33"/>

## Task 3.3:  Case study

At "[Racó (course intranet)](https://raco.fib.upc.edu/)" you can find a small dataset as an example (please do not distribute due to Twitter licensing). This dataset contains 1060 tweets downloaded from around 18:05 to 18:15  on January 13. We used "Barcelona" as a `track` parameter at `twitter_stream.filter` function. Save the file in your folder but **do not add it to the git repository**.

We wanted to get a rough idea of what was telling people about Barcelona. For example, we could count and sort the most commonly used hashtag:

```python
fname = 'Lab3.CaseStudy.json'
with open(fname, 'r') as f:
    count_all = Counter()
    for line in f.readlines():
        tweet = json.loads(line)
        # Create a list with all the terms
        terms_hash = [term for term in preprocess(tweet['text']) if term.startswith('#') and term not in stop]
        count_all.update(terms_hash)
# Print the first 10 most frequent words
print(count_all.most_common(15))

```
The output is (ordering may differ):
`[(u'#Barcelona', 68), (u'#Messi', 30), (u'#FCBLive', 17), (u'#UDLasPalmas', 13), (u'#VamosUD', 13), (u'#barcelona', 10), (u'#CopaDelRey', 8), (u'#empleo', 6), (u'#BCN', 6), (u'#riesgoimpago', 6), (u'#news', 5), (u'#LaLiga', 5), (u'#SportsCenter', 4), (u'#LionelMessi', 4), (u'#Informe', 4)]`

To see a more visual description we can plot it. There are different options to create plots in Python using libraries such as [**matplotlib**](https://matplotlib.org/) or [**ggplot**](http://ggplot.yhathq.com/). We have decided to use matplotlib and the following code:

```python
import matplotlib as mpl
mpl.rcParams['figure.figsize'] = (6,6)
import matplotlib.pyplot as plt

sorted_x, sorted_y = zip(*count_all.most_common(15))
print(sorted_x, sorted_y)

plt.bar(range(len(sorted_x)), sorted_y, width=0.75, align='center')
plt.xticks(range(len(sorted_x)), sorted_x, rotation=60)
plt.axis('tight')

plt.show()                  # show it on IDE

plt.savefig('CaseStudy.png')     # save it on a file
```

We obtain the following plot:

<p align="center"><img src="./images/Lab03-Plot.png " alt="lab03-Plot" title="lab03-Plot"/></p>

**As an insight**, we can see that people were talking about football, more than other things! Moreover, it seems that they were mostly talking about the football [league match played the day after](http://www.fcbarcelonanoticias.com/Calendario-y-resultados-liga.php?IDR=184).

**Q33: Create a "matplot" with the dataset generated by you in the previous task.** Add the file `CaseStudy.png` to your repository.

<a name="Tasks34"/>

## Task 3.4:  Student proposal

We ask the student following this hands-on to create an example that presents compelling insights from Twitter, using some realistic data taken as explained above.

Using what you have learned in this and the previous Lab sessions you can download some data using the real-time tweets API, pre-process the data in JSON format and extract some curious terms and hashtags from the tweets.

Create a file named `StudentProposal.py` with the solution and add it to your repository.

**Q34: Write in `README.md` a brief description of your proposal and the compelling insights that you have gained on this topic.**

**Q35: How long have you been working on this session? What have been the main difficulties you have faced and how have you solved them?** Add your answers to `README.md`.


# How to submit this assignment:

Use the **private** repo named *https://github.com/CCBDA-UPC/2022-3-xx*. It needs to have, at least, two files `README.md` with your responses to the above questions and `authors.json` with both members email addresses:

```json5
{
  "authors": [
    "FIRSTNAME1.LASTNAME1@estudiantat.upc.edu",
    "FIRSTNAME2.LASTNAME2@estudiantat.upc.edu"
  ]
}
```

1. Link to your dataset created in task 3.4.
2. Add any comment that you consider necessary at the end of the 'README.md' file

Make sure that you have updated your local GitHub repository (using the `git`commands `add`, `commit` and `push`) with all the files generated during this session. 

**Before the deadline**, all team members shall push their responses to their private *https://github.com/CCBDA-UPC/2022-3-xx* repository.
---
layout: post
title: 'Monitoring Twitter For Illegal Wildlife Trade'
---

*This project is currently ongoing, and as such, is updated on a continual basis.*

**Tech used: data mining, PostgreSQL, natural language processing**

Conservation and conservation justice is a topic that is very dear to my heart. Endangered species are under constant threat from poachers, traffickers, and sellers of “luxury” or “medicinal” wildlife products. The trade of trafficked wildlife and flora is moving increasingly online in an effort to globalize black markets and social media acts a valuable advertisement network. By monitoring and removing wildlife trafficking content, traffickers lose a crucial element of their online business and can be quickly identified and apprehended, both of which would serve as an incredible victory in the fight against illegal wildlife trade. This project is an attempt at making a small contribution to this fight. I expect this to be a very difficult project: in the paper linked below, very few positive hits were found (43 out of 133,584 tweets – 0.0003% of all tweets!). This coupled with the fact that this is my first attempt at a natural-language-processing-oriented ML project, success might be difficult to come by. But overcoming the difficult challenges are what define us, and the reward for such a topic is too great to give up on. In short, the juice is well worth the squeeze. 

This project is inspired by the following paper by Xu, et al:
Citation: Xu Q, Li J, Cai M and Mackey TK (2019) Use of Machine Learning to Detect Wildlife Product Promotion and Sales on Twitter. Front. Big Data 2:28. doi: 10.3389/fdata.2019.00028

and by the following TRAFFIC report by Xiao Yu and Wang Jia:
Moving targets: Tracking online sales of illegal wildlife products in China, available at (https://www.traffic.org/site/assets/files/2536/moving_targets_china-monitoring-report.pdf)

Having a firm grasp on the product I wanted to build, the first step was to outline how that product would take shape. To begin, I’m going to focus on monitoring Twitter for suspicious tweets relating to the sale of ivory and will move on to other social media and ecommerce platforms such as eBay and Instagram. I will use the Tweepy API to stream, search, filter, and extract various tweets. Since there are (to my knowledge) no publicly available datasets similar to this project, the tweets extracted using Tweepy will form my entire dataset, and, if successful, will be open sourced for others to use. A database is required to store my tweets – for this I will be using PostgreSQL. Tweets will be loaded from the database into a Pandas DataFrame and analyzed via Natural Language Processing (NLP) techniques. The key technique used here is the Biterm Topic Model (BTM). BTM was developed to deal with the sparsity problem in short texts, where traditional topic modelling techniques such as Latent Dirichlet Allocation (LDA) typically struggle. This difficulty lies in the fact that techniques like LDA model word-document cooccurrences (how often a word occurs in a document), while BTM models word-word cooccurrences. This is useful, since, for example, tweets will often consist of < 20 or so words – making a word-document model unhelpful. [Check this paper to read about the details of BTM.](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.402.4032&rep=rep1&type=pdf) After the BTM step, various clusters, or topics, will be formed, from which should be able to determine whether tweets in this topic cluster is related to wildlife trafficking or the sale of illegal wildlife products.

I want to extract two sets of features – user information and tweet information. Important user information consists of the User ID, the User’s location, and, as a way to measure a User’s potential reach, the User’s follower count. Tweet data will consist of the tweet text itself, the number of times it was retweeted, the tweet date, and, if possible, any images which were attached to the tweet.

#### Designing the database
Knowing that there are two types of fundamental data I want to store, two tables were created – a Users table “twitter_users” and a Tweet table “twitter_tweets.” While designing the database, the Tweepy documentation was consulted to choose the appropriate datatypes. The User ID and tweet ID serve as the primary key for the twitter_users table and the twitter_tweets table, respectively, with the User ID acting as the foreign key of the twitter_tweets table. Another important consideration is how to store images attached to tweets. Storing the images in the database itself would result in tremendously long query times. Therefore, rather than storing the images themselves, links pointing to the name and location of the image on disk is stored in the “images” column. Finally, the psycopg2 library is used to interact with PostgreSQL via Python. 

#### Natural Language Processing 
Before analyzing the tweets, there is a lot of preprocessing that needs to be done. This preprocessing can be broken down into the following steps:

   1.	**Cleaning up the tweet**
   This will consist of removing hyperlinks, usernames, emojis, invalid characters (for example, removing ‘ö’,’ü’,’ä’, etc. when examining English language tweets), stop-words, and punctuation marks. This step requires making use an external library such as nltk and genism or one can make extensive use of regular expressions. I chose to practice using regular expressions to clean up the tweets, but we’ll use the nltk package later, don’t worry.

   2.	**Bringing the remaining tweet text into a “normal” format**
   This includes making all characters lowercase and then either stemming or lemmatizing all words. Lemmatizing refers to the process of reducing words to their base form. An example of this is changing the words “running” and “ran” to “run” and changing plural nouns to their singular equivalent – “animals” to “animal.”

   3.	**Creating the biterms themselves**
   This step is comprised of generating a word-document matrix and extracting the biterms from the word-document matrix.
   
There’s a little too much code to show off here, so visit the project repository to take a look at the respective functions.

#### Streaming vs Searching using the Tweepy API
  
Tweepy allows users to access tweets using two different methods – reading in a continuous stream of tweets in near-real time filtered by keywords or by searching and filtering tweets, also via keywords. Both have their respective advantages and disadvantages, and both techniques were applied in this project.

The key advantage of the streaming API lies in the number of tweets that are collected each second. It’s very easy to collect thousands of tweets in very little time. However, tweets are limited to what is being tweeted *right at that moment.* There is no way to access what was being tweeted yesterday, or even 15 minutes before you started the stream. This issue is somewhat remedied by using the search API. Using the search API allows one to search and view tweets from up to one week ago – and can even access tweets from the year 2006(!) – provided that one is using the *premium, paid* API. As a free tier user, one week ago isn’t much, but it’s better than being limited to right here and now. Unfortunately though, according to the Tweepy documentation… “Please note that Twitter’s search service and, by extension, the Search API is not meant to be an exhaustive source of Tweets. Not all Tweets will be indexed or made available via the search interface.” …we still won’t see all relevant tweets. Bummer. I think I remember reading somewhere that only 20% of tweets are available when using the search API, but maybe I’m making that up, because I’m not able to find this reference again. Hmm. 
So, with this information in mind, I thought it best to try both and hope for the best. There aren’t really any significant differences in the implementation – the only difference really is list of keywords with which the tweets are filtered. This will be touched on more in the following section.

#### Streaming tweets with the streaming API
Two sets of keywords were developed – a set of general keywords (‘sale’, ‘selling’, ‘price’) and ivory specific keywords. In total, 18 keywords were used, compared to 17 keywords in the original aforementioned paper. 
```
GENERAL_KEYWORDS = ['sale', 'selling', 'price']
IVORY_KEYWORDS = ['ivory', 'horn', 'tusk', 'carved', 'mammoth', 'antique',
            'trophy', 'bone', 'medicine', 'medicinal', 'authentic', 'decorative']
```

After streaming the tweets to my database, the results are…well, interesting to say the least:

{% include image.html image="projects/proj-3/raw_tweets_stream.png" %}

Not *extremely* relevant. There are plenty of tweets that have one or more of the keywords in their text. However, there are so *many* tweets that just shouldn’t be there. The main culprits seem to be the general keywords:

{% include image.html image="projects/proj-3/barplot_stream_final.png" %}

“Medicine” and “trophy” are the only ivory keywords to even make it into the top 20 most frequent words. We can also check out the co-occurrence matrix to see how often certain word pairings occur together. This matrix below should be read as “How often a word in a column occurs when a word in a row is present, and vice-versa.” 

{% include image.html image="projects/proj-3/cooccurence_matrix_stream_final.png" %}

Not a strong showing for the cooccurrence matrix either. The tweets really are dominated by the noise – noise being general commerce tweets. I don’t think the general keywords should be left out entirely. A good idea might be to add some additional logic to include at least one general keyword AND at least one ivory keyword – something to keep in mind when using the search API. For now, though, let’s keep chugging along and see what we come up with. 

#### Topic modelling

After cleaning up and “standardizing” the tweets (see points (1.) and (2.) in the NLP section above), we’re ready to begin constructing biterms to fit our model. The first step in extracting biterms, as is typical in many NLP applications, is to form a word-document matrix. The word-document matrix is a sparse representation that counts the number of times a word (token) appears in a document and looks a lot like one-hot encoding. Each row in the word-document matrix represents a single document in the whole corpus and each column represents a single word in the entire corpus. The word-document matrix will be an MxN matrix, where M is the number of documents in the corpus and N is the number of unique words in the corpus. Scikit-learn has a great function for this (as usual) so I’ll be using that to create the word-document matrix.
```
def generate_word_document_matrix(corpus: pd.Series) -> np.array:
    
    try:
        CountVectorizer
    except: 
        from sklearn.feature_extraction.text import CountVectorizer
    
    # take only the words that appear in less than 95% of tweets and words that appear at least 5 times
    x = CountVectorizer(max_df = 0.95, min_df = 5)
    wd_matrix = x.fit_transform(corpus).toarray()
    vocab = x.get_feature_names()
    
    return wd_matrix, vocab
```

With the word-document matrix created, we can then look at each index of words per document and combine them to form our biterms. The biterms and total vocabulary are passed to the model, which creates the topics/clusters. Let’s take a look.

{% include image.html image="projects/proj-3/btm_stream_cluster1.png" %}

Hmm…there are many overlapping clusters right in the center with a few distinct topics spread out along mostly the first principal component. There don’t seem to be any obvious clusters that could be related to wildlife trafficking either. Unfortunately, this isn’t much of a surprise. The quality of the tweets doesn’t seem to be great… and combined the with knowledge that the original paper reported *extremely* noisy data with very few positive IDs for wildlife trafficking, it was always going to be a bit of a stretch. Maybe there will be better results if we try again with the search API.

#### Extracting tweets using the search API
It turns out, there are several new challenges when using the search API. The first major challenge is the query complexity limit. I’m able to implement the logic that requires at least one ivory keyword AND at least one general keyword; however, the query is now too complex for the Tweepy API – instead of fetching the tweets, an error is returned. After several attempts, I finally found a query that fit within the complexity limits, albeit with a much smaller list of keywords. The total number of keywords equals only 11 compared with the previously used 18. There are quite a few notable missing keywords here, specifically “mammoth”, which was identified as an important keyword in the original paper. 
```
ELEPHANT_KEYWORDS = ['ivory', 'tusk', 'carved', 'antique',
            'trophy', 'bone', 'medicine', 'authentic', 'decorative']
GENERAL_KEYWORDS = ['sale','selling']
```

The second key challenge when using the search API is limited range of tweets. The search API allows one to fetch a subset of relevant tweets from within the last week. However, if the same search is run a second time within the same week, the same tweets are returned. This means there will be duplicate tweets in the database. The only way I know to avoid this is to wait one week between each mining session. At the time of writing, the database has only 6,189 tweets, which is hardly enough for even simple tasks. This has slowed progress to a crawl, unfortunately, and I’m forced to work with a significantly smaller dataset. Despite these shortcomings, I will carry on with the analysis in hopes of finding a true signal. Looking again at the most common words and their co-occurrences: 

{% include image.html image="projects/proj-3/barplot_search_final.png" %}
{% include image.html image="projects/proj-3/cooccurence_matrix_search_final.png" %}

And after applying the same NLP techniques and passing the biterms to our BTM model just as with the streaming API, we get the following clusters:

{% include image.html image="projects/proj-3/btm_search_20k_cluster5.png" %}

Again, we see a high concentration of topics around the center, with the more distinct clusters (and most of the variance) along the first principal component. Topic 5 seems to be the most closely related cluster to the potential sale of ivory – but closer inspection shows that this topic is most likely about *banning* the sale of ivory. Such a small dataset likely warrants fewer topics, which is the probable reason that there are so many overlapping clusters. Let’s see how the topic distribution looks with half the number of topics.

{% include image.html image="projects/proj-3/btm_search_cluster5.png" %}

This already seems a bit more promising judging strictly by the topic distribution. Cluster 2,3, and 5 all have high occurrences of ivory-specific keywords and general commerce keywords. But a more thorough review of the tweets showed that there were positive hits of ivory sales. 

#### Next steps
A reduced number of keywords seemed to be more effective for the search API. This could also produce more favorable results for the streaming API. Furthermore, the more robust logic of at least one ivory-specific keyword and at least one general keyword was an improvement, and again, applying this logic to the streaming API should result in higher quality tweets. The next iteration of this project will include streaming tweets for a significant amount of time with the aforementioned improvements. 

#### Final thoughts
I’m thrilled that no instances of ivory sales were found. This genuinely makes me very happy. However, manual searches of Twitter do find instances of ivory trade – likely the antique legal variety. So, it’s a bit distressing that the searches or the streams didn’t find *any* instance of ivory sales. At the same time, it’s reasonable to think that Twitter has its own checks, and these checks catch any illegal sales. It should also be noted that only English-language tweets were examined. As mentioned in the Xu paper, there should be investigations made into tweets in other languages, particularly Asian languages, where there is an enormous black market for such items. That being said, this project does not end here. I’ll continue to update this post as I make advances.

Until next time.

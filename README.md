# Awesome Twitter Algo :bird:

Curated by [Igor Brigadir](https://github.com/igorbrigadir) and [Vicki Boykis](https://github.com/veekaybee).

An annotated look through the release of the Twitter algorithm, through the context of engineering and recsys, with notes from repo creators on significance of specific parts of the code. Since it can be hard to parse through so much code and derive meaning and context, we do it for you!

This code focuses on the services used to build the Home timeline `For You` feed, the algorithmic tab that is now served first on both web and mobile next to the   `Following` feed. 

<img width="591" alt="Screenshot 2023-03-31 at 9 36 04 PM" src="https://user-images.githubusercontent.com/3837836/229259504-fd08c5f5-a346-4e6a-b7d0-2f5514e02915.png">

# Contributing

We're happy to take changes that add and contextualize Twitter's recommendations algorithm as it's been released over the past week. To contribute, please submit a PR with good formatting and grammar and lots of links to references where relevant. We're especially happy for feedback from tweeps or former tweeps who can tell us where we got it wrong. 

 # High-level Context and Summary
One thing that's immediately obvious is that this is not the entire codebase or even a working majority of it. Missing from this codebase are 

1) many flows that process,enrich, and refine model input data
2) YAML configuration metafiles which could tell us quite a bit about how the code actually works. There are only 7 of them, the rest have been redacted.
3) Most code related to spinning up the actual infrastructure
4) Git commit history that shows us how some of this code has evolved
5) A large portion of the trust and safety codebase, which Twitter [has noted they've left out for now](https://github.com/twitter/the-algorithm/tree/main/trust_and_safety_models#trust-and-safety-models)

An important high-level concept discussed in the Spaces releasing this code was in-network and out-of-network. In-network tweets are those from people you follow, out-of-network is everyone else. A blend of 50%/50% are offered in the daily ~1500 tweets run through rankers. 

# Code Links

What was released? The majority of the code and algorithms, but not the data or parameters or configurations or build tools of the Recommneder Systems behind "For You" timeline recommendations. The Candidate Retrieval code was also not released, and neither was the Trust and Safety components, and the Ads components - those remain closed off. No User Data or credentials were inside the repositories and code comments were sanitized (or at least, none were obviously there on first look).

[Twitter Algo Repo](https://github.com/twitter/the-algorithm) || [Twitter ML Algo Repo](https://github.com/twitter/the-algorithm-ml) || [Blog Post](https://blog.twitter.com/engineering/en_us/topics/open-source/2023/twitter-recommendation-algorithm)

# Recsys Architecture Diagram

![twitter architecture@2x (1)](https://user-images.githubusercontent.com/3837836/229310537-620628de-7a81-4ced-b34f-94eb3f3e2370.png)

[Link to update here.](https://whimsical.com/twitter-archtecture-PoR7TJb1eac2UofLVSY28e)


# Twitter Data-Centric Historical Architecture Context 

There is a very, very old post from 2013 on [High Scalability](http://highscalability.com/blog/2013/7/8/the-architecture-twitter-uses-to-deal-with-150m-active-users.html) which gives some context to how these systems were initially constructed. 

As context, Twitter initially ran all workloads on-prem but has been [moving to Google Cloud.](https://cloud.google.com/blog/products/data-analytics/how-twitter-modernized-its-data-processing-with-google-cloud). In 2019, Twitter began by migrating to BigQuery and DataFlow from a data and analytics perspective. Before the move to BigQuery, [much of the data was stored in HDFS using Thrift](https://blog.twitter.com/engineering/en_us/topics/infrastructure/2022/scaling-data-access-by-moving-an-exabyte-of-data-to-google-cloud). It currently lives in BigQuery and is processed for many of the pipelines described below using [DataFlow](https://cloud.google.com/dataflow), GCP's Spark/Scalding-processing equivaent platform.  

 # Programming Languages and Frameworks
 
 The released code comes in a variety of languages. The most common languages used at Twitter are: 

 ## Java
 
 + Used in Lucene for search indexing
 + Used in the [GraphJet](https://github.com/twitter/GraphJet) library


 ## Scala
 + [Scalding](https://github.com/twitter/scalding), written at Twitter, pre-cursor to Spark
 + [Scio](https://github.com/spotify/scio) for parallel cluster computing computations

 ## Python
 + Python for the machine learning models in the stack, includes both PyTorch and Tensorflow (legacy) code

 ## Frameworks and Metalanguages

 + Bazel for [building Scala and Java services](https://bazel.build/)
 + Starlark for [bazel configuration](https://github.com/bazelbuild/starlark)
  + [Thrift](https://thrift.apache.org/), a cross-platform framework for RPC calls originally developed at Facebook(Meta)
  + [Hadoop](https://blog.twitter.com/engineering/en_us/topics/infrastructure/2023/kerberizing-hadoop-clusters-at-twitter) - Twitter still runs one of the largest installs of Hadoop out there

# Internal Libraries 

+ [Finagle](https://twitter.github.io/finagle/) is a service written in Scala with Java and Scala APIs used to manage RPCs
+ Snowflake is [a service that generates unique identifiers](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake) for each tweet based on timestamp, worker number, and sequence number

# Recsys

![recsys](https://user-images.githubusercontent.com/3837836/229260535-27c3bcc6-403b-4d71-b301-f381b0b1be33.png)


The typical recommender system pipeline has four steps: candidate generation, ranking, filtering, and serving. Twitter has many pipelines for performing verious parts of this this across the overall released codebase. 

+ **Candidate generation** occurs when you have millions or billions of potential items in your source data based on user-item interactions. This piece usually includes collaborative filtering or neural algorithms to reduce the size of the candiate dataset for downstream tasks.  
+  These need to then be **ranked** against each other, **filtered** against business logic and blended and served to the user in a given surface area, in this case the `For You` feed in the Twitter timeline. 

## Input Data

+ The system starts with [500 million tweets posted on a daily basis](https://blog.twitter.com/engineering/en_us/topics/open-source/2023/twitter-recommendation-algorithm). 

The [input data](https://blog.twitter.com/engineering/en_us/topics/infrastructure/2021/processing-billions-of-events-in-real-time-at-twitter-) 
comes from: 

+ Kafka
+ Twitter Eventbus
+ GCS
+ Vertica
+ [Manhattan](https://blog.twitter.com/engineering/en_us/a/2014/manhattan-our-real-time-multi-tenant-distributed-database-for-twitter-scale), a real-time multitenant distributed database that was initially developed as a serving layer on top of Hadoop and includes both observability and other metrics.


<img width="841" alt="Screenshot 2023-04-01 at 3 26 24 PM" src="https://user-images.githubusercontent.com/3837836/229310405-a4839079-bb77-427e-bfe8-4cebd1d1a6af.png">

In migrating to GCP, the current data ingest looks something like this: 
+ Streaming Dataflow jobs to apply deduping
+ Perform real-time aggregation and sink data into BigTable

That data is then made available to the candidate generation phase. There is not much about the actual data, even what a schema might look like, in the repo. 

## Candidate Generators 
(also called "features" in the chart)

### GraphJet
--- 
+ [GraphJet](https://github.com/twitter/GraphJet) - A realtime Java graph processing library that allows for in-memory processing on a single server and focuses on providing content recommendations. [Paper here.](http://www.vldb.org/pvldb/vol9/p1281-sharma.pdf) Recommendations are provided based on shared interests, correlated activities, and a number of other input signals.  GraphJet maintains a [realtime bipartite interaction graph](https://mathworld.wolfram.com/BipartiteGraph.html) that  keeps track of user–tweet interactions over the most recent n hours and reads from Kafka. Each individual GraphJet serever can ingest one million graph edges per second and compute 500 recommendations/second. 

They describe the reasons specifically for creating an in-memory DB in the GraphJet paper: 

<img width="445" alt="Screenshot 2023-04-02 at 12 21 06 PM" src="https://user-images.githubusercontent.com/3837836/229365611-72b263d6-bc7e-4089-8e23-fe8ac68e1d6a.png">

The precursor to GraphJet was WTF, Who to Follow, which focused [only on recommending users to other users.](https://web.stanford.edu/~rezab/papers/wtf_overview.pdf), using [Cassovary, an in-memory graph processing engine](https://github.com/twitter/cassovary) built specifically for WTF, also built on the JVM. 

<img width="447" alt="Screenshot 2023-04-02 at 12 22 15 PM" src="https://user-images.githubusercontent.com/3837836/229365667-96855c3a-6238-4138-bdd7-faf396d8397e.png">

GraphJet implements two random walk algorithms:  
 + Circle of Trust (internal to Twitter) and 
 + SALSA (Stochastic Approach for Link-Structure Analysis). 

<img width="423" alt="Screenshot 2023-04-02 at 12 38 16 PM" src="https://user-images.githubusercontent.com/3837836/229366503-3f860693-6dfd-4755-90c4-e67c85964700.png">
GraphJet Architecture

+A large portion of the traffic to GraphJet comes from
clients who request content recommendations for a partic-
ular user.

Graphjet includes [CLICK, FAVORITE, RETWEET, REPLY, AND TWEET as input node types](https://github.com/twitter/the-algorithm/blob/ec83d01dcaebf369444d75ed04b3625a0a645eb9/src/scala/com/twitter/recos/graph_common/NodeInfoHandler.scala#L22) and keeps track of left (input) and right(output) nodes. 

### SimClusters
--- 


### TwHIN
--- 

### RealGraph
--- 


### TweepCred
--- 



At the end of the candidate generation phase, 1500 Tweets are available for serving to your feed. 

+ The largest candidate generator is [Earlybird](https://blog.twitter.com/engineering/en_us/a/2011/the-engineering-behind-twitter-s-new-search-experience), a Lucene based real-time retrieval engine. There is an [Earlybird paper.](http://notes.stephenholiday.com/Earlybird.pdf) 



## Rankers

+ **Recap** The "Heavy Ranker" is a [parallel masknet](https://arxiv.org/abs/2102.07619). Majority of the code for this is in the [ML repo](https://github.com/twitter/the-algorithm-ml/blob/main/projects/home/recap/README.md). The [ranker itself](https://github.com/twitter/the-algorithm-ml/blob/main/projects/home/recap/README.md) is run after the candidate generators. 

[Input features](https://github.com/twitter/the-algorithm-ml/blob/main/projects/home/recap/FEATURES.md) 

Outputs are predictions on how user will respond to the tweet: 
+ probability the user will favorite the Tweet
+ probability the user will click into the conversation of this tweet and reply or like a Tweet
+ probability the user will click into the conversation of this Tweet and stay there for at least 2 minutes.
+probability the user will react negatively (requesting "show less often" on the Tweet or author, block or mute the Tweet author) 
+ probability the user opens the Tweet author profile and Likes or replies to a Tweet
+ probability the user replies to the Tweet
+ probability the user replies to the Tweet and this reply is engaged by the Tweet author 
+ probability the user will click Report Tweet 
+ probability the user will ReTweet the Tweet
+ probability (for a video Tweet) that the user will watch at least half of the video

All of these are combined and weighted into a score. Hyperparameters for the model [and weighting are here.](https://github.com/twitter/the-algorithm-ml/blob/78c3235eee5b4e111ccacb7d48e80eca019e480c/projects/home/recap/config/local_prod.yaml#L1)

For more details on the model, see the [Architecture overview](masknet.md).

+ The Light Ranker


## Filters

+ Coming into this stage from the light ranker, there are other heuristics that are [used to filter out more tweets after scoring.](https://github.com/twitter/the-algorithm-ml/blob/main/projects/home/recap/README.md)

+ Remove [out-of-network competitor site URLs](https://github.com/twitter/the-algorithm/blob/main/home-mixer/server/src/main/scala/com/twitter/home_mixer/functional_component/filter/OutOfNetworkCompetitorURLFilter.scala) from potential offered candidate Tweets

## Timeline Mixer

+ The timeline mixer has a [ratio where](https://github.com/twitter/the-algorithm/blob/7f90d0ca342b928b479b512ec51ac2c3821f5922/home-mixer/server/src/main/scala/com/twitter/home_mixer/param/HomeGlobalParams.scala#L89) verified blue checkmark tweets are offered twice as more if they're out-of-network and four times as more if they're in-network. 


## Business Terms and Logic

These are Twitter specific terms and names that keep coming up across different code bases and blog posts.

+ Twepoch - A "magic number" `1288834974657L`, which is a timestamp for `2010-11-04T01:42:54Z` the date that Twitter introduced the Snowflake ID system, used as Twitter's own ["Unix Epoch"](https://en.wikipedia.org/wiki/Epoch_(computing))
+ Snowflake - Twitter's system for [assigning unique IDs](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake) to tweets, users, lists, DMs, media etc. 
+ WTF - [Who to follow](https://web.stanford.edu/~rezab/papers/wtf_overview.pdf)
+ DDG - Duck Duck Goose, Twitter's [A/B Testing Platform](https://blog.twitter.com/engineering/en_us/a/2015/twitter-experimentation-technical-overview).
+ Earlybird - Twitter's Lucene based real-time search index. [Notes](https://stephenholiday.com/notes/earlybird/) and a [blog post here](https://blog.twitter.com/engineering/en_us/a/2011/the-engineering-behind-twitter-s-new-search-experience).
+ "Unregretted user minutes" - the metric Twitter publicly states is the thing they are optimizing for. It is unknown how exactly they measure this.

## Bias and Manipulation

Cases of potential bias, manipulation, favouritism, hacks, etc. The focus on this repository is on the concrete, techincal aspects of the code, not speculating on anything twitter may or may not have done. That exercise is left to the reader, however, there are some technical aspects that should still be described about these popular accusations, this is a section for those. Unfortunately, much of the configuration that would contain specific instances of interventions is not in the code.

### Deboosting Rival Sites

It was long speculated youtube links get massively deboosted, and Spaces links massively boost Tweets in recommendations. There are no specific references to this in the code. However, there are filters that could be configured for this, referencing [OutOfNetworkCompetitorURLFilter](https://github.com/twitter/the-algorithm/blob/ec83d01dcaebf369444d75ed04b3625a0a645eb9/home-mixer/server/src/main/scala/com/twitter/home_mixer/product/scored_tweets/ScoredTweetsRecommendationPipelineConfig.scala#L231) for example.

### Elon Musk feature

The Elon Musk / Democrat / Republican Code: [Now Removed](https://github.com/twitter/the-algorithm/commit/ec83d01dcaebf369444d75ed04b3625a0a645eb9). One of the first widely shared cases, falsely assuming this is something that directly affects Recommendations when it was actually for internal A/B testing, to monitor for effects (DDG is Duck Duck Goose, the A/B Testing Platfom). It was also mentioned in [the space](https://twitter.com/elonmusk/status/1641880448061120513) and denied there. However, a former twitter employee also offered an [alternative explanation](https://twitter.com/igb/status/1641910616939167745) (A/B Testing measures behavior, so one way or another Twitter is tuning your TL, indirectly).

### Ukraine
There are two mentions related to Ukraine in the Twiter Algo repo. Whereas [one of them](https://github.com/twitter/the-algorithm/blob/7f90d0ca342b928b479b512ec51ac2c3821f5922/visibilitylib/src/main/scala/com/twitter/visibility/rules/PublicInterestRules.scala#L54) is a flag for Ukraine-related misinformation used for moderation, or warning labels, there is another _safety label_ for Twitter Spaces called _[UkraineCrisisTopic](https://github.com/twitter/the-algorithm/blob/7f90d0ca342b928b479b512ec51ac2c3821f5922/visibilitylib/src/main/scala/com/twitter/visibility/models/SpaceSafetyLabelType.scala#L39)_. Here are some facts about these labels and their function:
  * Each safety label "[describes a particular policy violation, and usually leads to reduced visibility of the labeled entity in product surfaces](https://github.com/twitter/the-algorithm/blob/main/visibilitylib/README.md)"
  * `SafetyLabel` results in tweet interstitial or notice, are publicly [documented here previously](https://help.twitter.com/en/resources/addressing-misleading-info) and specifically for [Armed Conflicts here](https://help.twitter.com/en/rules-and-policies/crisis-misinformation).
  * All [other Twitter Spaces safety labels](https://github.com/twitter/the-algorithm/blob/7f90d0ca342b928b479b512ec51ac2c3821f5922/visibilitylib/src/main/scala/com/twitter/visibility/models/SpaceSafetyLabelType.scala#L26) are related to misinformation, NSFW, toxic or harmful content, DMCA takedowns, etc.

## Changes

+ 2 hours after it was released, [Twitter removed](https://github.com/twitter/the-algorithm/commit/ec83d01dcaebf369444d75ed04b3625a0a645eb9) feature flags that specifically higlighted Elon's account

## Resources for Learning More about Recsys

+ @karlhigley's [blog](https://practicalrecs.com/the-rest-of-the-owl.html) and [thread of threads](https://twitter.com/karlhigley/status/1138122648934916097) are very accessible things about Recommender Systems in practice.

+ A good Recommender Systems entry point is the Google Machine Learning for [Recommender Systems course](https://developers.google.com/machine-learning/recommendation), it also has a good [glossary](https://developers.google.com/machine-learning/glossary/recsystems) of terms.

+ The biggest academic recsys community is [ACM Recsys](https://recsys.acm.org/) and state-of-the-art recommender systems research is usually openly available in the [proceedings](https://dl.acm.org/conference/recsys). A lot of the presentations are on [youtube](https://www.youtube.com/@acmrecsys/videos).

+ Admittedly out of date, but still useful, is the [RecSys Wiki](https://www.recsyswiki.com/wiki/Main_Page).

+ The latest edition of the [Recommender Systems Handbook](https://link.springer.com/book/10.1007/978-1-0716-2197-4) is also a good book that covers the field well.

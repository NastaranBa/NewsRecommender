
#News4U: Recommend stories based on collaborative reader behavior

##Yuan Huang is an Insight alumnus from the New York Summer 2017 Data Science Fellowship. While at Insight, Yuan developed a news…

##News4U: *Recommend stories based on collaborative reader behavior*

[*Yuan Huang](https://www.linkedin.com/in/yuan-huang-ds/) is an Insight alumnus from the New York Summer 2017 Data Science Fellowship. While at Insight, Yuan developed a news recommendation engine based on collaborative user behavior. *

Online news reading has become very popular as the web provides access to news articles from millions of sources around the world. A critical problem is that the volumes of articles can be overwhelming to the readers. Therefore, building a news recommendation system to help users find news that are interesting to read is a crucial task for every online news service. 

News recommendations must perform well on fresh content: breaking news that hasn’t been viewed by many readers yet. Thus we need to leverage on the article content data available at publishing time, such as topics, categories, and tags, to build a content-based model, and match it to readers’ interests learnt from their reading histories. However, one drawback of the [*content-based recommendations](https://en.wikipedia.org/wiki/Recommender_system#Content-based_filtering)* is that when there’s not enough history about a user, the coverage of the recommendations will become very limited, which is the common [*cold-start](https://en.wikipedia.org/wiki/Cold_start)* problem in recommender systems.

This blog post introduces a news recommendation engine which combines [*collaborative-filtering](https://en.wikipedia.org/wiki/Recommender_system#Collaborative_filtering)* with [*content-based filtering](https://en.wikipedia.org/wiki/Recommender_system#Content-based_filtering) *to try to make news recommendations more diverse. This so-called [*hybrid-filtering](https://en.wikipedia.org/wiki/Recommender_system#Hybrid_recommender_systems)* recommendation system takes into account not only the content of the articles and the user’s reading history, but also the reading history of people who share similar interests. By learning from the history of people with similar interests, this engine will recommend news with a much broader coverage of topics, even when the history information about a particular user is very limited. [Give it a try!](http://www.yuanhuang.club/)

![](https://medium2.global.ssl.fastly.net/max/2464/1*afmPggKapPpZDXW6iRgEbw.gif)**

In this post, I’ll explain how I build the recommendation engine from the ground up. The discussions will include:

* Step 1: Finding readers with similar interests

* Step 2: Topic modeling

* Step 3: Making recommendations

* Step 4: Evaluation of the recommender

##Finding readers with similar interests

As the first step, the engine identifies readers with similar interests on news based on their retweeting behavior of news posted on twitter. The data was collected from news posted on twitter by three different publishers (New York Times, Washington Post, and Bloomberg) for half a month. The information of all the twitter users who are retweeting the articles are also requested from Twitter. By looking at how many news posts each two users share in common, we can define a [*cosine similarity](https://en.wikipedia.org/wiki/Cosine_similarity)* score for the users. Therefore a user network can be constructed by assigning the weight of link between two users to their similarity.

Applying [*hierarchical clustering algorithm](https://en.wikipedia.org/wiki/Hierarchical_clustering_of_networks)* to the user network, we can detect the community structures among the readers. The hierarchical clustering algorithm uses a [*greedy method](https://en.wikipedia.org/wiki/Greedy_algorithm)* to try to optimize the [*modularity](https://en.wikipedia.org/wiki/Modularity_(networks))* of clusters. The modularity is an important metric for network clustering, which indicates how dense the connections within clusters are compared to the connections between different clusters. The definition of modularity is as follows:

![Modularity score of a clustering in networks. In the equation, m is the total number of links in a network, v and w are indices of the nodes in the network (users), A is the link matrix between each pair of nodes (defined as the similarity score), kv is the total link weight that is connected to a specific node v, and cv is the group label of node v.](https://medium2.global.ssl.fastly.net/max/2000/1*YZ1p3NgerqqXRqUFiyV4KQ.png)*Modularity score of a clustering in networks. In the equation, m is the total number of links in a network, v and w are indices of the nodes in the network (users), A is the link matrix between each pair of nodes (defined as the similarity score), kv is the total link weight that is connected to a specific node v, and cv is the group label of node v.*

The modularity of a randomly connected graph is 0, while the modularity of a unconnected graph with each connected part as an individual cluster will be 1. Any real world network will have a modularity value between those two extreme cases. In our user network, the modularity score of the hierarchical clustering algorithm peaks at 6 clusters with value 0.151. 

![Community structure in the user network detected from the clustering algorithm. Each point represents a user, with colors representing their group label. Due to visualization limitation, in the graph I only show the users with more than 30 retweets. The length of links between users are inversely related to the similarity between them. More similar users will be pulled closely together while dissimilar users will be pushed further apart.](https://medium2.global.ssl.fastly.net/max/2000/1*A1wo7nrxqUcWVrzO47VmPA.png)*Community structure in the user network detected from the clustering algorithm. Each point represents a user, with colors representing their group label. Due to visualization limitation, in the graph I only show the users with more than 30 retweets. The length of links between users are inversely related to the similarity between them. More similar users will be pulled closely together while dissimilar users will be pushed further apart.*

With the help of [D3](https://d3js.org/) visualization package, I plotted the community structure in the subset of readers with a high number of retweets. In the graph, users with high similarities are pulled closely with each other and less similar users will be pushed further apart. Some groups (for example the red, purple, green and light blue groups) show a very tight structure with extremely high similarities within the group, while the other groups are in a more diffuse shape, indicating more diverse interests among the group members. Now we identified several user groups with high similarities, but how do we know what kinds of news do they like to read?

##Topic modeling

In order to understand the topics of news articles, I used a natural language processing tool called [*Latent Dirichlet Allocation](https://en.wikipedia.org/wiki/Latent_Dirichlet_allocation) *(LDA)* *that allows computers to identity hidden topics of documents based on the cooccurrence frequency of words collected from those documents. LDA can also help find out how much of an article is devoted to a particular topic, which allows the system to categorize an article, for instance, as 50% environment and 40% politics.

I trained the LDA model on the texts of more than 8,000 articles collected using a package [newspaper](http://newspaper.readthedocs.io/en/latest/). The number of topics was chosen by trying to achieve a diverse topic coverage without having too many topics. The diversity of topics can be evaluated by the average [*Jaccard similarity](https://en.wikipedia.org/wiki/Jaccard_index)* between topics. Jaccard index measures similarity between two finite sample sets, and is defined as the size of the* [intersection](https://en.wikipedia.org/wiki/Intersection_(set_theory))* divided by the size of the [*union](https://en.wikipedia.org/wiki/Union_(set_theory))* of the sample sets. High Jaccard similarity indicates strong overlap and less diversity between topics, while low similarity means the topics are more diverse and have a better coverage among all the aspects in the articles.

![The average Jaccard similarity between each pairs of topics. The overlap between topics drops rapidly when increasing the number of topics from 5 to 20, and starts to saturate after 20 topics.](https://medium2.global.ssl.fastly.net/max/2000/1*ZAOG8mgMAGhy7CzgMrMzeg.png)*The average Jaccard similarity between each pairs of topics. The overlap between topics drops rapidly when increasing the number of topics from 5 to 20, and starts to saturate after 20 topics.*

The following figure shows the probability distributions of all the articles among the 20 topics learnt from LDA model. There are several hot topics with high probability among all the articles we collected, which include politics, finance, technology, environment, etc.

![Topic distribution among 20 topics learnt from LDA in all the articles. Topics with high probability are about politics(1), finance(2), technology(3), and environment(4).](https://medium2.global.ssl.fastly.net/max/2000/1*GSd8XDgE8wW4RvVzExRizQ.png)*Topic distribution among 20 topics learnt from LDA in all the articles. Topics with high probability are about politics(1), finance(2), technology(3), and environment(4).*

The interests among different topics from user group can be learnt from the topics of articles with a high number of retweets by the readers in the group. By aggregating the topics of each article weighted by the number of retweets, we obtained the topic probability distribution for all the six user groups. The following figure shows two examples, the blue and green groups, which show totally different interest upon different topics.

![Topic distribution among 20 topics learnt from LDA for user group 0 and group 3. Group 0 (blue) shows more interests in finance and technology topics, while group 3 (green) has more interests in politics.](https://medium2.global.ssl.fastly.net/max/2000/1*wt06WmPuwDj7ujG_sc8miQ.png)*Topic distribution among 20 topics learnt from LDA for user group 0 and group 3. Group 0 (blue) shows more interests in finance and technology topics, while group 3 (green) has more interests in politics.*

Another way to visualize the different topic distributions is by looking at the keywords in each group. Here I show two examples of user group 0 (blue in the network) and user group 3 (green).

![Word clouds generated from the topic distributions of different user groups. Here I show two examples, group 0 (left) which has more interests on finance, and group 3 (right) with more focused interest on politics.](https://medium2.global.ssl.fastly.net/max/2000/1*zVR4uR_jqAaT_qs4-b3VfA.png)*Word clouds generated from the topic distributions of different user groups. Here I show two examples, group 0 (left) which has more interests on finance, and group 3 (right) with more focused interest on politics.*

##Making recommendations

Now that we divided the users into different groups based on their similarity and identified their interests among different topics, the next step is to build a content-based news articles recommendation by matching the topics of the fresh news with the topic profile of each user group. In other words, our recommendation engine doesn’t provide personalized recommendations sorely based on a particular user’s interest, but instead gives group-based recommendations in order to obtain a more diverse result. 

When recommending new articles to a user group, we want to find articles that have the most similar topics with the group’s interests. A similarity score between each new article and the group is calculated as the [*cosine similarity](https://en.wikipedia.org/wiki/Cosine_similarity)* of their topic distributions, which is defined as follows:

![Cosine similarity between two topic distribution profiles, where A is the vector of topic distributions in the article, B is the vector of topic distributions of the user group.](https://medium2.global.ssl.fastly.net/max/2000/1*2OMuc-1Uc9lzpUjHQFau9A.png)*Cosine similarity between two topic distribution profiles, where A is the vector of topic distributions in the article, B is the vector of topic distributions of the user group.*

![Demonstration of the cosine similarity between each article and the user groups.](https://medium2.global.ssl.fastly.net/max/2076/1*S-24ZTp2MCwr6OLR4nXcqw.png)*Demonstration of the cosine similarity between each article and the user groups.*

Ranking by the similarity scores, the best-matching articles will be recommended to all the readers within the reader group.

##Evaluation of the recommender

Okay, that was cool. But how do I know whether the recommendation engine is working well or just spitting out random selections? How much will the readers like the recommendations that they get? In fact, the evaluation of a recommending system can be quite tricky. The golden metric for a recommending system is how much the system will add value to the reader and business. Ultimately, you want to perform A/B testing to see whether recommendations will increase usage, subscriptions, clicks, etc.

However, in practice there are other common metrics to evaluate a recommender, which can still help us gain some insights of the performance before actually putting the system into use. Most of these offline methods need us to hold out a subset of the items from the training data set (in our case, holding out a subset of previously retweeted news posts), pretending the users haven’t seen these items and trying to recommend them back to the users. Since the reader groups’ history about this test set already exists, we can leverage on this information to validate the performance of recommender.

One natural goal of recommender systems is to distinguish good recommendations from bad ones. In the binary case, this is very natural — a “1” is a good recommendation, while “0” means a bad recommendation. However, since the data that we have (number of retweets of an article by users from a user group) is non-binary, a threshold must be chosen such that all ratings above the threshold are good and called “1”, while the rest are bad with label “0”. A natural way to set the threshold is to choose the median value of the number of retweets in a user group, leaving all the articles above median “good” recommendations and all the rest “bad”. The predicted score from the recommending system is the cosine similarity between the topics of an article and the topics in a user group, which ranges from 0 to 1.

This good/bad, positive/negative framework is the same as binary classification in other machine learning settings. Therefore, standard classification metrics could be useful here. Two basic metrics are [precision and recall](https://en.wikipedia.org/wiki/Precision_and_recall). In our project, precision is fraction of good recommendations we got correct, out of all the articles got recommended by the system. Recall is the fraction good recommendations we got correct, out of all the “good” articles in the test set. 

All these metrics are clear once we have defined the threshold of good/bad in our predictions. For instance, in a binary situation our labels are 0, 1 while our predictions are continuous from 0–1. To compare the predictions, we select a threshold (here we choose the median value of all the predictions), above which we call predictions 1 and below 0. From 10 rounds of validations with 1000 hold-out articles, the average precision score of our recommending system is 65.9%, while the average recall is 66.5%.

The choice of the threshold is left to the user, and can be varied depending on desired tradeoffs (for a more in-depth discussion, see [this blog](http://blog.insightdatalabs.com/visualizing-classifier-thresholds/)). Therefore to summarize classification performance generally, we need metrics that can provide summaries over this threshold. One tool for generating such a metric is the [*Receiver Operator Characteristic (ROC)* ](https://en.wikipedia.org/wiki/Receiver_operating_characteristic)curve, which plots the [*True Positive Rate (TPR)](https://en.wikipedia.org/wiki/Sensitivity_and_specificity)* versus the [*False Positive Rate (FPR)](https://en.wikipedia.org/wiki/Sensitivity_and_specificity) *at different threshold levels. The area under the curve (often referred to as the AUC) indicates the probability that a classifier will rank a randomly chosen positive instance higher than a randomly chosen negative one, which can provide some insight on the predictive power of our model.

![The receiver operating characteristic curve for different user groups calculated from 10 rounds of validations on 1000 hold-out articles. The average area under curve among all the user groups is 0.77.](https://medium2.global.ssl.fastly.net/max/2000/1*DZJ0gFBcBcV-ZCcGI-S5pA.png)*The receiver operating characteristic curve for different user groups calculated from 10 rounds of validations on 1000 hold-out articles. The average area under curve among all the user groups is 0.77.*

##Summary & What’s Next

I learned a lot about developing a data science project from start to finish during this project. and gave me a chance to apply my technical skills to a real-world business problem.

The framework of this recommendation engine is quite versatile: it can be adapted to combine data from different platforms (Twitter, Facebook, news websites) to construct the user network; if combined with more personal information from the users, the system can also be developed to analyze user behavior and help making better marketing and advertising decisions.

Want more details? Check out the recommender’s code on [github](https://github.com/huangy22/NewsRecommender).

---
layout: post
title: Lessons from a Kaggle Competition
---

I'm currently in the final week of the [Liberty Mutual Group: Property Inspection Prediction](https://www.kaggle.com/c/liberty-mutual-group-property-inspection-prediction) competition on Kaggle, and it seems a good time to reflect on some of takeaways from my initial foray into the world of competitive data analysis. 

Before diving in, I'll mention that I used Python 3 throughout my work, though R seems to work fine for many people as well. In the competition forums, nearly everyone seems to use one of the two. 

### The Competition

The competition is to predict "Hazard Scores" for properties based on features that are available before an actual inspection takes place. This can help determine which properties truly require an inspection (and perhaps how thorough of one) before moving forward with insuring it. The competition has been a perfect opportunity for exploring data analysis for a couple reasons. 

First off, presumably for liability reasons, Liberty Mutual has thoroughly masked all variables, so there is essentially no opportunity for domain knowledge or other experience in the field to help. Without any knowledge of what a feature represents, all contestants are on equal footing using only the abstract numbers to drive their analysis. 

Secondly, the size and format of the dataset makes it very approachable. Some of Kaggle's competitions feature huge amounts of data (over 100x the size of this set), which can require access to more powerful computers or knowledge of special frameworks that specialize in "big data". At 50,000 rows of 32 features, my laptop could handle nearly all my modeling needs. I did use some beefier Amazon EC2 spot instances to speed things up at times, but even so my total expense will be less than $15. Given better planning (running longer analyses overnight, etc) and some code optimization, I could likely have gone without any outside resources at all. 

### The Gini Coefficient

One of the first things I noticed about the competition was the scoring metric. Entries are graded using a normalized Gini coefficient. I wasn't especially familiar with the Gini coefficient prior to this competition, so I started poking around to see how it worked and how that would affect results. 

Gini scoring's primary feature is that an entry's actual results don't matter, only the order. So, for example, if the real values are [1, 1, 2, 5, 10], then the entry [1, 2, 3, 4, 5] and [500, 600, 1000, 5000, 7000] would both receive a perfect score, because their ordering is the same as the correct ordering. 

However, even though the magnitude of predictions doesn't matter, the magnitude of actual values does. In other words, given the previous example of real values of [1, 1, 2, 5, 10], a guess of [1, 2, 1, 3, 4] would receive only a mild correction, since the out of order examples (1 and 2) are close to each other. A guess of [1, 1, 2, 6, 4], despite similarly having only two out-of-place entries, would be penalized much more, because the mixup of 5 and 10 is greater. 

So correctly ordering higher scored values is more important than correctly ordering lower scored ones. I spent some time looking for ways to leverage this, but (spoiler!) ultimately didn't succeed. Common methods of error measuring like mean squared error already give greater weight to higher magnitude misses and are a good start, but I believe an algorithm that properly optimized for this metric could see significant gains over standard error measuring. I considered building one, but I joined the competition with only a couple weeks remaining and didn't want to sink too much time into one approach, especially since I entered primarily to explore more of the data analysis ecosystem. 

### Data Modification

When approaching the training set, I first had to deal with the categorical variables. Given that variables had already been masked / shifted, it seemed unlikely ordinal values would be changed to categorical instead of discrete, so I used a OneHotEncoder for most of my analyses. In one-hot-encoding, a categorical variable with n options gets transformed into (n-1) binary variables. Checking against an ordinal-based Label Encoder, which turns a categorical variable into an ordinal one, confirms the decision in nearly all models. 

Some in the competition forums claimed modest success adding new features like average Hazard Score, skew, and similar "meta" metrics for a given variable, but I haven't had time to explore that route. 

To check for feature interaction and quadratic effects, I ran sci-kit learn's Polynomial processor on the data. I should note that this would be terribly impractical in a dataset with more features, and testing added variables one or two at a time would be far more efficient. To pare down the ballooning dataset after category encoding and polynomial effects, I cross-referenced Random Forest and Gradient Boosting feature importances, linear model's coefficients, and the results from a Boruta analysis (which also uses Random Forest feature importance, in a gradual elimination way over many interations). (Note: the boruta analysis was extremely slow and one of the few parts that actually required a specialized EC2 instance, where it required several hours). This allowed me to prune down the many engineered features into just the useful ones. 

One potential solution to the Gini problem was to modify the dataset to emphasize higher scores. I used the [UnbalancedDataset](https://github.com/fmfn/UnbalancedDataset) package to try a few methods to give greater weight to higher Hazard Scores. It seemed a multi-tiered SMOTE (progressively adding more of the heigher scores) over-sampler showed the most promise, but still performed roughly equal to or slightly worse than the original training data across all models. However, the fact that approximately doubling the sample size didn't affect performance says to me that there are probably gains to be made if I could figure out the right combination. A custom scoring metric to train models on would be ideal. 


That's enough for one post! 


**Addendum:** To my embarrassment, I got busy and completely missed the end of the competition. So I never got to try out ensembling methods and make a strong final contender. However, my individual model scores seem to have done alright in comparison with what people are saying on the competition forum, so that's good news! Have to leave some room for improvement, right?

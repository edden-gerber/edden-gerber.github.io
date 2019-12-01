---
title: "An in-depth empirical study of Shapley Values and the SHAP library"
canonical_url: "https://edden-gerber.github.io/shapley-values/"
date: 2019-11-24
share: true
excerpt: "An empirical look into what we are asking when we ask about feature importance using Shapley values"
header:
  image: "assets/images/shapley/header.png"
  teaser: "assets/images/shapley/header.png"
toc: true
toc_sticky: true
---

## Why should you read this post?
* **If you want to better understand the SHAP python library and what it does**. There are numerous sources online that discuss Shapley values and the SHAP tool, but very few primary sources (like the SHAP documentation) that are not based on existing information. This post is an attempt to contribute fresh empirically-based insights on this topic.
* **If you ever waited too long for the SHAP kernel explainer to compute SHAP values for your entire dataset, and wonder if there might be a faster approach** (spoiler: _maybe_, if you have less than 15-20 features in your model - or see the end of the post for another theoretical suggestion).
* **If you want to learn about the concept of Shapley values** (there are plenty of other sources for that, but I include a [brief explanation](#what-are-shapley-values) here too).

**In a hurry?** I've emphasized the key sentences **in bold** to assist your speed-reading :)

## A more wordy introduction
If you ever looked for a way to make your complex model's results more explainable, you probably encountered the notion of Shapley values and the SHAP python library. It is an increasingly popular and theoretically robust approach method of quantifying how each feature contributes to a model's prediction for a particular sample. **While Shapley values can be highly expensive to compute, the SHAP library provides ways to sidestep this complexity. But is this a free lunch? And if not, how are we paying for it?**

My interest in this topic was sparked when I was using the SHAP library during a [recent hackathon](https://edden-gerber.github.io/datahack-2019/). I noticed that in the specific case we were working on, the SHAP code seemed to be quite inefficient - taking so long to run for the entire dataset, that I wondered whether in this case it would actually be faster to compute Shapley values using a direct, "brute force" approach. This led me to write a function that does exactly that, based on the direct, inefficient formulation of Shapley values - producing what I call "radical" Shapley values (since they are derived "from the root", that is, by re-training the model each time instead of relying on an existing model) - and test how it behaves compared to the SHAP library (and under which circumstances it might outperform it). The more valuable outcome of this study, however, was a much better understanding of Shapley values and the SHAP library.

**So the general idea of this post is to compare the SHAP methods to a more intuitive approach, in my opinion, to Shapley value derivation**, by looking at them empirically in different scenarios (one thing I learned as a scientist: if you want to really trust your analysis tools, you need to subject them to the same type of rigorous empirical study as your actual object of research). My main motivation is not to suggest a better alternative to the SHAP library (as I explain below, it is only "better" in some infrequent cases), but mostly to use this comparison to get a more nuanced understanding of this widely used method.

Code for the Shapley function and the examples used in this post is available [here](https://github.com/edden-gerber/radical-shapley-values).

## Outline (not quite a tl;dr)
If you need an introduction to Shapley values, I've added one at the [end of this post](#what-are-shapley-values) so as not to encumber the reading of those already familiar with the topic. A detailed description of my code is likewise included at the end. In this post I will try to show the following:

* **"Radical" Shapley values can be computed for a low number of features** by retraining the model for each of 2<sup>M</sup> feature subsets.
* **The SHAP library explainers and the radical Shapley method provide two different interpretations to Shapley values**, the former best suited for explaining individual predictions for a given (trained) model, and the latter better suited for explaining global feature importance for a given dataset and model class.
* **Under some (limited) circumstances, the direct Shapley computation can be faster than the SHAP library explainers.** These circumstances are broadly when you **a.** have a low number of features (<~15), **b.** are using a model that is not supported by the efficient SHAP explainers and which has a relatively low training/prediction run time ratio (such as Isolation Forest), and **c.** need to compute Shapley values for a large number of samples (e.g., the entire dataset). Another option which I may discuss in a future post is a compromise of polynomially-complex radical Shapley value estimation by coalition-sampling.


## So what are "radical" Shapley values
A Shapley value reflects the expected value of the surplus payoff generated by adding a player to a coalition, across all possible coalitions that don't include the player (or, in the machine learning realm, the expected value of the difference in model output generated by adding a feature to the model). However, implementing the concept of Shapley values for explaining predictive models is matter of some interpretation. Specifically:
* **the SHAP library interprets "adding a feature" in relation to its value being unknown, for a given sample, during the prediction phase**, while
* **the radical Shapley method is based on the alternative intuition of measuring a feature's impact in relation to it being absent from the model altogether during training**. As I didn't find a suitable term for this approach in the literature, **I chose the term "radical" to refer to the method going back to the "root" of the model** and re-training it for every iteration, rather than being based on an existing trained model. If a better term is out there I'd be happy to hear about it.

Both interpretations are consistent with the mathematical notion of Shapley values, but they measure slightly different things. The radical Shapley method is not an entirely new one, of course, nor are these the only two existing interpretations of Shapley values for machine learning. Finally to avoid possible confusion, note that _SHAP≠Shapley_ - it is an acronym of _SHapley Additive exPlanations_, e.g., an method based on Shapley values.

**The function for computing radical Shapley values (code [here](https://github.com/edden-gerber/radical-shapley-values)) takes a dataset and a payoff function, computes the payoff for each possible feature combination (or, "player coalition") and derives Shapley values** according to the formula:
{% include figure image_path="../assets/images/shapley/shapley-formula.png" alt="Shapley value formula" caption="_&phi;<sub>i</sub>_ is the Shapley value for feature _i_, _S_ is a coalition of features, _v(S)_ is the payoff for this coalition, and N is the total number of features. _N\\{i}_ is all the possible feature coalitions not containing _i_. The first term within the sum corresponds to the fraction of times _S_ appears within the possible feature permutations; intuitively, this gives the highest weight to the most informative contributions of a feature, i.e. when it is isolated or when it is added to a full set of features. " %}

**The payoff function can be any function that takes a dataset and returns a score** (for instance, the profit generated by a team of workers). It is thus a general function that can be used for any kind of Shapley computation, but for the purpose of generating radical Shapley values it will always be a function that trains a particular type of model on the dataset, and returns a prediction for each row. This means that while we specify the model parameters in advance within the function (e.g. number of trees in a random forest), the model is re-trained each time on a dataset containing a subset of features supplied by the Shapley function.

**For example**: let's say we want to compute radical Shapley values for a model that predicts _y_ using _x<sub>1</sub>_, _x<sub>2</sub>_, and _x<sub>3</sub>_ with XGBoost. We will write a custom payoff function that initializes an XGB model, trains it on input arguments _X_ and _y_ and returns a prediction for each sample (perhaps splitting them non-randomly into training/validation and returning predictions for the validation only). The Shapley function will feed the payoff function each possible feature combination in _X_ - {_x<sub>1</sub>_}, {_x<sub>1</sub>_,_x<sub>2</sub>_}, etc. - and use the scores to compute a Shapley value for each feature and each sample. The output is then the same as that of the SHAP library explainers, and so all the SHAP plotting tools can be used to visualize it.

**The main disadvantage of this algorithm is its computational complexity** - it needs to run 2<sup>M</sup> times (where _M_ is the number of features), re-training the model each time. As a rule of thumb, if model training takes 1 second and you don't want more than about an hour of run time, you shouldn't use this method when you have more than 12 features. This complexity is of course the main reason the SHAP library was needed; on the other hand, under some limited circumstances this may be a faster option than using the SHAP Kernel explainer. The issue of comparative run time is covered later in this post.


## What do the SHAP explainers do differently, and why should we care?
The SHAP library provides three main "explainer" classes - TreeExplainer, DeepExplainer and KernelExplainer. The first two are specialized for computing Shapley values for tree-based models and neural networks, respectively, and implement optimizations that are based on the architecture of those models. The kernel explainer is a "blind" method that works with any model. I explain these classes below, but for another in-depth explanation of how the SHAP classes work I recommend reading [this chapter](https://christophm.github.io/interpretable-ml-book/shap.html) of [Interpretable Machine Learning](https://christophm.github.io/interpretable-ml-book/) by Christoph Molnar.


### KernelExplainer
The kernel explainer differs from the radical Shapley method in two main ways:
1. **Instead of iterating over all 2<sup>M</sup> possible feature coalitions, it samples a small subset of them** (the default value is _nsamples=2*M+2048_. So if we have for example 20 features, then this will sample 2088/2<sup>20</sup> or about 0.2% of the possible coalitions). Coalitions are not selected completely randomly; rather, those with higher weight in the Shapley formula will be selected first (so, coalitions of 0 or M-1 features, which are most informative about the effect of adding  another feature, will be included first).
2. **Instead of re-training the model at each iteration, it runs the already-trained input model but replaces the values of "missing" features with randomly selected values** from other samples (a small "background" dataset). This additionally decreases total runtime because most models require significantly less time for prediction than for training.

**The significance of (1)** is that Shapley values are basically estimated using random sampling. Appropriately, increasing _nsamples_ (or the size of the background dataset) leads to increased runtime (more or less linearly) and decreased variance in the estimation (meaning that when you run the explainer multiple times, the Shapley values will be more similar to each other and to the expected value). This is easily demonstrated by running the Kernel Explainer with different _nsamples_ values:

{% include figure image_path="../assets/images/shapley/kernel_exp_nsamples.png" alt="SHAP kernel explainer runtime and estimation variance as a function of nsamples" caption="Based on the Census Income database included in the SHAP library. There are 12 features in the dataset and so _nsamples_ is effectively capped at 2<sup>12</sup>=4096. Estimation variance and mean run time are computed across 30 iterations of running the SHAP explainer on a K-Nearest-Neighbors classification model."  %}

**The significance of (2)** - handling "missing" features by replacing their values with surrogate "background" data - is more complex:
* First and most simply, it is an **additional source of variance** to the Shapley value estimation, as the results are to at least some extent dependent on the selection of the background dataset.
* Second, it makes an **assumption about feature independence**: if for example features _x<sub>1</sub>_, _x<sub>2</sub>_ are highly correlated in our training data, replacing values of  _x<sub>1</sub>_ with random ones from the background dataset ignores this dependence and generates predictions based on {_x<sub>1</sub>_, _x<sub>2</sub>_} instances which are not likely to appear in the training set, making the SHAP value estimation less reliable.
* And finally, it represents an **interpretation of feature contribution to the prediction** not as how they affect the content of the prediction model itself (as in the radical Shapley method that re-trains the model without the missing features), but rather how they affect the result of the already-trained model if the values of these feature were unknown. The outcome of this difference will become clearer in the next sections.

**Let's see how these points add up with an example**. We will look at Shapley values for one of the datasets included in the SHAP library - the [adult census database](https://archive.ics.uci.edu/ml/datasets/adult), with 12 demographic features that are used to predict whether an individual's income is >50K$. Following the [example of this SHAP library notebook](https://slundberg.github.io/shap/notebooks/Census%20income%20classification%20with%20scikit-learn.html), we will use a KNN model to make this prediction and the KernelExplainer to provide Shapley values - comparing them to those generated with the radical Shapley method. For clarity and reduced computation runtime, we'll include only 6 of the 12 features in our model (_Age_, _Hours per week_, _Education_, _Marital status_, _Capital gain_, and _Sex_).

```python
X,y = shap.datasets.adult()
X = X[['Age', 'Hours per week','Education-Num', 'Marital Status', 'Capital Gain', 'Sex']]
```

Let's define a model and use the KernelExplainer to get SHAP values (we'll compute them only for the first 1000 samples because the KernelExplainer is quite slow...):
```python
import shap
from sklearn.neighbors import KNeighborsClassifier
num_samples = 1000
knn = KNeighborsClassifier()
knn.fit(X, y)
f = lambda x: knn.predict_proba(x)[:,1] # Get the predicted probability that y=True
explainer = shap.KernelExplainer(f, X.iloc[0:100]) # The second argument is the "background" dataset; a size of 100 rows is gently encouraged by the code
kernel_shap = explainer.shap_values(X.iloc[0:num_samples])
```

Now let's compute radical Shapley values. First we'll define our payoff function, which trains the KNN model and outputs the probability of _y=True_ for each sample:
```python
def shap_payoff_KNN(X, y):
    knn = KNeighborsClassifier()
    knn.fit(X, y)
    return knn.predict_proba(X)[:,1] # this returns the output probability for y=True
```

And now run the function itself (_reshape_shapley_output_ just re-arranges the original output, since _compute_shapley_values_ returns a dictionary that does not assume a particular payoff format. An explanation of the function inputs and output is provided on the [github ReadMe doc](https://github.com/edden-gerber/radical-shapley-values).
```python
import numpy as np
from radical_shapley_values import compute_shapley_values
from radical_shapley_values import reshape_shapley_output
mean_prediction = np.mean(shap_payoff_KNN(X,y))
radical_shapley = reshape_shapley_output(compute_shapley_values(shap_payoff_KNN, X, y, zero_payoff=np.ones(X.shape[0])*mean_prediction))
radical_shapley = radical_shapley[0:num_samples] # compute_shapley_values returns Shaply values for all rows, so this is just to match the output of the kernel explainer.
```

<br>
Comparing radical and KernelExplainer Shapley values:
{% include figure image_path="../assets/images/shapley/kernel_vs_radical_scatter.png" alt="scatter plot" caption="" %}

The two method produce different but highly correlated results. Another way to summarize the differences is that if we sort and rank the Shapley values of each sample (from 1 to 6), the order would be different by about 0.75 ranks on average (e.g. in about 3/4 of the samples two adjacent features' order is switched). The discussion of the nature of these differences will wait until the next section, where they will be easier to understand. For now let's just remember that we are not looking at the relation between "true" values and their noisy estimation: instead, **_radical Shapley values are a deterministic measure of one thing, and the kernel SHAP values are a noisy estimation of another (related) thing_**.


### TreeExplainer
TreeExplainer is a class that computes SHAP values for tree-based models (Random Forest, XGBoost, LightGBM, etc.). Compared to KernelExplainer it is:
* **Exact**: Instead of simulating missing features by random sampling, it makes use of the tree structure by simply ignoring decision paths that rely on the missing features. The TreeExplainer output is therefore deterministic and does not vary based on a background dataset.
* **(Much) faster**: Instead of iterating over each possible feature combination (or a subset thereof), all combinations are pushed through the tree simultaneously, using a more complex algorithm to keep track of each combination's result - reducing complexity from _O(TL2<sup>M</sup>)_ to the polynomial _O(TLD<sup>2</sup>)_ (where _M_ is the number of features, _T_ is number of trees, _L_ is maximum number of leaves and _D_ is maximum tree depth).

This means that compared to the radical Shapley method, the TreeExplainer is similarly deterministic but typically much faster. On the other hand, **it still behaves in the same way as KernelExplainer in that it evaluates the contribution of features at the prediction phase, rather than the effect of adding them to the model itself** - and is therefore more suitable for explaining individual predictions given a trained model, than explaining how different features interact in the dataset.

**A simple artificial example can demonstrate this**. Let's generate a 3-feature linear regression model, with one feature _x<sub>1</sub>_ which is a strong predictor of _y_, a second feature _x<sub>2</sub>_ which is strongly correlated with it (and so slightly less predictive), and a third non-predictor feature _x<sub>3</sub>_:

```python
ns = 10000
nf = 4

X = pd.DataFrame(np.random.randn(ns, nf)) # 3 random features
X[[1]] = X[[0]] + np.random.randn(ns,1)*0.2 # x2 is correlated with x1 but noisier
y = X[[0]] + np.random.randn(ns,1) # y depends on x1 (and so on x2 as well) but not x3
X.columns = ['Strong predictor of y', 'Correlated with strong predictor', 'Not a predictor of y']
```
We'll define an XGB regressor model and compute TreeExplainer SHAP and radical Shapley values. How do they compare with each other? Let's use the SHAP library's _summary_plot_ visualization tool:
{% include figure image_path="../assets/images/shapley/artificial_data_summary_plots.png" alt="summary plots" caption="" %}

Let's break this down:
* With the TreeExplainer, the model has already been trained with all 3 features, so **SHAP values reflect the fact that _x<sub>1</sub>_ has the highest impact within the trained model**, while  _x<sub>2</sub>_ has a much smaller role as it is mostly redundant.
* With the radical Shapley method on the other hand, _x<sub>2</sub>_'s impact on _y_ is almost as large as _x<sub>1</sub>_'s because **when the model is trained without _x<sub>1</sub>_, _x<sub>2</sub>_ is nearly just as informative**.
* At the same time, the non-predictive  _x<sub>3</sub>_ is credited with a higher impact using the radical Shapley method - simply because, especially as we did not do a training/validation split, it can  over-fitted to the data in the absence of better predictors.


**Now let's try a real-world example**, using the previous case of the simple 6-feature census dataset predicting a >500K$ income. This time, we'll use an XGBoost classsifier as a model to allow the use of TreeExplainer. How do the TreeExplainer SHAP values and radical Shapley values compare with each other?

{% include figure image_path="../assets/images/shapley/treeexplainer_vs_radical_scatter.png" alt="scatter plot" caption="" %}

The results seem to be highly correlated but still slightly different. Again but with _summary_plot_:
{% include figure image_path="../assets/images/shapley/census_summary_plots.png" alt="summary plot" caption="" %}

Here too the results seem similar enough (although different enough that the order of global feature importance is changed a bit). But let's zoom in on the differences in how Shapley values for the _Sex_ variable are distributed:

{% include figure image_path="../assets/images/shapley/census_sex_shapley_hist.png" alt="histogram of sex feature Shapley values" caption="" %}

What's going on here?
* The radical Shapley results show us that **averaged all possible feature combinations, adding this variable will have a consistent impact on prediction for this dataset** (at least until we topple Patriarchy).
* The TreeExplainer results show us that **in our trained model, this variable has a smaller and less consistent impact on prediction across our samples**, most likely because it is used to explain smaller residual variance after most of the information it conveys was provided by other, more predictive features.

A benefit of implementing our own custom Shapley function is that we have easy access to a wealth of intermediate results - for example, the payoff margins that we calculated for each possible feature combination with vs. without a given feature (and whose weighted average is the Shapley value for each sample). Just for fun, I extracted it from the _compute_shapley_values_ function so we can have a look at how the final Shapley values arise from these individual payoff margins. These are the distributions of payoff margins for the _Sex_ variable, plotted against for the number of features to which it is added:

{% include figure image_path="../assets/images/shapley/sex_margin_dist.png" alt="histogram of sex feature Shapley values" caption="Each point is the prediction difference for a single sample caused by adding the feature to a specific feature combination. Color corresponds to feature value levels (female/male). There were originally more points in the middle rows due to more possible feature combinations, which was mitigated by random sub-sampling. " %}

We can see how the more features are already in the model, the impact of this feature becomes smaller and less bimodal (the bimodality we see in the previous histogram is therefore mostly driven by feature combinations from the upper rows). Contrast this to the distribution of margins for _Education_Num_ for example, whose contribution remains pretty much fixed no matter how many other features make up the model:

{% include figure image_path="../assets/images/shapley/education_margin_dist.png" alt="histogram of sex feature Shapley values" caption="" %}

Note that the gradual loss of impact we see for the _Sex_ feature is not something we'd expect to see if we had access to the corresponding data from the SHAP TreeExplainer, since in the SHAP case we are using the same trained model to compute feature impact for all feature combinations (while having "missing features" just means that this impact is, broadly speaking, averaged across possible values of those unknown features).

**So - which method should we use** to explain the role of the _Sex_ variable in our data and model? I think the best way to put this is:
* **Radical Shapley values better represent the global impact of the features in our dataset**, while
* **SHAP values better explain specific predictions given our existing trained model**.  

<br>

Two additional notes before moving on:
* **A practical note**: **I am not suggesting that if you care about global feature impact in your dataset you should necessarily use the radical Shapley method** - mainly because in most cases that would be computationally intractable (although see the next two sections for when it might not be). It is also apparent from my examples that the SHAP explainers' results are typically not so different that they should not be used for this purpose. My main motivation here is to get a better understanding of SHAP results and their limitations.

* **A technical note**: If you are familiar with TreeExplainer, you may know that since in the case of binary classification the weights of the tree nodes hold not probabilities but log-odd values (which are transformed into probabilities with the logistic function as a final step) - the default optimization approach utilized by TreeExplainer provides SHAP values that add up to these untransformed values (and not the final probabilities). Simply applying the logistic function to the SHAP values themselves wouldn't work, since the sum of the transformed values != the transformed value of the sum. To produce SHAP values that correspond directly to probability outputs, the TreeExplainer has to sacrifice some of its efficiency and use an approach similar to the KernelExplainer of simulating missing features by replacement with a background dataset - naturally a slower and less exact method. On the other hand, to directly explain probability outputs with the radical Shapley method all that we need to do is choose the output of the payoff function to be the probability. **Since with the radical Shapley method we always pay the maximum computation cost, we might as well use a payoff function that gives us exactly what we want our Shapley values to explain**.


### Wait, what about DeepExplainer?
DeepExplainer is typically used in models where the input size is relatively large (e.g. images with at least hundreds of pixels), and so a direct comparison with the radical Shapley method would require using edge cases of very small-input networks, which I think would be uninformative. However, like the KernelExplainer, the DeepExplainer uses a background dataset to estimate SHAP values for a trained model, and so similar conclusions about the nature of the computed Shapley values can be applied in this case - they vary (though not to a large extent) based on the selection of background data, they may not respect dependencies between features when generating the bootstrapped samples used for estimation, and their significance is in relation to the _trained model_, rather than to the _dataset and model class_.

## Why is KernelExplainer taking so long to run? Can the radical Shapley method be faster?

As we've established, the radical Shapely method needs to re-train the model and produce predictions 2<sup>num. features</sup> times. This makes it impractical whenever the number of features is not low (let's say, more than 15-20). But is it comparable to the SHAP explainers for a low number of features?

One thing to get out of the way is that **the optimized SHAP explainers will always be faster than the radical Shapley method, but the model-blind KernelExplainer can be very slow when explaining a large dataset**. To get a better idea, let's see what goes into the run time of the KernelExplainer vs the radical Shapley method:

|| In relation to... || The KernelExplainer run time is... || The radical Shapley run time is... ||
||---    ||: ---                 ||: ---            ||
|| Number of features   || fixed (aside from the model-dependent effect on prediction time) || **exponential** ||
|| Training time  || fixed || **linear** ||
|| Prediction time   || **linear (but computed ~200K times for each explained prediction)** || linear ||
|| Number of samples to explain || **linear** || fixed (aside from the model-dependent effect on train/predict time) ||

The bold entries emphasize the weaknesses of each method. The radical Shapley method is of course most vulnerable to increasing the number of features, and is also (linearly) slower with increased model training time, whereas KernelExplainer is not affected by these factors (although its prediction becomes more variable with increased number of features). The disadvantages of the KernelExplainer in terms of run time is that while it does not need to spend time re-training the model, it runs separately for each explained prediction (while radical Shapley runs for all predictions at once), each time having to produce predictions for ~200K samples (_nsamples_ X _num. background samples_, which by default are 2048+2M and 100, respectively).

**A good example for a model for which the radical Shapley method can perform faster than KernelExplainer, for a low number of features, is an _Isolation Forest_ model**, a popular tool for anomaly detection, as despite being a tree-based model it is not supported by TreeExplainer and its training time (compared to prediction) is relatively fast. To demonstrate this, I am using the [Credit Card Fraud Detection dataset from Kaggle](https://www.kaggle.com/mlg-ulb/creditcardfraud), a ~285K sample, 30 features dataset used to predict anomalous credit card transaction. For our demonstration let's use 100K samples and  reduce the 30 features down to 15. On my old laptop I get the following approximate run times: <br>
_Train the model_:                              **8 sec** <br>
_Make predictions for all 100K samples_:        **8 sec** <br>
_Compute SHAP values for a single prediction_:  **18 sec** (which is about the time it takes to make predictions for ~200K bootstrapped samples...)<br>

Based on this we can make a rough estimate of how long it would take to compute Shapley values for the entire dataset. The KernelExplainer should simply take 100K x 18 seconds, or **about 500 hours**. The radical Shapley function will run for up to 2<sup>15</sup> x (15+13) seconds, or **about 150 hours** (actually, a better estimate may be about 50 hours since the dataset used for training in each iteration of the algorithm will have 1-14 features, or 7 on average). So both methods are slow although both could also be implemented with parallelization... Anyway, what's important here is not the specific example but understanding where the computation time comes from in each case. **If you need to explain only a small group of "important" predictions, KernelExplainer should be fast enough. If you need to explain a million predictions and you have less than 10-15 features, the radical Shapley method should be much faster.**


## A hybrid solution? estimating radical Shapley values with random sampling
**Okay, but what if our model is not supported by TreeExplainer or DeepExplainer and we have too many features to compute radical Shapley values, but we really need Shapley values for our entire huge dataset? I believe a relatively simple solution exists in this case, which is to estimate radical Shapley values using a sampling approach**. Using random sampling to estimate Shapley values for a high number of players (as is done e.g. by the KernelExplainer) has been thoroughly discussed in the literature and improved methods are still being developed (see for example [Castro et al. 2009](https://www.sciencedirect.com/science/article/pii/S0305054808000804), [Castro et al. 2017](https://www.sciencedirect.com/science/article/pii/S030505481730028X) or [Benati et al. 2019](https://www.sciencedirect.com/science/article/abs/pii/S0377221719304448)). I would suggest that it can be beneficial to use sampling in combination with the radical Shapley approach - that is, to sample the space of feature combinations on which the model is trained (thus not needing to simulate missing features by averaging over bootstrapped samples).

The reason that this could be more efficient that using the KernelExplainer should be clear when looking at the table in the previous section: it eliminate the exponential computational complexity represented in the first row (by paying for it with increased estimation variance), leaving the dependence on training time the only computational disadvantage compared to the KernelExplainer, which in turn suffers from having to run the model a large number of times for every explained prediction. In the example given above, limiting the number of model re-training iterations to the same _nsamples_ parameters of the KernelExplainer would cut run time from 50-150 hours to about 10 hours (compared to KernelExplainer's ~500)- and crucially, **the sampling would assure that this would not increase significantly when increasing the number of features**

This is only a theoretical idea, and I would not make this post longer than it already is by developing it further. I would however be happy to get any comments you may about this (maybe you've already encountered this idea somewhere else?), and I'd probably be interested making another project out of it and report on in a future post.

**Have you made it this far?** I hope you found any part of this discussion helpful. The last section is a short explanation of Shapley values if you still need a clarification on this topic.


## Appendix: what are Shapley Values?
If you are familiar with the concept of Shapley values, you can skip this section. A quick Google search will also provide numerous other sources online that explain this topic very well (I found that staring at the mathematical formula on the Wikipedia page until I got it gave me the best intuition), so my explanation will be brief.

Shapley values are a concept from game theory, describing how the contribution to a total payoff generated by a coalition of players is divided across the players. The relevance of this concept to machine learning is apparent if you translate "payoff" to "prediction" and "players" to "features".
The definition of the Shapley value (using ML terminology) is quite straightforward: the contribution of any feature is the expected value, across all possible permutations of feature combinations not containing this feature, of the prediction change caused by adding this feature. It looks like this:

{% include figure image_path="../assets/images/shapley/shapley-formula.png" alt="Shapley value formula" caption="_&phi;<sub>i</sub>_ is the Shapley value for feature _i_, _S_ is a coalition of features, _v(S)_ is the payoff for this coalition, and N is the total number of features. _N\\{i}_ is all the possible feature coalitions not containing _i_. The first term within the sum corresponds to the fraction of times _S_ appears within the possible feature permutations; intuitively, this gives the highest weight to the most informative contributions of a feature, i.e. when it is either isolated or when it is added to a full set of features. " %}

Let's look at an example: you have a model that predicts the probability of rain today, using three input Boolean features: A: is it winter?, B: is the sky cloudy?, and C: are some people carrying umbrellas? The model's output probability is a cumulative 20% for each individual positive feature, but if _A_ is true and any of the other features is true as well, the output probability is 90%. If today _A,B,_ and _C_ are all true - how much does each feature contribute to the final 90% output?

Let's compute this for feature _C_, using all possible feature combinations not containing it. For the sake of simplicity, let's say that each missing feature can be assumed to be _false_ rather than _unknown_.

|| Coalition without _C_ || Output without _C_ || Coalition with _C_ ||  Output with _C_ || Change in prediction after adding _C_ ||
||---    ||: --- ||: --- ||: ---||: ---||
|| { }   || 0%      || {C}      || 20%   || **20%** ||
|| {A}   || 20%     || {A,C}    || 90%   || **70%** ||
|| {B}   || 20%     || {B,C}    || 40%   || **20%** ||
|| {A,B} || 90%     || {A,B,C}  || 90%   || **0%**  ||

Applying the formula (the first term of the sum in the Shapley formula is 1/3 for {} and {A,B} and 1/6 for {A} and {B}), we get a Shapley value of **21.66%** for feature _C_. Feature _B_ will naturally have the same value, while repeating this procedure for feature _A_ will give us **46.66%**. A crucial characteristic of Shapley values is that the features' contributions always add up to the final score: **21.66% + 21.66% + 46.67% = 90%**.


 <font size="-1"> <b>If you wish to comment on this post you may do so on <a href="https://medium.com/@edden.gerber...">Medium</a>.</b> <font size="+1">

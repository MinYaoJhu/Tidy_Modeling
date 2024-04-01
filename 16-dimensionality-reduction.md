---
title: "16 Dimensionality Reduction"
author: "Min-Yao"
date: "2024-03-30"
output: 
  html_document: 
    keep_md: true
---




# Dimensionality Reduction {#dimensionality}

Dimensionality reduction transforms a data set from a high-dimensional space into a low-dimensional space, and can be a good choice when you suspect there are "too many" variables. An excess of variables, usually predictors, can be a problem because it is difficult to understand or visualize data in higher dimensions. 

## What Problems Can Dimensionality Reduction Solve?

Dimensionality reduction can be used either in feature engineering or in exploratory data analysis. For example, in high-dimensional biology experiments, one of the first tasks, before any modeling, is to determine if there are any unwanted trends in the data (e.g., effects not related to the question of interest, such as lab-to-lab differences). Debugging the data is difficult when there are hundreds of thousands of dimensions, and dimensionality reduction can be an aid for exploratory data analysis.

Another potential consequence of having a multitude of predictors is possible harm to a model. The simplest example is a method like ordinary linear regression where the number of predictors should be less than the number of data points used to fit the model. Another issue is multicollinearity, where between-predictor correlations can negatively impact the mathematical operations used to estimate a model. If there are an extremely large number of predictors, it is fairly unlikely that there are an equal number of real underlying effects. Predictors may be measuring the same latent effect(s), and thus such predictors will be highly correlated. Many dimensionality reduction techniques thrive in this situation. In fact, most can be effective only when there are such relationships between predictors that can be exploited.

:::rmdnote
When starting a new modeling project, reducing the dimensions of the data may provide some intuition about how hard the modeling problem may be. 
:::

Principal component analysis (PCA) is one of the most straightforward methods for reducing the number of columns in the data set because it relies on linear methods and is unsupervised (i.e., does not consider the outcome data). For a high-dimensional classification problem, an initial plot of the main PCA components might show a clear separation between the classes. If this is the case, then it is fairly safe to assume that a linear classifier might do a good job. However, the converse is not true; a lack of separation does not mean that the problem is insurmountable.

The dimensionality reduction methods discussed in this chapter are generally _not_ feature selection methods. Methods such as PCA represent the original predictors using a smaller subset of new features. All of the original predictors are required to compute these new features. The exception to this are sparse methods that have the ability to completely remove the impact of predictors when creating the new features.

:::rmdnote
This chapter has two goals: 

 * Demonstrate how to use recipes to create a small set of features that capture the main aspects of the original predictor set.
 
 * Describe how recipes can be used on their own (as opposed to being used in a workflow object, as in Section \@ref(using-recipes)). 
:::
 
The latter is helpful when testing or debugging a recipe. However, as described in Section \@ref(using-recipes), the best way to use a recipe for modeling is from within a workflow object. 

In addition to the `pkg(tidymodels)` package, this chapter uses the following packages: `pkg(baguette)`, `pkg(beans)`, `pkg(bestNormalize)`, `pkg(corrplot)`, `pkg(discrim)`, `pkg(embed)`, `pkg(ggforce)`, `pkg(klaR)`, `pkg(learntidymodels)`,[^learnnote] `pkg(mixOmics)`,[^mixnote] and `pkg(uwot)`. 

[^learnnote]: The `pkg(learntidymodels)` package can be found at its GitHub site: <https://github.com/tidymodels/learntidymodels>

[^mixnote]: The `pkg(mixOmics)` package is not available on CRAN, but instead on Bioconductor: <https://doi.org/doi:10.18129/B9.bioc.mixOmics>

## A Picture Is Worth a Thousand... Beans {#beans}

Let's walk through how to use dimensionality reduction with `pkg(recipes)` for an example data set. @beans published a data set of visual characteristics of dried beans and described methods for determining the varieties of dried beans in an image. While the dimensionality of these data is not very large compared to many real-world modeling problems, it does provide a nice working example to demonstrate how to reduce the number of features. From their manuscript:

> The primary objective of this study is to provide a method for obtaining uniform seed varieties from crop production, which is in the form of population, so the seeds are not certified as a sole variety. Thus, a computer vision system was developed to distinguish seven different registered varieties of dry beans with similar features in order to obtain uniform seed classification. For the classification model, images of 13,611 grains of 7 different registered dry beans were taken with a high-resolution camera.

Each image contains multiple beans. The process of determining which pixels correspond to a particular bean is called _image segmentation_. These pixels can be analyzed to produce features for each bean, such as color and morphology (i.e., shape). These features are then used to model the outcome (bean variety) because different bean varieties look different. The training data come from a set of manually labeled images, and this data set is used to create a predictive model that can distinguish between seven bean varieties: Cali, Horoz, Dermason, Seker, Bombay, Barbunya, and Sira. Producing an effective model can help manufacturers quantify the homogeneity of a batch of beans. 

There are numerous methods for quantifying shapes of objects [@Mingqiang08]. Many are related to the boundaries or regions of the object of interest. Example of features include:

-   The *area* (or size) can be estimated using the number of pixels in the object or the size of the convex hull around the object.

-   We can measure the *perimeter* using the number of pixels in the boundary as well as the area of the bounding box (the smallest rectangle enclosing an object).

-   The *major axis* quantifies the longest line connecting the most extreme parts of the object. The *minor axis* is perpendicular to the major axis.

-   We can measure the *compactness* of an object using the ratio of the object's area to the area of a circle with the same perimeter. For example, the symbols "`cli::symbol$bullet`" and "`cli::symbol$times`" have very different compactness.

-   There are also different measures of how *elongated* or oblong an object is. For example, the *eccentricity* statistic is the ratio of the major and minor axes. There are also related estimates for roundness and convexity.

Notice the eccentricity for the different shapes in Figure \@ref(fig:eccentricity).

<div class="figure">
<img src="premade/morphology.svg" alt="Some example shapes and their eccentricity statistics. Circles and squares have the smallest eccentricity values while X shapes and lightning bolts have the largest. Also, the eccentricity is the same when shapes are rotated." width="95%" />
<p class="caption">Some example shapes and their eccentricity statistics</p>
</div>

Shapes such as circles and squares have low eccentricity while oblong shapes have high values. Also, the metric is unaffected by the rotation of the object.

Many of these image features have high correlations; objects with large areas are more likely to have large perimeters. There are often multiple methods to quantify the same underlying characteristics (e.g., size).

In the bean data, `ncol(beans) - 1` morphology features were computed: `knitr::combine_words(gsub("_", " ", names(beans)[-ncol(beans)]))`. The latter four are described in @symons1988211. 

We can begin by loading the data:


```r
library(tidymodels)
tidymodels_prefer()
library(beans)
```

:::rmdwarning
It is important to maintain good data discipline when evaluating dimensionality reduction techniques, especially if you will use them within a model. 
:::

For our analyses, we start by holding back a testing set with `initial_split()`. The remaining data are split into training and validation sets:


```r
set.seed(1601)
bean_split <- initial_validation_split(beans, strata = class, prop = c(0.75, 0.125))
```

```
## Warning: Too little data to stratify.
## • Resampling will be unstratified.
```

```r
bean_split
```

```
## <Training/Validation/Testing/Total>
## <10206/1702/1703/13611>
```

```r
# Return data frames:
bean_train <- training(bean_split)
bean_test <- testing(bean_split)
bean_validation <- validation(bean_split)


set.seed(1602)
# Return an 'rset' object to use with the tune functions:
bean_val <- validation_set(bean_split)
bean_val$splits[[1]]
```

```
## <Training/Validation/Total>
## <10206/1702/11908>
```

To visually assess how well different methods perform, we can estimate the methods on the training set (n = `format(nrow(bean_train), big.mark = ",")` beans) and display the results using the validation set (n = `format(nrow(bean_validation), big.mark = ",")`).

Before beginning any dimensionality reduction, we can spend some time investigating our data. Since we know that many of these shape features are probably measuring similar concepts, let's take a look at the correlation structure of the data in Figure \@ref(fig:beans-corr-plot) using this code.


```r
library(corrplot)
tmwr_cols <- colorRampPalette(c("#91CBD765", "#CA225E"))
bean_train %>% 
  select(-class) %>% 
  cor() %>% 
  corrplot(col = tmwr_cols(200), tl.col = "black", method = "ellipse")
```

![](16-dimensionality-reduction_files/figure-html/dimensionality-corr-plot-1.png)<!-- -->

<div class="figure">
<img src="16-dimensionality-reduction_files/figure-html/beans-corr-plot-1.png" alt="A correlation matrix of the predictors with variables ordered via clustering. There are two to three clusters that have high within cluster correlations." width="70%" />
<p class="caption">Correlation matrix of the predictors with variables ordered via clustering</p>
</div>

Many of these predictors are highly correlated, such as area and perimeter or shape factors 2 and 3. While we don't take the time to do it here, it is also important to see if this correlation structure significantly changes across the outcome categories. This can help create better models.

## A Starter Recipe

It's time to look at the beans data in a smaller space. We can start with a basic recipe to preprocess the data prior to any dimensionality reduction steps. Several predictors are ratios and so are likely to have skewed distributions. Such distributions can wreak havoc on variance calculations (such as the ones used in PCA). The [`pkg(bestNormalize)` package](https://petersonr.github.io/bestNormalize/) has a step that can enforce a symmetric distribution for the predictors. We'll use this to mitigate the issue of skewed distributions:


```r
library(bestNormalize)
bean_rec <-
  # Use the training data from the bean_val split object
  recipe(class ~ ., data = bean_train) %>%
  step_zv(all_numeric_predictors()) %>%
  step_orderNorm(all_numeric_predictors()) %>% 
  step_normalize(all_numeric_predictors())
```

:::rmdnote
Remember that when invoking the `recipe()` function, the steps are not estimated or executed in any way. 
:::

This recipe will be extended with additional steps for the dimensionality reduction analyses. Before doing so, let's go over how a recipe can be used outside of a workflow. 

## Recipes in the Wild {#recipe-functions}

As mentioned in Section \@ref(using-recipes), a workflow containing a recipe uses `fit()` to estimate the recipe and model, then `predict()` to process the data and make model predictions. There are analogous functions in the `pkg(recipes)` package that can be used for the same purpose: 

* `prep(recipe, training)` fits the recipe to the training set. 
* `bake(recipe, new_data)` applies the recipe operations to `new_data`. 

Figure \@ref(fig:recipe-process) summarizes this. Let's look at each of these functions in more detail.

<div class="figure">
<img src="premade/recipes-process.svg" alt="A summary of the recipe-related functions." width="80%" />
<p class="caption">Summary of recipe-related functions</p>
</div>


### Preparing a recipe {#prep}

Let's estimate `bean_rec` using the training set data, with `prep(bean_rec)`:



```r
bean_rec_trained <- prep(bean_rec)
bean_rec_trained
```

```
## 
```

```
## ── Recipe ──────────────────────────────────────────────────────────────────────
```

```
## 
```

```
## ── Inputs
```

```
## Number of variables by role
```

```
## outcome:    1
## predictor: 16
```

```
## 
```

```
## ── Training information
```

```
## Training data contained 10206 data points and no incomplete rows.
```

```
## 
```

```
## ── Operations
```

```
## • Zero variance filter removed: <none> | Trained
```

```
## • orderNorm transformation on: area and perimeter, ... | Trained
```

```
## • Centering and scaling for: area and perimeter, ... | Trained
```

:::rmdnote
Remember that `prep()` for a recipe is like `fit()` for a model.
:::

Note in the output that the steps have been trained and that the selectors are no longer general (i.e., `all_numeric_predictors()`); they now show the actual columns that were selected. Also, `prep(bean_rec)` does not require the `training` argument. You can pass any data into that argument, but omitting it means that the original `data` from the call to `recipe()` will be used. In our case, this was the training set data. 

One important argument to `prep()` is `retain`. When `retain = TRUE` (the default), the estimated version of the training set is kept within the recipe. This data set has been pre-processed using all of the steps listed in the recipe. Since `prep()` has to execute the recipe as it proceeds, it may be advantageous to keep this version of the training set so that, if that data set is to be used later, redundant calculations can be avoided. However, if the training set is big, it may be problematic to keep such a large amount of data in memory. Use `retain = FALSE` to avoid this. 

Once new steps are added to this estimated recipe, reapplying `prep()` will estimate only the untrained steps. This will come in handy when we try different feature extraction methods.

:::rmdwarning
If you encounter errors when working with a recipe, `prep()` can be used with its `verbose` option to troubleshoot: 
:::


```r
bean_rec_trained %>% 
  step_dummy(cornbread) %>%  # <- not a real predictor
  prep(verbose = TRUE)
```

```
## oper 1 step zv [pre-trained]
## oper 2 step orderNorm [pre-trained]
## oper 3 step normalize [pre-trained]
## oper 4 step dummy [training]
```

```
## Error in `step_dummy()`:
## Caused by error in `prep()`:
## ! Can't select columns that don't exist.
## ✖ Column `cornbread` doesn't exist.
```

Another option that can help you understand what happens in the analysis is `log_changes`:


```r
show_variables <- 
  bean_rec %>% 
  prep(log_changes = TRUE)
```

```
## step_zv (zv_RLYwH): same number of columns
## 
## step_orderNorm (orderNorm_Jx8oD): same number of columns
## 
## step_normalize (normalize_GU75D): same number of columns
```

### Baking the recipe {#bake}

:::rmdnote
Using `bake()` with a recipe is much like using `predict()` with a model; the operations estimated from the training set are applied to any data, like testing data or new data at prediction time. 
:::

For example, the validation set samples can be processed: 


```r
bean_val_processed <- bake(bean_rec_trained, new_data = bean_validation)
```

Figure \@ref(fig:bean-area) shows histograms of the `area` predictor before and after the recipe was prepared.


```r
library(patchwork)
p1 <- 
  bean_validation %>% 
  ggplot(aes(x = area)) + 
  geom_histogram(bins = 30, color = "white", fill = "blue", alpha = 1/3) + 
  ggtitle("Original validation set data")

p2 <- 
  bean_val_processed %>% 
  ggplot(aes(x = area)) + 
  geom_histogram(bins = 30, color = "white", fill = "red", alpha = 1/3) + 
  ggtitle("Processed validation set data")

p1 + p2
```

![](16-dimensionality-reduction_files/figure-html/dimensionality-bake-off-1.png)<!-- -->

<div class="figure">
<img src="16-dimensionality-reduction_files/figure-html/bean-area-1.png" alt="The `area` predictor before and after preprocessing. The before panel shows a right-skewed, slightly bimodal distribution. The after panel has a distribution that is fairly bell shaped."  />
<p class="caption">The `area` predictor before and after preprocessing</p>
</div>

Two important aspects of `bake()` are worth noting here. 

First, as previously mentioned, using `prep(recipe, retain = TRUE)` keeps the existing processed version of the training set in the recipe. This enables the user to use `bake(recipe, new_data = NULL)`, which returns that data set without further computations. For example: 


```r
bake(bean_rec_trained, new_data = NULL) %>% nrow()
```

```
## [1] 10206
```

```r
bean_train %>% nrow()
```

```
## [1] 10206
```

If the training set is not pathologically large, using this value of `retain` can save a lot of computational time. 

Second, additional selectors can be used in the call to specify which columns to return. The default selector is `everything()`, but more specific directives can be used. 

We will use `prep()` and `bake()` in the next section to illustrate some of these options. 

## Feature Extraction Techniques

Since recipes are the primary option in tidymodels for dimensionality reduction, let's write a function that will estimate the transformation and plot the resulting data in a scatter plot matrix via the `pkg(ggforce)` package:


```r
library(ggforce)
plot_validation_results <- function(recipe, dat = bean_validation) {
  recipe %>%
    # Estimate any additional steps
    prep() %>%
    # Process the data (the validation set by default)
    bake(new_data = dat) %>%
    # Create the scatterplot matrix
    ggplot(aes(x = .panel_x, y = .panel_y, color = class, fill = class)) +
    geom_point(alpha = 0.4, size = 0.5) +
    geom_autodensity(alpha = .3) +
    facet_matrix(vars(-class), layer.diag = 2) + 
    scale_color_brewer(palette = "Dark2") + 
    scale_fill_brewer(palette = "Dark2")
}
```

We will reuse this function several times in this chapter.

A series of several feature extraction methodologies are explored here. An overview of most can be found in [Section 6.3.1](https://bookdown.org/max/FES/numeric-many-to-many.html#linear-projection-methods) of @fes and the references therein. The UMAP method is described in @mcinnes2020umap.

### Principal component analysis

We've mentioned PCA several times already in this book, and it's time to go into more detail. PCA is an unsupervised method that uses linear combinations of the predictors to define new features. These features attempt to account for as much variation as possible in the original data. We add `step_pca()` to the original recipe and use our function to visualize the results on the validation set in Figure \@ref(fig:bean-pca) using:


```r
bean_rec_trained %>%
  step_pca(all_numeric_predictors(), num_comp = 4) %>%
  plot_validation_results() + 
  ggtitle("Principal Component Analysis")
```

![](16-dimensionality-reduction_files/figure-html/dimensionality-pca-1.png)<!-- -->

<div class="figure">
<img src="16-dimensionality-reduction_files/figure-html/bean-pca-1.png" alt="Principal component scores for the bean validation set, colored by class. The classes separate when the first two components are plotted against one another."  />
<p class="caption">Principal component scores for the bean validation set, colored by class</p>
</div>

We see that the first two components `PC1` and `PC2`, especially when used together, do an effective job distinguishing between or separating the classes. This may lead us to expect that the overall problem of classifying these beans will not be especially difficult.

Recall that PCA is unsupervised. For these data, it turns out that the PCA components that explain the most variation in the predictors also happen to be predictive of the classes. What features are driving performance? The `pkg(learntidymodels)` package has functions that can help visualize the top features for each component. We'll need the prepared recipe; the PCA step is added in the following code along with a call to `prep()`:


```r
library(learntidymodels)
bean_rec_trained %>%
  step_pca(all_numeric_predictors(), num_comp = 4) %>% 
  prep() %>% 
  plot_top_loadings(component_number <= 4, n = 5) + 
  scale_fill_brewer(palette = "Paired") +
  ggtitle("Principal Component Analysis")
```

![](16-dimensionality-reduction_files/figure-html/dimensionality-pca-loadings-1.png)<!-- -->

This produces Figure \@ref(fig:pca-loadings).

<div class="figure">
<img src="16-dimensionality-reduction_files/figure-html/pca-loadings-1.png" alt="Predictor loadings for the PCA transformation. For the first component, the major axis length, second shape factor, convex area, and area have the largest effect. "  />
<p class="caption">Predictor loadings for the PCA transformation</p>
</div>

The top loadings are mostly related to the cluster of correlated predictors shown in the top-left portion of the previous correlation plot: perimeter, area, major axis length, and convex area. These are all related to bean size. Shape factor 2, from @symons1988211, is the area over the cube of the major axis length and is therefore also related to bean size. Measures of elongation appear to dominate the second PCA component.

### Partial least squares

PLS, which we introduced in Section \@ref(submodel-trick), is a supervised version of PCA. It tries to find components that simultaneously maximize the variation in the predictors while also maximizing the relationship between those components and the outcome. Figure \@ref(fig:bean-pls) shows the results of this slightly modified version of the PCA code:


```r
bean_rec_trained %>%
  step_pls(all_numeric_predictors(), outcome = "class", num_comp = 4) %>%
  plot_validation_results() + 
  ggtitle("Partial Least Squares")
```

![](16-dimensionality-reduction_files/figure-html/dimensionality-pls-1.png)<!-- -->

<div class="figure">
<img src="16-dimensionality-reduction_files/figure-html/bean-pls-1.png" alt="PLS component scores for the bean validation set, colored by class. The first two PLS components are nearly identical to the first two PCA components."  />
<p class="caption">PLS component scores for the bean validation set, colored by class</p>
</div>

The first two PLS components plotted in Figure \@ref(fig:bean-pls) are nearly identical to the first two PCA components! We find this result because those PCA components are so effective at separating the varieties of beans. The remaining components are different. Figure \@ref(fig:pls-loadings) visualizes the loadings, the top features for each component.


```r
bean_rec_trained %>%
  step_pls(all_numeric_predictors(), outcome = "class", num_comp = 4) %>%
  prep() %>% 
  plot_top_loadings(component_number <= 4, n = 5, type = "pls") + 
  scale_fill_brewer(palette = "Paired") +
  ggtitle("Partial Least Squares")
```

![](16-dimensionality-reduction_files/figure-html/dimensionality-pls-loadings-1.png)<!-- -->

<div class="figure">
<img src="16-dimensionality-reduction_files/figure-html/pls-loadings-1.png" alt="Predictor loadings for the PLS transformation. For the first component, the major axis length, second shape factor, the equivalent diameter, convex area, and area have the largest effect. "  />
<p class="caption">Predictor loadings for the PLS transformation</p>
</div>

Solidity (i.e., the density of the bean) drives the third PLS component, along with roundness. Solidity may be capturing bean features related to "bumpiness" of the bean surface since it can measure irregularity of the bean boundaries.

### Independent component analysis

ICA is slightly different than PCA in that it finds components that are as statistically independent from one another as possible (as opposed to being uncorrelated). It can be thought of as maximizing the "non-Gaussianity" of the ICA components, or separating information instead of compressing information like PCA. Let's use `step_ica()` to produce Figure \@ref(fig:bean-ica):


```r
bean_rec_trained %>%
  step_ica(all_numeric_predictors(), num_comp = 4) %>%
  plot_validation_results() + 
  ggtitle("Independent Component Analysis")
```

![](16-dimensionality-reduction_files/figure-html/dimensionality-ica-1.png)<!-- -->

<div class="figure">
<img src="16-dimensionality-reduction_files/figure-html/bean-ica-1.png" alt="ICA component scores for the bean validation set, colored by class. There is significant overlap in the first two ICA components."  />
<p class="caption">ICA component scores for the bean validation set, colored by class</p>
</div>

Inspecting this plot, there does not appear to be much separation between the classes in the first few components when using ICA. These independent (or as independent as possible) components do not separate the bean types.

### Uniform manifold approximation and projection

UMAP is similar to the popular t-SNE method for nonlinear dimension reduction. In the original high-dimensional space, UMAP uses a distance-based nearest neighbor method to find local areas of the data where the data points are more likely to be related. The relationship between data points is saved as a directed graph model where most points are not connected.

From there, UMAP translates points in the graph to the reduced dimensional space. To do this, the algorithm has an optimization process that uses cross-entropy to map data points to the smaller set of features so that the graph is well approximated.

To create the mapping, the `pkg(embed)` package contains a step function for this method, visualized in Figure \@ref(fig:bean-umap).


```r
library(embed)
bean_rec_trained %>%
  step_umap(all_numeric_predictors(), num_comp = 4) %>%
  plot_validation_results() +
  ggtitle("UMAP")
```

![](16-dimensionality-reduction_files/figure-html/dimensionality-umap-1.png)<!-- -->

<div class="figure">
<img src="16-dimensionality-reduction_files/figure-html/bean-umap-1.png" alt="UMAP component scores for the bean validation set, colored by class. There is a very high degree of separation between clusters, but several of the clusters contain more than one class."  />
<p class="caption">UMAP component scores for the bean validation set, colored by class</p>
</div>

While the between-cluster space is pronounced, the clusters can contain a heterogeneous mixture of classes.

There is also a supervised version of UMAP:


```r
bean_rec_trained %>%
  step_umap(all_numeric_predictors(), outcome = "class", num_comp = 4) %>%
  plot_validation_results() +
  ggtitle("UMAP (supervised)")
```

![](16-dimensionality-reduction_files/figure-html/dimensionality-umap-supervised-1.png)<!-- -->

<div class="figure">
<img src="16-dimensionality-reduction_files/figure-html/bean-umap-supervised-1.png" alt="Supervised UMAP component scores for the bean validation set, colored by class. There is again a very high degree of separation between clusters, and there are now fewer instances of one cluster containing multiple classes."  />
<p class="caption">Supervised UMAP component scores for the bean validation set, colored by class</p>
</div>

The supervised method shown in Figure \@ref(fig:bean-umap-supervised) looks promising for modeling the data.

UMAP is a powerful method to reduce the feature space. However, it can be very sensitive to tuning parameters (e.g., the number of neighbors and so on). For this reason, it would help to experiment with a few of the parameters to assess how robust the results are for these data.

## Modeling {#bean-models}

Both the PLS and UMAP methods are worth investigating in conjunction with different models. Let's explore a variety of different models with these dimensionality reduction techniques (along with no transformation at all): a single layer neural network, bagged trees, flexible discriminant analysis (FDA), naive Bayes, and regularized discriminant analysis (RDA).

Now that we are back in "modeling mode," we'll create a series of model specifications and then use a workflow set to tune the models in the following code. Note that the model parameters are tuned in conjunction with the recipe parameters (e.g., size of the reduced dimension, UMAP parameters).


```r
library(baguette)
library(discrim)
library(earth)
```

```
## Loading required package: Formula
```

```
## Loading required package: plotmo
```

```
## Loading required package: plotrix
```

```
## 
## Attaching package: 'plotrix'
```

```
## The following object is masked from 'package:scales':
## 
##     rescale
```

```r
library(mda)
```

```
## Loading required package: class
```

```
## Loaded mda 0.5-4
```

```r
mlp_spec <-
  mlp(hidden_units = tune(), penalty = tune(), epochs = tune()) %>%
  set_engine('nnet') %>%
  set_mode('classification')

bagging_spec <-
  bag_tree() %>%
  set_engine('rpart') %>%
  set_mode('classification')

fda_spec <-
  discrim_flexible(
    prod_degree = tune()
  ) %>%
  set_engine('earth')

rda_spec <-
  discrim_regularized(frac_common_cov = tune(), frac_identity = tune()) %>%
  set_engine('klaR')

bayes_spec <-
  naive_Bayes() %>%
  set_engine('klaR')
```

We also need recipes for the dimensionality reduction methods we'll try. Let's start with a base recipe `bean_rec` and then extend it with different dimensionality reduction steps:


```r
bean_rec <-
  recipe(class ~ ., data = bean_train) %>%
  step_zv(all_numeric_predictors()) %>%
  step_orderNorm(all_numeric_predictors()) %>%
  step_normalize(all_numeric_predictors())

pls_rec <- 
  bean_rec %>% 
  step_pls(all_numeric_predictors(), outcome = "class", num_comp = tune())

umap_rec <-
  bean_rec %>%
  step_umap(
    all_numeric_predictors(),
    outcome = "class",
    num_comp = tune(),
    neighbors = tune(),
    min_dist = tune()
  )
```

Once again, the `pkg(workflowsets)` package takes the preprocessors and models and crosses them. The `control` option `parallel_over` is set so that the parallel processing can work simultaneously across tuning parameter combinations. The `workflow_map()` function applies grid search to optimize the model/preprocessing parameters (if any) across 10 parameter combinations. The multiclass area under the ROC curve is estimated on the validation set.


```r
ctrl <- control_grid(parallel_over = "everything")
bean_res <- 
  workflow_set(
    preproc = list(basic = class ~., pls = pls_rec, umap = umap_rec), 
    models = list(bayes = bayes_spec, fda = fda_spec,
                  rda = rda_spec, bag = bagging_spec,
                  mlp = mlp_spec)
  ) %>% 
  workflow_map(
    verbose = TRUE,
    seed = 1603,
    resamples = bean_val,
    grid = 10,
    metrics = metric_set(roc_auc),
    control = ctrl
  )
```

We can rank the models by their validation set estimates of the area under the ROC curve:


```r
rankings <- 
  rank_results(bean_res, select_best = TRUE) %>% 
  mutate(method = map_chr(wflow_id, ~ str_split(.x, "_", simplify = TRUE)[1])) 

tidymodels_prefer()
filter(rankings, rank <= 5) %>% dplyr::select(rank, mean, model, method)
```

```
## # A tibble: 5 × 4
##    rank  mean model               method
##   <int> <dbl> <chr>               <chr> 
## 1     1 0.996 mlp                 pls   
## 2     2 0.996 discrim_regularized pls   
## 3     3 0.995 discrim_flexible    basic 
## 4     4 0.995 naive_Bayes         pls   
## 5     5 0.994 naive_Bayes         basic
```

Figure \@ref(fig:dimensionality-rankings) illustrates this ranking.

<div class="figure">
<img src="16-dimensionality-reduction_files/figure-html/dimensionality-rankings-1.png" alt="Area under the ROC curve from the validation set. The three best model configurations use PLS together with regularized discriminant analysis, a multi-layer perceptron, and a naive Bayes model."  />
<p class="caption">Area under the ROC curve from the validation set</p>
</div>

It is clear from these results that most models give very good performance; there are few bad choices here. For demonstration, we'll use the RDA model with PLS features as the final model. We will finalize the workflow with the numerically best parameters, fit it to the training set, then evaluate with the test set:


```r
rda_res <- 
  bean_res %>% 
  extract_workflow("pls_rda") %>% 
  finalize_workflow(
    bean_res %>% 
      extract_workflow_set_result("pls_rda") %>% 
      select_best(metric = "roc_auc")
  ) %>% 
  last_fit(split = bean_split, metrics = metric_set(roc_auc))

rda_wflow_fit <- extract_workflow(rda_res)
```

What are the results for our metric (multiclass ROC AUC) on the testing set?


```r
collect_metrics(rda_res)
```

```
## # A tibble: 1 × 4
##   .metric .estimator .estimate .config             
##   <chr>   <chr>          <dbl> <chr>               
## 1 roc_auc hand_till      0.995 Preprocessor1_Model1
```

Pretty good! We'll use this model in the next chapter to demonstrate variable importance methods.


```r
save(rda_wflow_fit, bean_train, file = "RData/rda_fit.RData", version = 2, compress = "xz")
```

## Chapter Summary {#dimensionality-summary}

Dimensionality reduction can be a helpful method for exploratory data analysis as well as modeling. The `pkg(recipes)` and `pkg(embed)` packages contain steps for a variety of different methods and `pkg(workflowsets)` facilitates choosing an appropriate method for a data set. This chapter also discussed how recipes can be used on their own, either for debugging problems with a recipe or directly for exploratory data analysis and data visualization. 
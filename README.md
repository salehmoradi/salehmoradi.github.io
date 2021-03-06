---
title: "Quality of Physical Activities"
author: "Saleh Moradi"
date: "23 August 2018"
output: html_document
---
## Summary  

The objective here is to predict the manner in which subjects did exercise, correctly or incorrectly. To this end, a prediction model has been explicated, devised, developled and cross-validated to predict the manner of physical activity using the most relevant variables offered in the Training Datasset.  
Apparently, data has been collected by means of four devices mounted on participants belt, forearm, arm, and dumbell. Each device has produced roll, pitch, yaw measures as well as 3-dimensional acceleration, gyros and magnet data (*I am not sure about the nature of the last two type of data, namely gyros and magnet*) and total acceraltion. The model has been mainly built on these information. To make the best use of data, however, the explanatory value of the participant and the time of activity (i.e., hour of activity) have also been examined.    

First, data was partitioned into two datasets, 80% Training dataset and 20% Validation dataset. (* Please note that this validation dataset is in fact part of the original training dataset and is different from the test set. *) Next, two algorithms of Random Forest and Gradient Boosting Classification was used to predict the activity class for each observation. The results of modeling shows that a Random Forest model provides a great overall accuracy 99.3% in predicting the Class of Activity for the training dataset. This accuracy level stayed high at 99.4% when using the same model to predict the dependent variable in the Validation dataset, hence, suggesting a low out-of-sample error rate. When fitting a Gradient Boosting Model (GBM), however, a higher level of out-of-sample error was witnessed with accuracy level decreasing from 99.3% for the training dataset to 98.% for the validation dataset. This difference appreared even though 10-fold cross validation technique was used for all modeling steps.

All in all, the random forest model was chosen to be sued to predict the classifications for the test dataset.

*DISCLAIMER: As I could not find a metadata explaining rows and columns of the dataset, it should be considered that the following analysis and interpretations are subject to my personal understanding of the dataset.* 

### Data Aquisition and Wrangling
```{r echo=TRUE, message=FALSE}
# Loading required packages:
library(dplyr)
library(ggplot2)
library(h2o)
library(caret)
library(lubridate)

# Loading and Probing the data
train_df <- read.csv("pml-training.csv")
dim(train_df)

# Data Clean Up
## Seperating Aggregated vs Raw Observations (seemingly indicated by "new_window" column, where "yes" seems to indicate aggregation)
train_df_tmp <- train_df %>%
    filter(new_window == "no") %>%
    mutate(id = recode(as.character(user_name),
                       "carlitos" = 1, "pedro" = 2, "adelmo" = 3, "eurico" = 4, "jeremy" = 5, "charles" = 6)) %>%
    select(-12:-36,-50:-59,-69:-83,-87:-101,-103:-112,-125:-139,-141:-150)
# Sort data sets by outcome variable being the first column
train_df_Raw <- train_df_tmp %>% select(60,61,2,-1,3:59)
# Create a column indicating the hour of the activity
train_df_Raw$hour <- as.factor(hour(dmy_hm(train_df_Raw$cvtd_timestamp)))
# Final Training Dataset
train_df_final <- train_df_Raw %>%
    select(-3:-8) %>%
    mutate(id = as.factor(id)) %>%
    mutate(total_accel_belt = as.numeric(total_accel_belt)) %>%
    mutate(accel_belt_x = as.numeric(accel_belt_x)) %>%
    mutate(accel_belt_y = as.numeric(accel_belt_y)) %>%
    mutate(accel_belt_z = as.numeric(accel_belt_z)) %>%
    mutate(magnet_belt_x = as.numeric(magnet_belt_x)) %>%
    mutate(magnet_belt_y = as.numeric(magnet_belt_y)) %>%
    mutate(magnet_belt_z = as.numeric(magnet_belt_z)) %>%
    mutate(total_accel_arm = as.numeric(total_accel_arm)) %>%
    mutate(accel_arm_x = as.numeric(accel_arm_x)) %>%
    mutate(accel_arm_y = as.numeric(accel_arm_y)) %>%
    mutate(accel_arm_z = as.numeric(accel_arm_z)) %>%
    mutate(magnet_arm_x = as.numeric(magnet_arm_x)) %>%
    mutate(magnet_arm_y = as.numeric(magnet_arm_y)) %>%
    mutate(magnet_arm_z = as.numeric(magnet_arm_z)) %>%
    mutate(total_accel_dumbbell = as.numeric(total_accel_dumbbell)) %>%
    mutate(accel_dumbbell_x = as.numeric(accel_dumbbell_x)) %>%
    mutate(accel_dumbbell_y = as.numeric(accel_dumbbell_y)) %>%
    mutate(accel_dumbbell_z = as.numeric(accel_dumbbell_z)) %>%
    mutate(magnet_dumbbell_x = as.numeric(magnet_dumbbell_x)) %>%
    mutate(magnet_dumbbell_y = as.numeric(magnet_dumbbell_y)) %>%
    mutate(magnet_dumbbell_z = as.numeric(magnet_dumbbell_z)) %>%
    mutate(total_accel_forearm = as.numeric(total_accel_forearm)) %>%
    mutate(accel_forearm_x = as.numeric(accel_forearm_x)) %>%
    mutate(accel_forearm_y = as.numeric(accel_forearm_y)) %>%
    mutate(accel_forearm_z = as.numeric(accel_forearm_z)) %>%
    mutate(magnet_forearm_x = as.numeric(magnet_forearm_x)) %>%
    mutate(magnet_forearm_y = as.numeric(magnet_forearm_y)) %>%
    mutate(magnet_forearm_z = as.numeric(magnet_forearm_z))

dim(train_df_final)
```

### Exploratory Analysis
```{r echo=TRUE, message=FALSE}
# Exploratory Plots
ggplot(data = train_df_final, aes(x = classe)) + 
    facet_wrap(~id) +
    geom_histogram(binwidth = 5, stat = "count") +
    labs(title ="Type of Physical Activity by Participant",x = "Participant", y = "Total Count")
ggplot(data = train_df_final, aes(x = classe)) + 
    facet_wrap(~hour) +
    geom_histogram(binwidth = 5, stat = "count") +
    labs(title ="Type of Physical Activity by Hour",x = "Hour", y = "Total Count")
```
In as much as it seems that different subjects are showing somewhat different frequency in type of physical activity, no unique patterns seem to be seen among different hours. *Hence, hour will not be included in the following models.*

```{r echo=TRUE, message=FALSE}
# Removing the Hour variable from the dataset.
train_df_final <- train_df_final %>% select(-55)

# Partitioning Training Dataset into a training dataframe (80%) and a validation dataframe (20%)
set.seed(1234)
indices <- createDataPartition(train_df_final$classe, p = .8, list = FALSE)
training_df <- train_df_final[indices,]
validation_df  <- train_df_final[-indices,]
dim(training_df)
dim(validation_df)
```

### Model Building and Cross Validation
For model building and cross-validation, h2o package was used. Using h2o package, not only the computations can be performed faster, the arguments of each algorithm provides an easy way to tune the model, enter both the training and the validation model at once, and also choose the desired cross-validation regime. To learn more about H2O, please visit http://docs.h2o.ai/h2o/latest-stable/h2o-docs/data-science.html.   

In the follwoing, given that the outcome variable is categorical (with 5 levels), I have used two alternative algorithms, namely Random Forest and Gradient Boosting Classification to be able to predict the classification of each observation in one of the 5 physical activity classes.  
```{r echo=TRUE, message=FALSE, cache=TRUE, results=FALSE}
h2o.init(nthreads = -1)
# First, a Random Forest agorithm was fitted.
rf <- h2o.randomForest(training_frame = as.h2o(training_df), #Training dataframe
                       validation_frame = as.h2o(validation_df), #Validation dataframe
                       y = 1, #The Dependent Variable
                       nfolds = 10, #10-fold cross-validation
                       ntrees = 50, #number of trees
                       seed = 123 #(time-based random number)
                       )
```

```{r echo=TRUE, message=FALSE, cache=TRUE}
h2o.performance(rf,train = TRUE)
```

```{r echo=TRUE, message=FALSE, cache=TRUE, results=FALSE}
# Next, a Gradient Boosting Classification agorithm was fitted.
gbm <- h2o.gbm(training_frame = as.h2o(training_df), #Training dataframe
               validation_frame = as.h2o(validation_df), #Validation dataframe
               y = 1, #The Dependent Variable
               nfolds = 10, #10-fold cross-validation
               ntrees = 50, #number of trees
               seed = 231 #(time-based random number)
               )
```

```{r echo=TRUE, message=FALSE, cache=TRUE, results=TRUE}
h2o.performance(gbm,train = TRUE)
```
As the perforemnce results for each algorithm shows, both algorithms produce somewhat satisfactory results. However, considering the overall accuracy of each algorithm (i.e., `r round(h2o.hit_ratio_table(rf, train = T)[1,2]*100,2)` percent for Random Forest and `r round(h2o.hit_ratio_table(gbm, train = T)[1,2]*100,2)` percent for Gradient Boosting Classification(GBM)) it seems that GBM is offereing a superior performance in predicting the activity classes for the training dataset.

However, as the models has been built based on the training dataset, the performance of each model in predicting the activity classes for the validiation dataset is more important and less biased. Hence, comparing `r round(h2o.hit_ratio_table(rf, valid = T)[1,2]*100,2)` percent for Random Forest and `r round(h2o.hit_ratio_table(gbm, valid = T)[1,2]*100,2)` percent for GBM, it seems that Random Forest offers a more accurate prediction with less *out-of-sample-error*. This difference might be explained by the potential for over-fitting with GBM algorithm. In the following, a full performance metrics of each algorithm on the validation dataset has been revealed.
```{r echo=TRUE, message=FALSE, cache=TRUE,}
h2o.performance(rf,valid = TRUE)
plot(rf)
h2o.performance(gbm,valid = TRUE)
plot(gbm)
```

Comparing a few more metrics given in the performance results above, it seems Random Forest is the algorithm to go on with. For example, using the Mean per-Class Error metric, we can see that while GBM results in on average of 1.5 % error in classifying each activity in its correct class, this value for Random Forest is only 0.6%.
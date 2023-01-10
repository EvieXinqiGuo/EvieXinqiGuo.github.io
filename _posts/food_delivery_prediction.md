---
layout: post
title: "Food Delivery Time Expectation"
subtitle: "Code Review in R to Ensure Reproducibility and Accuracy of Statistical Analyses in Research"
date: 2021-02-04 14:45:13 -0400
background: ''
---

```{r knitr.global.options.TYT, include=F}
knitr::opts_chunk$set(echo=F, 
                      warning=F, 
                      message = F,
                      fig.align='center', 
                      fig.pos='H',
                      fig.width=12, 
                      fig.height=8, 
                      fig.path='Figs/', 
                      tidy.opts=list(width.cutoff=60),
                      tidy=TRUE)
```
# Overview of the challenge
![](https://ahseeit.com//king-include/uploads/2019/02/50775485_137073873988501_8263291442820303042_n-7723924764.jpg) 

# Food delivery companies connects consumers with their favorite resturants through machine-learning-backed service. Once the order arrives late, the convinence due to delivery now becomes frustration.
# To ensure that the prediction of delivery time closely reflects the actual delivery time, information from the three sides involved - the store, the dashers' capacity, and the consumer's order - all needs to be taken into consideration.
# Conceptually, the estimated delivery duration after the customer placing an order can be divided into these components (it's not necessarily a chain of events since some can be done in paralell): 
- The time that a store takes to receive an order from DoorDash
- The time for a store to repare for an order
- Assigning the order to the right dasher
- The dasher travels to the store
- The dasher picks up order and goes to the destination
- The time of dashing trying to find the specific location of delivery after getting off delivery vehicle

# Overview of the approach
* New features like "Weekday or Weekend" and "Day or Night" for are created for each delivery job based on exisiting date-and-time data to extract information useful for time prediction. These features makes the model more accurate since it captures the clustered effects like stores' kitchens being busier during weekends than weekdays, and delivery jobs with daylight being eaiser and faster than they are after dark. 
* Other potential new features (e.g. average parking time at a specific lcoation) that could further improve the model are raised and discussed below. 
* The potential models of time prediction are evaluated both by the mean of abosolute errors with the actual delivery time and a categorical metric that better fulfills the business goal. Specifically, the categorical metric separates delivery predictions based on their over/underestimation of the actual duration. And there are 6 ordinal slots that ranks each delivery prediction from "Very late - over 10 min" to "Very early - over 10 min". When interpreting and evaluating the model performance, the lateness (underestimating the delivery) are penalized more than early estimations (overestimating the delivery time), and very late/early deliveries should be viewed as stronger signals of call for model improvement than being slightly late/early.

## Models used
* A linear regression model was created first. The model does well in keeping the overall late delivary within 15%, and the very-late (over 5 minutes) deliveries is around 6.9%. But the RMSE score is high - 1672.92. Note: Overall lateness is defined as the proportion of late delivery, which ranges from very late (over 10 min) to about on time (late within 5 min).
* A mixed effect model with the store_id as random effects (meaning that we can later take these irrelevant clustering effect out to make our estimation of the meaningful variables more accurate). The mixed effect model outperformed the linear regression on both the RMSE (1627.13) and the lateness (very late: 6.3%, overall lateness: 14.1%). 
* A XGBoost model also performs well with a low "very late" (6.8%) delivery and slightly higher overal lateness (14.9%) than the mixed effect model . Although the cross validation RMSE of the XGBoost model (1478.721) is much lower.
* As those the XGBoost and the mixed effect model are very different, averaging predictions is likely to improve the predictions. In the prediction for submission, I averaged the prediction between the XGBoost model and the mixed effect model, with higher weight given to the XGBoost model.

# Introduction and potential NEW features to add

The dataset provides 16 important aspects of a typical delivery. But there are much more could influence the time cost of a duration: 

## Below I raise and discuss 5 potential new features that could improve the model performance, if collected
### 1. Community/Zipcode
The location of the destinations alters the delivery time. Destinations in suburb areas have easier access and parking than locations in urbdan enviroment, where tall buildings and stricter security is more common. Further, during weekdays the parking at urban area might be harder than suburb community, and this flunctuation in parking time throughout the week is not captured by the existing estimated point-to-point duration.

### 2. Condo vs. Home
This feature provides a more detailed account of the environments of the destination than the Community feature above. Condos are in buildings (with stairs and/or security entrance), which means longer time is needed between getting off the delivery car and arriving at the customer's door. 

### 3. Weather
Although the estimated point-to-point delivery by other models captures the weather of that day, the impact of weather on the first and the last mile of the delivery might still impacted by the weather. For instance, potential delay in delivery caused by heavy precipitation can be taken into calculation with this new feature.

### 4. Vehicle of delivery
Whether the vehicle of delivery has four wheels or two wheels significantly impact how big of an order a dasher can take. This would make the number of available dashers for a certain order more accurate, which enhances the capacity prediction. 

### 5. Average parking time at a specific location 
As more and more datapoints accumulate, the average time to park at a specific location would be more accurate. This feature could siginificantly improve the model fit.

### Further reading: A relevant paper I found online [Order Fulfillment Cycle Time Estimation for On-Demand Food Delivery by Zhu et al.](https://dl.acm.org/doi/abs/10.1145/3394486.3403307?casa_token=flXKViuOCpcAAAAA:oho4jzUswXyJza_ZbBTeap2mKJ2NP1T_bqHsMOAFInuqb2WM2Pa8lrfBU2yjtbUc1cXFXw1xX8m2)

# Loading and Exploring Existing Data

##Loading libraries required and reading the data into R

```{r, message=FALSE, warning=FALSE}
library(readr)
library(knitr)
library(ggplot2)
library(plyr)
library(dplyr)
library(corrplot)
library(caret)
library(gridExtra)
library(scales)
library(Rmisc)
library(ggrepel)
library(psych)
library(xgboost)
library(skimr)
library(lme4) 
library(merTools)
library(mice)
library(lubridate)


library(apa)
library(citr)
library(papaja)
library(MOTE)
library(tidyverse)
library(kableExtra)
```

Below, I am reading the csv's as dataframes into R.

```{r importing datasets}
train_raw = read_csv("/Users/guoxinqieve/Applications/OneDrive - UC San Diego/DD_takeHomeExercise/historical_data.csv")
test_raw =  read_csv("/Users/guoxinqieve/Applications/OneDrive - UC San Diego/DD_takeHomeExercise/predict_data.csv")
```

#Pre-processing - data exploration, recoding, and cleaning

```{r Overview}
# overview of the variables
glimpse(train_raw)
glimpse(test_raw)
```

```{r exposing the missing values in each variable and dealing with outliers}
# looking at missing values in each column
skimmed_train = skim(train_raw)
View(skimmed_train) 
# total_onshift_dashers, total_busy_dashers, total_outstanding_orders all have 16262 NAs. They probably come from the same deliveries.
# store_primary_category has 	4760 NAs
# market_id and order_protocol has 987 and 995 NAs, repectively
# estimated_store_to_consumer_driving_duration	526 NAs
# actual_delivery_time	7 NAs


skimmed_test = skim(test_raw)
View(skimmed_test) 
# total_onshift_dashers, total_busy_dashers, total_outstanding_orders all have 4633 NAs. They probably come from the same deliveries.
# store_primary_category has 	1343 NAs
# market_id and order_protocol has 250 and 283 NAs, repectively
# estimated_store_to_consumer_driving_duration	526 NAs
# estimated_order_place_duration	11 NAs

```

```{r recoding the numeric columns that are supposed to be categorical}
# recoding to make the variables type match the description (some numeric columns should be categorical)
train = train_raw %>%
  mutate(market_id = as.character(market_id), 
         store_id = as.character(store_id),
         store_primary_category = as.character(store_primary_category),
         order_protocol = as.character(order_protocol))

test = test_raw %>%
  mutate(market_id = as.character(market_id), 
         store_id = as.character(store_id),
         store_primary_category = as.character(store_primary_category),
         order_protocol = as.character(order_protocol))
```

```{r visualizing variable distribution and excluding obvious outliers}
summary(train$created_at) # created_at No NAs, but has a weird minimal: "2014-10-19 05:24:15" this should be excluded

train$created_at[year(train$created_at) ==2014] = NA # now the earliest created_at is in year 2015

summary(train$actual_delivery_time) # actual_delivery_time has 7 NAs, other than that looks normal

# looking at the distribution of the type of cuisines
table(train$store_primary_category) # some categories are much more popular than the others.


# looking at the distribution of the means of how stores accpet DoorDash order
table(train$order_protocol) # order_protocol has a lot of NAs, ranges from 1 to 7, are distinct, some are much more popular than others
summary(train$order_protocol)

# looking at the number of items in total
summary(train$total_items) # ranges form 1 to more than 411, No NAs
hist(train$total_items)

#looking at the subtotal of the order
summary(train$subtotal) # the min is zero, the median is 2200, highest 27100

#looking at the # of unique items of the order
summary(train$num_distinct_items) # no NAs, ranges from 1 to 20
hist(train$num_distinct_items)

#looking at the minimum item price
summary(train$min_item_price) # has negative numbers, like -86, need to exclude it. 
hist(train$min_item_price)
train$min_item_price[train$min_item_price < 0] = NA # after the exclusion, the price for min-price is 0

#looking at the maximum item price
summary(train$max_item_price) # min is 0
hist(train$max_item_price)

## the min and maximum item price are not very interpretable or informative. Further, many delivery's minimum and maximum item price are overlapping, since they just ordered one item. The essential info can be captured by subtotal and # of items. They will be removed from the models. 



# Looking at the supply side of the delivery capacity --- on-shift dasher
table(train$total_onshift_dashers) # has a couple of negative values, will leave them as it is since the train data also has negative values.
hist(train$total_onshift_dashers)

# looking at busy dashers
table(train$total_busy_dashers) # has a couple of negative values, will leave them as it is since the train data also has negative values.
hist(train$total_busy_dashers)

# looking at the outstanding orders, which reduces the capacity
table(train$total_outstanding_orders) # has a couple of negative values, will leave them as it is since the train data also has negative values.
hist(train$total_outstanding_orders)


# looking at the estimated duration to for the resturant to receive an order
summary(train$estimated_order_place_duration) # min is 0, max is 2715

# looking at the estimated point-to-point delivery time
summary(train$estimated_store_to_consumer_driving_duration) # has many NA values, min is 0, max is 2088
```

At this point of pre-processing, it becomes clear that variables like maximum and minimum item prices are neither meaningful nor necessary for predicting delivery time. The subtotal and number of items can capture information about the food ordered that is relevant to the time prediction.

## Based on the existing data, some useful new features like "Weekday or Weekend" and "Day or Night" can be created 
```{r adding features based on existing data}
train = train %>%
  mutate(actual_duration_sec = as.numeric(actual_delivery_time -created_at)*60,
         # the new variable captures the time that cannot be captured by the point-to-point and order-receive time
         diff_actual_estimate_sec = actual_duration_sec - (estimated_store_to_consumer_driving_duration + estimated_order_place_duration),
       
          DayOrNight = case_when(
           hour(train$created_at) >= 7 & hour(train$created_at) < 18 ~ "Day",
           TRUE ~ "Night"),
         
         WeekdayOrWeekend = case_when(
           weekdays(train$created_at) == "Friday" ~ "Weekend",
           weekdays(train$created_at) == "Saturday" ~ "Weekend",
           weekdays(train$created_at) == "Sunday" ~ "Weekend",
           TRUE ~ "Weekday")
    
    )


test = test %>%
  mutate( DayOrNight = case_when(
           hour(test$created_at) >= 7 & hour(test$created_at) < 18 ~ "Day",
           TRUE ~ "Night"),
         
         WeekdayOrWeekend = case_when(
           weekdays(test$created_at) == "Friday" ~ "Weekend",
           weekdays(test$created_at) == "Saturday" ~ "Weekend",
           weekdays(test$created_at) == "Sunday" ~ "Weekend",
           TRUE ~ "Weekday")
    
    )
```

### Since I decided to not include uninformative variables like maximum item price when predicting, I'll only select relevant columns to be included in the cleaned train and raw dataset.
```{r selecting relevant data for model building}

selected_data = as.data.frame(train[, c(1,4:8, 12:16, 18:20)])


selected_data_test = as.data.frame(test[, c(1, 3: 7, 11:15, 18, 19, 16)])

```

## Selecting metrics
I'd evaluate the models based on the aspect that matters to the business goal the most. Since severe underestimation of delivery time (i.e., being late over 10 minutes) would be really painful for the customers, the model that better reduce the late over 10 minute would be better. Further, the penalization of being late, even within 5 minutes, should be more than the being early when comparing the existing and the potential new model.

## The first model is a linear regression model with all the variables left after feature engineering as predictors. The second model is a linear mixed effect model with store_id as a random effect (since we are not intestered in the clustering effect introduced by each specific stores).

### The linear regression model and the mixed effect model outcomes were compared based on their Root Mean Square Error (RMSE) and their percentage of underestimating the delivery time (being late is bad).
### The mixed effect model outperforms the linear regression model in both RMSE and the reduced overall lateness (14.93% -> 14.1%) and reduced significant-late (6.9% -> 6.3%). The predicted values from the mixed effect model will be keep to compare with the next model.
```{r lm and lmer}
set.seed(100)
TrainingIndex <- createDataPartition(selected_data$diff_actual_estimate_sec[!is.na(selected_data$diff_actual_estimate_sec)], p=0.8, list = FALSE)
TrainingSet <- selected_data[TrainingIndex,] # Training Set
TestingSet <- selected_data[-TrainingIndex,] # Test Set

# below is a linear regression model with store_id removed, further, the primary store category of a store is also removed since new categories were added to the testing dataset
## the code below shows that there are new store categories
train_category = as.data.frame(table(selected_data$store_primary_category))$Var1
test_category = as.data.frame(table(selected_data_test$store_primary_category))$Var1
new_categoryInTesting = unique(train_category[! train_category %in% test_category]) #belgian   chocolate lebanese 
old_categoryOnlyInTraining = unique(test_category[! test_category %in% train_category]) # non of them are new to the test dataset but old to the train dataset

lm_model <- lm(diff_actual_estimate_sec ~ market_id + order_protocol+ total_items + subtotal+total_onshift_dashers+total_busy_dashers+ total_outstanding_orders + DayOrNight + WeekdayOrWeekend+ estimated_order_place_duration+  estimated_store_to_consumer_driving_duration, data = TrainingSet)


#### rsme for lm model
#### formula from https://stackoverflow.com/questions/43123462/how-to-obtain-rmse-out-of-lm-result
RSS <- c(crossprod(lm_model$residuals))
MSE <- RSS / length(lm_model$residuals) # faster than mean()
RMSE <- sqrt(MSE) ##  1672.919

predicted_lm <-predict(lm_model,newdata=TestingSet)


 lm_error = TestingSet$diff_actual_estimate_sec- (predicted_lm + TestingSet$estimated_order_place_duration + TestingSet$estimated_store_to_consumer_driving_duration) 
ErrorEvaluation_lm = data.frame(lm_error = lm_error,
                             EarlyLateCategory = case_when(lm_error > 10*60 ~ "Late Over 10 Min",
                                                                lm_error <= 10*60 &  lm_error > 5*60 ~ "Late 5-10 Min",
                                                                lm_error <= 5*60 & lm_error > 0 ~ "Late within 5 Min",
                                                                lm_error <= 0 & lm_error > -5*60 ~ "Early within 5 Min",
                                                                lm_error <= -5*60 & lm_error > -10*60 ~"Early 5-10 Min",
                                                                lm_error <= - 10*60 ~ "Early more than 10 Min"
                                                                ) ) 
## looking at the distribution of the early and late category, there are currently 

ErrorEvaluation_lm_EarlyLateCategory = as.data.frame(prop.table(table(ErrorEvaluation_lm$EarlyLateCategory)))

colnames(ErrorEvaluation_lm_EarlyLateCategory) = c("Early Late Category", "Frequency")

level_category <- c("Early more than 10 Min", "Early 5-10 Min", "Early within 5 Min", "Late within 5 Min",  "Late 5-10 Min","Late Over 10 Min")

ErrorEvaluation_lm_EarlyLateCategory = ErrorEvaluation_lm_EarlyLateCategory %>%
  mutate(`Early Late Category`=  factor(`Early Late Category`, levels = level_category)) %>%
  arrange(`Early Late Category`)

table_ErrorEvaluation_lm_EarlyLateCategory =ErrorEvaluation_lm_EarlyLateCategory %>%
  knitr::kable(digits = 3, caption = "Error Evaluation Linear Regression Model Early Late Category") %>%
  kableExtra::kable_styling(latex_options = "scale_down")


# the simple linear model also does well in having an overall lateness of 14.93%. But its very late category (6.9%) is higher than the mixed effect model prediction (below)



lmer_model <- lmer(diff_actual_estimate_sec ~ market_id  + order_protocol+ total_items + subtotal+total_onshift_dashers+total_busy_dashers+ total_outstanding_orders+ estimated_order_place_duration+  estimated_store_to_consumer_driving_duration+ DayOrNight + WeekdayOrWeekend + (1|store_id), data = TrainingSet)


lmer_rmse = RMSE.merMod(lmer_model) #  1627.128
predicted_lmer <-predict(lmer_model,newdata=TestingSet,  allow.new.levels=TRUE)

 lmer_error = TestingSet$diff_actual_estimate_sec- (predicted_lmer + TestingSet$estimated_order_place_duration + TestingSet$estimated_store_to_consumer_driving_duration) 
ErrorEvaluation_lmer = data.frame(lmer_error = lmer_error,
                             EarlyLateCategory = case_when(lmer_error > 10*60 ~ "Late Over 10 Min",
                                                                lmer_error <= 10*60 &  lmer_error > 5*60 ~ "Late 5-10 Min",
                                                                lmer_error <= 5*60 & lmer_error > 0 ~ "Late within 5 Min",
                                                                lmer_error <= 0 & lmer_error > -5*60 ~ "Early within 5 Min",
                                                                lmer_error <= -5*60 & lmer_error > -10*60 ~"Early 5-10 Min",
                                                                lmer_error <= - 10*60 ~ "Early more than 10 Min"
                                                                ) )
prop.table(table(ErrorEvaluation_lmer$EarlyLateCategory)) # The vast majority of the delivery is early under this model, the overall lateness is 14.1%. The the very (over 10 min) late category is around 6.3% percent


ErrorEvaluation_lmixed_EarlyLateCategory = as.data.frame(prop.table(table(ErrorEvaluation_lmer$EarlyLateCategory)))

colnames(ErrorEvaluation_lmixed_EarlyLateCategory) = c("Early Late Category", "Frequency")

level_category <- c("Early more than 10 Min", "Early 5-10 Min", "Early within 5 Min", "Late within 5 Min",  "Late 5-10 Min","Late Over 10 Min")

ErrorEvaluation_lmixed_EarlyLateCategory = ErrorEvaluation_lmixed_EarlyLateCategory %>%
  mutate(`Early Late Category`=  factor(`Early Late Category`, levels = level_category)) %>%
  arrange(`Early Late Category`)

table_ErrorEvaluation_lmixed_EarlyLateCategory =ErrorEvaluation_lmixed_EarlyLateCategory %>%
  knitr::kable(digits = 3, caption = "Error Evaluation Linear MIXED Model Early Late Category") %>%
  kableExtra::kable_styling(latex_options = "scale_down")

```

```{r TableEarlyLateLM, echo=FALSE}  
  table_ErrorEvaluation_lm_EarlyLateCategory
```

```{r TableEarlyLateLMixed, echo=FALSE}  
  table_ErrorEvaluation_lmixed_EarlyLateCategory
```

## The third prediction model for consideration is a XGBoost model

First, I used one-hot encoding to ensure that training is a numeric metrix before the XGBoost modeling.
```{r one-hot-encoding for training data}
# one_hot_encoding
dummy <- dummyVars(" ~ .", data=as.data.frame(TrainingSet[, c(1, 4, 13, 14)]))

newdata <- data.frame(predict(dummy, newdata = as.data.frame(TrainingSet[, c(1, 4, 13, 14)])))
newdata[is.na(newdata)] = 0

# imputing the missing values for the xgboost
RestOfOneHotEncoded = as.data.frame(TrainingSet[, -c(1: 4, 13, 14)])
RestOfOneHotEncoded$diff_actual_estimate_sec[is.na(RestOfOneHotEncoded$diff_actual_estimate_sec)]<-median(RestOfOneHotEncoded$diff_actual_estimate_sec,na.rm=TRUE)
RestOfOneHotEncoded$estimated_store_to_consumer_driving_duration[is.na(RestOfOneHotEncoded$estimated_store_to_consumer_driving_duration)]<-median(RestOfOneHotEncoded$estimated_store_to_consumer_driving_duration,na.rm=TRUE)
RestOfOneHotEncoded$total_outstanding_orders[is.na(RestOfOneHotEncoded$total_outstanding_orders)]<-median(RestOfOneHotEncoded$total_outstanding_orders,na.rm=TRUE)
RestOfOneHotEncoded$total_onshift_dashers[is.na(RestOfOneHotEncoded$total_onshift_dashers)]<-median(RestOfOneHotEncoded$total_onshift_dashers,na.rm=TRUE)
RestOfOneHotEncoded$total_busy_dashers[is.na(RestOfOneHotEncoded$total_busy_dashers)]<-median(RestOfOneHotEncoded$total_busy_dashers,na.rm=TRUE)

OneHotEncoded = cbind(newdata, RestOfOneHotEncoded)

```


```{r one-hot encoding for test data}

# one_hot_encoding
dummy_test <- dummyVars(" ~ .", data=as.data.frame(TestingSet[, c(1, 4, 13, 14)]))

newdata_test <- data.frame(predict(dummy_test, newdata = as.data.frame(TestingSet[, c(1, 4, 13, 14)])))
newdata_test[is.na(newdata_test)] = 0

# imputing the missing values for the xgboost
RestOfOneHotEncoded_test = as.data.frame(TestingSet[, -c(1: 4, 13, 14)])
RestOfOneHotEncoded_test$estimated_store_to_consumer_driving_duration[is.na(RestOfOneHotEncoded_test$estimated_store_to_consumer_driving_duration)]<-median(RestOfOneHotEncoded_test$estimated_store_to_consumer_driving_duration,na.rm=TRUE)
RestOfOneHotEncoded_test$total_outstanding_orders[is.na(RestOfOneHotEncoded_test$total_outstanding_orders)]<-median(RestOfOneHotEncoded_test$total_outstanding_orders,na.rm=TRUE)
RestOfOneHotEncoded_test$total_onshift_dashers[is.na(RestOfOneHotEncoded_test$total_onshift_dashers)]<-median(RestOfOneHotEncoded_test$total_onshift_dashers,na.rm=TRUE)
RestOfOneHotEncoded_test$total_busy_dashers[is.na(RestOfOneHotEncoded_test$total_busy_dashers)]<-median(RestOfOneHotEncoded_test$total_busy_dashers,na.rm=TRUE)

OneHotEncoded_test = cbind(newdata_test, RestOfOneHotEncoded_test)
```

```{r constructing Dmatrixs objects}

# put our testing & training data into two seperates Dmatrixs objects
dtrain <- xgb.DMatrix(data = as.matrix(OneHotEncoded[, colnames(OneHotEncoded) != "diff_actual_estimate_sec"]), label= OneHotEncoded$diff_actual_estimate_sec)
dtest <- xgb.DMatrix(data = as.matrix(OneHotEncoded_test[, colnames(OneHotEncoded_test) != "diff_actual_estimate_sec"]))
```

In addition, I am taking over the best tuned values from the caret cross validation.

```{r XGBoost parameters}
#### parameters inspired by https://www.kaggle.com/erikbruin/house-prices-lasso-xgboost-and-a-detailed-eda/report
default_param<-list(
        objective = "reg:squarederror",
        booster = "gbtree",
        eta=0.03,
        gamma=0,
        max_depth=3, #default=6
        min_child_weight=1, #default=1
        subsample=1,
        colsample_bytree=1
)
```

"The next step is to do cross validation to determine the best number of rounds (for the given set of parameters)." 
```{r XGBoost Cross Validation}
set.seed(100)
xgbcv <- xgb.cv( params = default_param, data = dtrain, nrounds = 500, nfold = 7, showsd = T, stratified = T, print_every_n = 50, early_stopping_rounds = 10, maximize = F)

#[1]	train-rmse:2578.948591+82.418023	test-rmse:2540.076835+453.918489 
#Multiple eval metrics are present. Will use test_rmse for early stopping.
#Will train until test_rmse hasn't improved in 10 rounds.

#[51]	train-rmse:1689.529698+121.945481	test-rmse:1581.529367+651.528071 
#[101]	train-rmse:1600.293596+120.399810	test-rmse:1506.473179+671.641646 
#[151]	train-rmse:1579.996565+119.180641	test-rmse:1494.741926+674.094247 
#[201]	train-rmse:1563.757132+117.390876	test-rmse:1490.016200+674.652216 
#[251]	train-rmse:1550.746966+114.153220	test-rmse:1485.470319+675.604640 
#[301]	train-rmse:1540.295776+112.428664	test-rmse:1483.263802+675.638323 
#[351]	train-rmse:1532.593663+112.149902	test-rmse:1481.385594+675.840846 
#[401]	train-rmse:1525.220895+110.456893	test-rmse:1480.015119+676.033333 
#[451]	train-rmse:1519.397914+109.869758	test-rmse:1478.765102+676.181584 
#Stopping. Best iteration:
#[442]	train-rmse:1520.736834+110.449768	test-rmse:1478.721776+676.307824
```

```{r train model with best iteration}
#train the model using the best iteration found by cross validation
xgb_mod <- xgb.train(data = dtrain, params=default_param, nrounds = 442)
```

```{r XGBoost Prediction}
XGBpred <- predict(xgb_mod, dtest)

head(XGBpred)
```

After obtaining the RMSE score, I use the pre-constructed early/late category to evaluate the XGBoost model. As presented below, the result shows that this model is by large overestimating the delivery time, which is less evil than underestimating the delivery time. However, its percentage of being very (over 10 min) early ranks between the linear regression and the linear mixed effect model.
```{r xgboost earlyLateCategory}
xgboost_error = TestingSet$diff_actual_estimate_sec- (round(XGBpred) + TestingSet$estimated_order_place_duration + TestingSet$estimated_store_to_consumer_driving_duration) 
xgboostErrorEvaluation = data.frame(xgboost_error = xgboost_error,
            EarlyLateCategory = case_when(xgboost_error > 10*60 ~ "Late Over 10 Min",
                                          xgboost_error <= 10*60 &  xgboost_error > 5*60 ~ "Late 5-10 Min",
                                          xgboost_error <= 5*60 & xgboost_error > 0 ~ "Late within 5 Min",
                                          xgboost_error <= 0 & xgboost_error > -5*60 ~ "Early within 5 Min",
                                          xgboost_error <= -5*60 & xgboost_error > -10*60 ~"Early 5-10 Min",
                                          xgboost_error <= - 10*60 ~ "Early more than 10 Min"
                                                                ) ) 
## looking at the distribution of the early and late category, there are currently 
prop.table(table(xgboostErrorEvaluation$EarlyLateCategory)) 
# this result indicates the XGBoost model has significantly lower RMSE, but higher "late over 10 min" percentage. The overall lateness of the XGBoost (14.9%) is similar with the rest of the two models

# Early 5-10 Min Early more than 10 Min     Early within 5 Min          Late 5-10 Min       Late Over 10 Min 
#            0.09628234             0.68098468             0.07050992             0.03323286             0.07113791 
#     Late within 5 Min 
 #           0.04785230 


ErrorEvaluation_xgboost_EarlyLateCategory = as.data.frame(prop.table(table(xgboostErrorEvaluation$EarlyLateCategory)) )

colnames(ErrorEvaluation_xgboost_EarlyLateCategory) = c("Early Late Category", "Frequency")

level_category <- c("Early more than 10 Min", "Early 5-10 Min", "Early within 5 Min", "Late within 5 Min",  "Late 5-10 Min","Late Over 10 Min")

ErrorEvaluation_xgboost_EarlyLateCategory = ErrorEvaluation_xgboost_EarlyLateCategory %>%
  mutate(`Early Late Category`=  factor(`Early Late Category`, levels = level_category)) %>%
  arrange(`Early Late Category`)

table_ErrorEvaluation_xgboost_EarlyLateCategory =ErrorEvaluation_xgboost_EarlyLateCategory %>%
  knitr::kable(digits = 3, caption = "Error Evaluation XGBoost Model Early Late Category") %>%
  kableExtra::kable_styling(latex_options = "scale_down")

```

```{r TableEarlyLateXGBoost, echo=FALSE}  
  table_ErrorEvaluation_xgboost_EarlyLateCategory
```


## By evaluating the RMSE and lateness category of the XGBoost model, I found out that the XGBoost's RMSE score is much better than the linear regression and mixed effect models. However, the mixed effect model has lower "very late" delivery than the XGBoost model. Thus, in the submission, I average the XGBoost and the mixed effect model prediction, with a heavier weight given to the XGBoost model.
```{r submission}
# one_hot_encoding for submission data
dummy_submit <- dummyVars(" ~ .", data=as.data.frame(test[, c(1, 5, 18, 19)])) # the platform feature is new. I omit it since it's absent in the training data

newdata_submit<- data.frame(predict(dummy_submit, newdata = as.data.frame(test[, c(1, 5, 18, 19)])))
newdata_submit[is.na(newdata_submit)] = 0

# imputing the missing values for the xgboost
RestOfOneHotEncoded_submit = as.data.frame(test[, -c(1: 5,8:10,16: 19)])
RestOfOneHotEncoded_submit$estimated_store_to_consumer_driving_duration[is.na(RestOfOneHotEncoded_submit$estimated_store_to_consumer_driving_duration)]<-median(RestOfOneHotEncoded_submit$estimated_store_to_consumer_driving_duration,na.rm=TRUE)
RestOfOneHotEncoded_submit$total_outstanding_orders[is.na(RestOfOneHotEncoded_submit$total_outstanding_orders)]<-median(RestOfOneHotEncoded_submit$total_outstanding_orders,na.rm=TRUE)
RestOfOneHotEncoded_submit$total_onshift_dashers[is.na(RestOfOneHotEncoded_submit$total_onshift_dashers)]<-median(RestOfOneHotEncoded_submit$total_onshift_dashers,na.rm=TRUE)
RestOfOneHotEncoded_submit$total_busy_dashers[is.na(RestOfOneHotEncoded_submit$total_busy_dashers)]<-median(RestOfOneHotEncoded_submit$total_busy_dashers,na.rm=TRUE)

OneHotEncoded_submit = cbind(newdata_submit, RestOfOneHotEncoded_submit)

dsubmit <- xgb.DMatrix(data = as.matrix(OneHotEncoded_submit))

XGB_pred_submmit <- predict(xgb_mod, dsubmit)

head(XGB_pred_submmit)

sub_avg <- data.frame(delivery_id = test$delivery_id, 'predicted duration' = (2*XGB_pred_submmit+predict(lmer_model,newdata=test,  allow.new.levels=TRUE))/3)
head(sub_avg)

write.csv(sub_avg,"/Users/guoxinqieve/Applications/OneDrive - UC San Diego/DD_takeHomeExercise/XinqiGuo_DoorDashSubmission.csv", row.names = F)

```





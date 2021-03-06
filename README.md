# Telcom-Customer-Churn-Analysis
The analysis focuses on telecom customers who are likely to leave the platform.In this report we have used Decision Tree, Random forest and Support Vector Machine to predict the customers who are more likely to churn.

#### Load the necessary packages
```{r message = F, warning = F}
library(ggplot2)
library(tidyverse)
library(dplyr)
library(tidyverse)
library(ggcorrplot)
library(caret)
library(caTools)
library(randomForest)
library(e1071)
library(pROC)
library(ROCR)
library(data.table)
```


#### Storing the data

```{r message = F, warning = F}
data <- read.csv("WA_Fn-UseC_-Telco-Customer-Churn.csv", header = TRUE)

```


```{r}
str(data)
```


```{r}
summary(data)
```

Check for NA values:

```{r}
sapply(data, function(x) sum(is.na(x)))
```

There are only 11 missing data in the TotalCharges field, so we can remove those rows from the dataset.

```{r}
data<-na.omit(data)
```

SeniorCitizen is in 'int' form, that can be changed to categorical.

```{r}
data$SeniorCitizen <- as.factor(ifelse(data$SeniorCitizen==1, 'YES', 'NO'))
```

Removing the duplicates:

```{r message = F, warning = F}
data <- data[!duplicated(data), ]
```


Visualization:

```{r}
data %>% 
group_by(Churn) %>% 
summarise(Count = n())%>% 
mutate(percent = prop.table(Count)*100)%>%
ggplot(aes(reorder(Churn, -percent), percent), fill = Churn)+
geom_col(fill = c("brown", "gold"))+
geom_text(aes(label = sprintf("%.2f%%", percent)), hjust = 0.01,vjust = -0.5, size =3)+ 
theme_bw()+  
xlab("Churn") + 
ylab("Percent")+
ggtitle("Churn Percent")
```


Around 26% of customers left the platform within the last month

Tenure: The median tenure for customers who have left is around 10 months.

```{r}
ggplot(data, aes(y= tenure, x = "", fill = Churn)) + 
geom_boxplot()+ 
theme_bw()+
xlab(" ")
```

MonthlyCharges: Customers who have churned, have high monthly charges. The median is above 75.

```{r}
ggplot(data, aes(y= MonthlyCharges, x = "", fill = Churn)) + 
geom_boxplot()+ 
theme_bw()+
xlab(" ")
```


TotalCharges:* The median Total charges of customers who have churned is low.
```{r}
ggplot(data, aes(y= TotalCharges, x = "", fill = Churn)) + 
geom_boxplot()+ 
theme_bw()+
xlab(" ")
```



#### **Data Preparation**:

From the EDA above, we know that there are some categorical features that have 'No' and 'No Internet Service' or 'No Phone Service' as a category, we can make them as 'No' and clean these features.


```{r}
data <- data.frame(lapply(data, function(x) {
                  gsub("No internet service", "No", x)}))

data <- data.frame(lapply(data, function(x) {
                  gsub("No phone service", "No", x)}))

data$Churn <- as.factor(data$Churn)

data$customerID<-NULL

```


Standardising Continuous features

```{r}
num_columns <- c("tenure", "MonthlyCharges", "TotalCharges")
data[num_columns] <- sapply(data[num_columns], as.numeric)

data_int <- data[,c("tenure", "MonthlyCharges", "TotalCharges")]
data_int <- data.frame(scale(data_int))
```


I am trying to create a derived feature from tenure, where i have made different bins of tenure(which is in months) such as '0-1 year', '2-3 years', '3-4 years' etc.

```{r}
data <- mutate(data, tenure_bin = tenure)

data$tenure_bin[data$tenure_bin >=0 & data$tenure_bin <= 12] <- '0-1 year'
data$tenure_bin[data$tenure_bin > 12 & data$tenure_bin <= 24] <- '1-2 years'
data$tenure_bin[data$tenure_bin > 24 & data$tenure_bin <= 36] <- '2-3 years'
data$tenure_bin[data$tenure_bin > 36 & data$tenure_bin <= 48] <- '3-4 years'
data$tenure_bin[data$tenure_bin > 48 & data$tenure_bin <= 60] <- '4-5 years'
data$tenure_bin[data$tenure_bin > 60 & data$tenure_bin <= 72] <- '5-6 years'

data$tenure_bin <- as.factor(data$tenure_bin)
```

Creating Dummy Variables:

```{r}
data_cat <- data[,-c(1,6,19,20)]

dummy<- data.frame(sapply(data_cat,function(x) data.frame(model.matrix(~x-1,data =data_cat))[,-1]))

head(dummy)
```


Creating the final dataset by combining the numeric and dummy data frames:

```{r}
data_final <- cbind(data_int,dummy)
head(data_final)
```


Data Split:

```{r}
set.seed(123)
data_index = sample.split(data$Churn, SplitRatio = 0.7)
train = data[data_index,]
validation = data[!(data_index),]
```

#### **Decision Tree**:

Decision Tree algorithm is a supervised learning algorithms. Unlike other supervised learning algorithms, decision tree algorithm can be used for solving regression and classification problems too.

The general motive of using Decision Tree is to create a training model which can use to predict class or value of target variables by learning decision rules inferred from prior data(training data).

Let us now perform decision tree analysis on this training data.


```{r message = F, warning = F}
start_time <- Sys.time()
dt_model <- train(Churn ~ ., data = train, method = "rpart",
                  tuneGrid = expand.grid(cp = seq(0, 0.1, 0.01)),
                  trControl = trainControl(method = "cv", number = 3))
dt_time <- Sys.time() - start_time
dt_time
```

```{r}
dt_model
```

Predicting the churn by testing the model on test data:

```{r message = F, warning = F}

predict_dt <- predict(dt_model, newdata = validation[,-24], type = "raw")
confusionMatrix(predict_dt, validation$Churn)
```

Decision tree model predicts the churn with 78.34% accuracy. 


#### **Random Forest**:

Random Forest algorithm can be used for both regression and classification problems. Two methods for tuning the model for, 1. Random Search and 2. Grid Search

Grid Search:
In grid search the model is evaluated with all the combinations that are passed in the function, using cross-validation

Random Search:
Unlike grid search, random search will not evaluate all the combinations of hyperparameters, instead a random combination is chosen at every iteration.

```{r}
start_time <- Sys.time()
rf_model <- randomForest(Churn ~ ., data=train, proximity=FALSE,importance = FALSE,
                        ntree=500,mtry=4, do.trace=FALSE)
rf_model
rf_time <- Sys.time() - start_time
rf_time
```

```{r}
predict_rf <- predict(rf_model, newdata=validation[,-24])
table(predict_rf, validation$Churn)

confusionMatrix(validation$Churn, predict_rf)
```

Random Forest model predicts the churn with 79.48% accuracy.Random forest has better accuracy when compared to the Decision tree model.

```{r}
varImpPlot(rf_model)
```

#### **Support Vector Machine**

A Support Vector Machine (SVM) is a supervised machine learning algorithm that can be employed for both classification and regression purposes. SVMs are more commonly used in classification problems. So, lets go ahead and train the model on the trianing data:

```{r}
start_time <- Sys.time()
svm_fit <- svm(Churn ~ ., probability = TRUE, data = train)
svm_fit
svm_time <- Sys.time() - start_time
svm_time
```


```{r}
pred_svm<- predict(svm_fit, validation, probability = TRUE)
svm_test_probs <- data.frame(attr(pred_svm, "probabilities"))
confusionMatrix(validation$Churn, pred_svm)
```

SVM model predicted the churn with 79.05% accuracy which is quite similar to Random forest.

Checking the AUC for all three models:

```{r,message=FALSE, warning=FALSE}

DT.roc <- roc(response = validation$Churn, predictor = as.numeric(predict_dt))
rf.roc <- roc(response = validation$Churn, predictor = as.numeric(predict_rf))
svm.roc <- roc(response = validation$Churn, predictor = as.numeric(pred_svm))

plot(DT.roc,      legacy.axes = TRUE, print.auc.y = 1.0, print.auc = TRUE)
plot(rf.roc, col = "red" , add = TRUE, print.auc.y = 0.85, print.auc = TRUE)
plot(svm.roc, col = "blue", add = TRUE, print.auc.y = 0.65, print.auc = TRUE)
legend("bottom", c("Random Forest","SVM", "Decision Tree"),
       lty = c(1,1), lwd = c(2, 2), col = c("red", "blue", "black"), cex = 0.75)

```

Comparison of all three models:

```{r}

master_table= data.table(
  Models=c("DT","RF","SVM"),
  Accuracy=c(78.34,79.48,79.05),
  Sensitivity=c(92.64,83.06,81.18),
  specificity=c(38.86,65.17,67.76),
  Approx.Time=c(1.15,3.70,9.86)
  
  
)
master_table
```

On comparing the above three models, Random Forest is considered to be the best model to predict the churn with 79.48% accuracy. Though SVM also got the accuracy of 79.05%, considering the time complexity and accuracy, Random Forest predicts the results better. 


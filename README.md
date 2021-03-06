---
title: "House Price Trend Analysis in King County"
author: "Ken Liu"
date: "11/29/2021"
output:
  pdf_document: default
  html_document: default
---
#### use house data only in the city of Seattle
#### Edit the date variable into year and month and romove some outliers which have super high price levels. 
```{r}
house <- read_csv("kc_house_data.csv")
table(house$zipcode)
# use house data only in the city of Seattle
house <- subset.data.frame(house,house$zipcode > 98100) # use house data only in the city of Seattle
# convert date into year and month
house$year <- substr(house$date, 1, 4)
house$month <- substr(house$date, 5, 6)
# remove houses with extreme price
plot(density(house$price))
house_remove <- house[which(house$price < (2*(10^6))), ]
plot(density(house_remove$price))
house <- house_remove
write.csv(house,"kc_house_data.csv")
```
#### Stratify the price into 4 levels because the characterics of houses in low price level are significantly different from the houses in the high price level. 
```{r}
house <- read.csv("kc_house_data.csv")
which(colnames(house)=="date" )
house <- house[,-3] # no date variable
summary(house)

# min: 78000; 1st Qu: 335375; Mean: 516968; 3rd Qu: 625000; Max: 1999000
house1 <- subset.data.frame(house,price <= 335375) # 2,225
house2 <- subset.data.frame(house,price > 335375 & price<=516968) # 3,151
house3 <- subset.data.frame(house,price > 516968 & price<=625000) # 1,324
house4 <- subset.data.frame(house,price > 625000 & price<=1999000) # 2,200
```

#### The affect of internal variables on housing price (exclude year and month the house sold). We will not use location variables (Zipcode, latitude, and longtitude) in the regression analysis the numbers behind the zipcode don't follow an order. For example, 98102 is not better than 98101. We will use these variables in the next step when we want to map the house and analyze the relationships between housing price and the demographics information in the census tracts. 
```{r}
house.internal.1 <- house1[,-c(1,2,9,17,18,19,22,23)] # not use X, ID, Zipcode, Lat, Long, Year, Month, Waterfront
house.internal.2 <- house2[,-c(1,2,17,18,19,22,23)] # not use X, ID, Zipcode, Lat, Long, Year, Month
house.internal.3 <- house3[,-c(1,2,17,18,19,22,23)] # not use X, ID, Zipcode, Lat, Long, Year, Month
house.internal.4 <- house4[,-c(1,2,17,18,19,22,23)] # not use X, ID, Zipcode, Lat, Long, Year, Month

fit.1 <- lm(price~bedrooms + bathrooms + sqft_living + sqft_lot + floors + view + condition + grade + sqft_above + sqft_basement + yr_built + yr_renovated + sqft_living15 + sqft_lot15, data = house.internal.1) # houses in the lower price range don't have waterfront, so we drop this variable.
corr <- cor(house.internal.1, use="pairwise.complete.obs")
corrplot::corrplot(corr,type="lower")

fit.2 <- lm(price~bedrooms + bathrooms + sqft_living + sqft_lot + floors + waterfront + view + condition + grade + sqft_above + sqft_basement + yr_built + yr_renovated + sqft_living15 + sqft_lot15, data = house.internal.2)
corr <- cor(house.internal.2, use="pairwise.complete.obs")
corrplot::corrplot(corr,type="lower")

fit.3 <- lm(price~bedrooms + bathrooms + sqft_living + sqft_lot + floors + waterfront + view + condition + grade + sqft_above + sqft_basement + yr_built + yr_renovated + sqft_living15 + sqft_lot15, data = house.internal.3)
corr <- cor(house.internal.3, use="pairwise.complete.obs")
corrplot::corrplot(corr,type="lower")

fit.4 <- lm(price~bedrooms + bathrooms + sqft_living + sqft_lot + floors + waterfront + view + condition + grade + sqft_above + sqft_basement + yr_built + yr_renovated + sqft_living15 + sqft_lot15, data = house.internal.4)
corr <- cor(house.internal.4, use="pairwise.complete.obs")
corrplot::corrplot(corr,type="lower")
```

#### external variables (year and month)
```{r}
house.external.1 <- data.frame(house1$price,house1$year,house1$month)
colnames(house.external.1) <- c("price","year","month")
house.external.2 <- data.frame(house2$price,house2$year,house2$month)
colnames(house.external.2) <- c("price","year","month")
house.external.3 <- data.frame(house3$price,house3$year,house3$month)
colnames(house.external.3) <- c("price","year","month")
house.external.4 <- data.frame(house4$price,house4$year,house4$month)
colnames(house.external.4) <- c("price","year","month")

summary(house$price[house$year==2014])
summary(house$price[house$year==2015]) # the yearly effect is small.

fit.5 <- lm(price~month,data=house.external.1)
fit.6 <- lm(price~month,data=house.external.2)
fit.7 <- lm(price~month,data=house.external.3)
fit.8 <- lm(price~month,data=house.external.4)

```

#### different methods

```{r}
fit.1 <- lm(price ~ ., data = house.internal.1) 
summary(fit.1)
# car::vif(fit.1)
# there are aliased coefficients in the model
# so find the culprit
alias(fit.1)
# remove sqft_livin
fit.1_per <- lm(price ~ bedrooms + bathrooms + sqft_lot + floors + view + condition + grade + sqft_above + sqft_basement + yr_built + yr_renovated + sqft_living15 + sqft_lot15, data = house.internal.1)
summary(fit.1_per)
car::vif(fit.1_per)

# stepwise regression
fit.1 <- lm(price ~ bedrooms + bathrooms + sqft_living + sqft_lot + floors + view + condition + grade + sqft_above + sqft_basement + yr_built + yr_renovated + sqft_living15 + sqft_lot15, data = house.internal.1)
step_fit.1 <- step(fit.1, direction = "both")
summary(step_fit.1)
# remain variables bedrooms + sqft_living + floors + view + condition + grade + sqft_above + yr_built + yr_renovated + sqft_living15 + sqft_lot15

# ridge regression
library(glmnet)
library(mice)

x.1 = model.matrix(price ~ ., house.internal.1)[, -1]
y.1 = house.internal.1$price


ridge.1 <- glmnet(x.1, y.1, family = "gaussian", alpha = 0) 
plot(ridge.1, xvar="lambda")

r1cv <- cv.glmnet(x.1, y.1, family="gaussian", alpha = 0, nfolds = 10)
plot(r1cv)

rimin.1 <-glmnet(x.1, y.1, family = "gaussian", alpha = 0, lambda = r1cv$lambda.min)
coef(rimin.1)

# lasso regression
lasso.1 <- cv.glmnet(x.1, y.1, family="gaussian", alpha = 1, nfolds = 10)
plot(lasso.1)

lasmin.1 <- glmnet(x.1, y.1, family = "gaussian", alpha = 1, lambda = lasso.1$lambda.1se*0.5)
coef(lasmin.1)
```

#### random effect of census tracts


```{r}
house.map <- data.frame(house$lat,house$long,house$id,house$price)
head(house.map)

install.packages("ggmap")
library("ggmap")
qmplot(house.long, house.lat, data = house.map, maptype = "toner-background", color = house.price )

```
















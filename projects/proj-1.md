---
layout: post
title: 'Project One'
---
**Note: This post being actively developed as the project grows and as I learn more about MLOps and productionizing models**

*Unfortunately, after spending so much time searching for recent, free historical data regarding air pollution in Beijing, I came up with nothing. Various approaches were looked at, including web-scraping, long-term cron-jobs, modelling the pollution distributions and drawing from said distribution to return reasonable estimates – correlations and all…(this is actually a very interesting idea and I’m planning to write more about this later), and even __*gulp*__ *paying for data. None of these really panned out after deeper consideration for one reason or another, but in the spirit of this post, we’ll deploy the LSTM to Heroku and never leave the year 2017 (things were better then anyway).*

---

Having started (and continued) my career on the R&D side of things, I’ve seldom seen a project be carried through all the way to production and deployment. Trawling research papers, testing algorithms, prototyping systems, and all the data-sciencey-things you can think of are really cool! But applied research is a far cry from the polished, final products that we depend on. So, I found myself wanting to learn more about productionizing a product – the testing, maintenance, monitoring, and all the hard stuff that comes with it. This project is the culmination of that desire. 

This project focuses on developing a simple deep learning model, testing it, deploying to the web, and monitoring its performance. I’ll be using a Long Short-Term Memory (LSTM) to predict the fine inhalable particle matter (PM 2.5) in the atmosphere over the Dongsi neighborhood in Beijing, China. The data can be [viewed and downloaded here.](https://archive.ics.uci.edu/ml/datasets/Beijing+Multi-Site+Air-Quality+Data)

#### Data preparation

Begin by importing the relevant libraries:

```
import tensorflow as tf
from tensorflow import keras
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
```

And reading in the data for some EDA.
```
df = pd.read_csv('/content/Data/PRSA_Data_Dongsi_20130301-20170228.csv')
df.head()
```
{% include image.html image="projects/proj-1/df_head_start.png" %}

Why not make some cool plots to study correlations?

{% include image.html image="projects/proj-1/correlation_matrix.png" %}
{% include image.html image="projects/proj-1/scatter_plots.png" %}

It's not super suprising that the pollutants correlate with one another. And it makes sense that PM2.5 and PM10 correlate with one another. In general, it doesn't look like the weather features are particularly important when predicting PM2.5. Maybe the wind speed matters. There's a slight negative correlation there. Sounds reasonable to me.


```
df.drop(columns=['No','station'], inplace=True)
df['datetime'] = pd.to_datetime(df[['year','month','day','hour']])
df.drop(columns=['year'], inplace=True)
df.set_index('datetime', drop = False, inplace = True)
```

We’re only interested in one station – Dongsi, so we don’t need the "station" column. The "No." column also doesn’t offer any useful information, so we can safely remove these two columns. Furthermore, we can combine the "Year", "Month", "Day", and "Hour" columns to one datetime column and set it as our index. I won't drop the created datetime column just yet, though. We still need that for now. We’re also seeing that there is a separate column for wind direction and wind speed – but wind direction shouldn’t matter if wind speed is zero - that seems like a good candidate to practice a little feature engineering.

The datetime and wind columns are useful information, but datetime and wind direction are strings. Strings don’t make very good inputs to ML models, so we need to encode these columns in a way that the model knows how to deal with them correctly. A common solution to encoding string values is *one-hot encoding*. This involves creating a sparse vector and populating the appropriate indices with 1s. However, time is *periodic*, and if we consider the cardinal directions as they appear on a compass, the wind direction is cyclical. If we were to use one-hot encoding, the model would interpret 23:00h (11:00 PM) and 01:00h (01:00 AM) as being very far apart and dissimilar. However, we know that these times are much closer than they are farther apart. The same situation occurs for wind directions – North-Northwest is very close to North-Northeast, but a one-hot encoded model would interpret them as being quite different. In short **the model should understand that these values are cyclical and should not be interpreted as a straight line of values.**
The alternative to one-hot encoding is to represent the wind direction and time as *two* features – sine and cosine components – and transforming time and wind direction into vectors (using the wind speed as the magnitude of our wind vector). 

Starting with our datetime feature:

```
timestamp_s = df.index.map(pd.Timestamp.timestamp)    # converting datetime to a Unix-timestamp
seconds_in_day = 24*60*60

df['sin time'] = np.sin(2*np.pi*timestamp_s/seconds_in_day)   # sin(datetime_time) = (2*pi*datetime_time / datetime_period)
df['cos time'] = np.cos(2*np.pi*timestamp_s/seconds_in_day)   # cos(datetime_time) = (2*pi*datetime_time / datetime_period)

df.drop(['month','day','hour'], axis=1, inplace=True)
```

And next our new wind vector feature:

```
directions = df.pop('wd')
directions = [(2*np.pi*i / len(directions)) for i in range(len(directions))]    # converting to radians

wind_speed = df.pop('WSPM')
df['wx'] = wind_speed*np.cos(directions)    # x_component of wind vector
df['wy'] = wind_speed*np.sin(directions)    # y component of wind vector
```

{% include image.html image="projects/proj-1/wind_with_speed.png" %}{% include image.html image="projects/proj-1/wind_bivariate.png" %}


Nice, our model can now easily understand the periodicity of these inputs.
Looking further into our data, it looks like there are quite a few rows with missing data – 22% of rows actually. 
```
# check how many nans there are in each column
print(f'There are {df.isnull().sum().sum()} rows with missing values - {np.round(df.isnull().sum().sum() / df.shape[0] * 100)}% of rows are missing values')

>> There are 7600 rows with missing values - 22.0% of rows are missing values
```
One common way to deal with missing data is to simply drop the respective rows. We don’t particularly want to throw away 22% of data, so let’s look to impute the NaN values with either the columns mean value or median value. Which one we use depends on the distribution of the features - if they're mostly Gaussian, we'll use the mean. Otherwise, we'll use the median.

{% include image.html image="projects/proj-1/hists.png" %}

The histograms of the variables aren’t very normal – only the temperature is normal – so let’s impute the missing values with the columns median value. I’ll be using scikit-learn’s SimpleImputer class for this task.

```
from sklearn.impute import SimpleImputer
imp = SimpleImputer(missing_values=np.nan, strategy='median')

df = pd.DataFrame(imp.fit_transform(df), columns = df.columns)
df.isnull().sum()   # confirming there are no more missing values
df.head()
```

{% include image.html image="projects/proj-1/df_head_after_imputation.png" %}

Cool, no more missing values. But we aren’t done yet. See how our columns have very different ranges of values (we can use df.describe() to check out the stats on that, by the way) and are in different units? That’s going to be a problem if we don’t want (hint: we don’t) any column to dominate the others during training. To fix this, we can scale each feature to fit within the same range of values. We have a couple of options here, with the two most common likely to be a standardization approach and a min-max-scaling approach. Standardization will subtract the feature mean from the value and then divide by the feature’s standard deviation to restrict the features into the same range, while min-max-scaling will scale the features to be between 0 and 1. Remembering that our features aren’t exactly normally distributed, we will opt to go for the min-max-scaling strategy. We’ll use scikit-learn’s MinMaxScaler class for this task as well. However, we can’t scale our data just yet. Scaling now will lead to a form of train-test contamination by exposing information about our unseen testing data. We must first split our data into training and testing sets. This time, we can’t just take a random sample of our data and assign it to training and testing – remember, we’re working with time series data and the context matters! So, for this project we will take just the last 25% of all rows and assign them to be testing data. We have to be aware here though, that this could make our testing set unbalanced, as it’s unlikely that we can get an entire year and are instead looking only at the cold weather months. 

```
from sklearn.preprocessing import MinMaxScaler

train_split_ratio = 0.75
train_df = df[0:int(df.shape[0]*train_split_ratio)]
test_df = df[int(df.shape[0]*train_split_ratio):]

scaler = MinMaxScaler(feature_range=(0, 1))

# Scikit-learn has this habit of turning our DataFrame objects and transforming them into Numpy arrays 
# after the scaling process. I’d rather keep these as pandas DataFrames
train_df = pd.DataFrame.from_records(scaler.fit_transform(train_df),columns=train_df.columns)
test_df = pd.DataFrame.from_records(scaler.fit_transform(test_df),columns=df.columns)

# sanity check to make sure data transformations check out
print(df.shape)
print(train_df.shape)
print(test_df.shape)

>> (35064, 14)
>> (26298, 14)
>> (8766, 14)
```


Okay, that looks good. The shapes check out. Our data is sufficiently cleaned up, split into two distinct training and testing sets, and we can start to think about how to build our prediction model. 

#### Developing our model

Because we’re working with time series data, we want to capture as much of the context surrounding our prediction as possible. Therefore, we’ll be designing an LSTM using Tensorflow and Keras. LSTMs are a special class of Recurrent Neural Networks (RNN), which themselves are a special type of neural networks (duh.) that incorporate a feedback loop so that recent information can persist in the model. Unfortunately, RNNs fail to connect information together the farther back in time they go. They can really only be used if the gap from the earliest time-step and the predicted time-step is quite small. This is where the LSTM comes in. LSTMs are more capable of handling the vanishing gradient problem that plagues the RNN, which, in turn, allows them to learn over a longer history. It’s this ability to learn longer histories which makes LSTMs ideal for this sort of problem – predicting a value from a time series. This is a very abbreviated explanation, and [Colah’s blog](https://colah.github.io/posts/2015-08-Understanding-LSTMs/) is an excellent resource to learn more about LSTMs. 

By the way, if we were hoping to implement our prediction model on a small, memory-constrained device, it might be possible to use tree-based methods such as a random forest or gradient boosting algorithm and transform numerous samples (rows) into a single vector, similar to what we’ll be doing with our LSTM model, to predict air pollution. 

Before setting up the model, we need to decide how far back we want to look and how far into the future we want to predict. I’m interested in seeing how well we can predict one hour into the future, 12 hours into the future, and 1 day into the future. I think we should be able to do that with 3 days of data. The data is hourly, so we will need 24 rows * 3 days = 72 rows of lookback data, and a prediction 1, 12, and 24 hours into the future. 

```
num_days = 3    # past days on which to make a prediction
lookback = 24*num_days
num_future = 24   # how far ahead to jump in the future

# initiating training lists
x_train = []
y_train = []

# separating the inputs from the labels
for i in range(lookback, len(train_df) - num_future +1):
  x_train.append(train_df.iloc[i - lookback:i, 0:train_df.shape[1]])
  y_train.append(train_df.iloc[i:i + num_future, 0])
  
# converting lists to numpy arrays
x_train, y_train = np.array(x_train), np.array(y_train)
print(f'x_train shape = {x_train.shape}')
print(f'y_train shape = {y_train.shape}')

>> x_train shape = (26203, 72, 14)
>> y_train shape = (26203, 24)
```

Finally, we’re ready to set up our model. I’m stacking 2 LSTMs with a dropout layer between them to help counter overfitting. I’m using a tanh activation function because I found that using a typical RELU function caused my gradients to explode. I’m also defining a callback function to automatically save the model if the validation loss decreases.

```
from keras.callbacks import ModelCheckpoint
checkpoint_filepath = '/content/best_model.epoch{epoch:02d}-loss{val_loss:.4f}.hdf5'
model_checkpoint_callback = tf.keras.callbacks.ModelCheckpoint(
    filepath=checkpoint_filepath,
    save_weights_only=True,
    monitor='val_loss',
    mode='min',
    save_best_only=True)
```

```
from keras.models import Sequential
from keras.layers import Dense, Dropout
from keras.layers import LSTM

model = Sequential()
model.add(LSTM(128, activation='tanh', input_shape=(x_train.shape[1], x_train.shape[2]), return_sequences=True))
model.add(Dropout(0.3))
model.add(LSTM(32, activation='tanh', input_shape=(x_train.shape[1], x_train.shape[2]), return_sequences=False))

model.add(Dense(y_train.shape[1]))

model.compile(optimizer='adam', loss='mse')
model.summary()
```

{% include image.html image="projects/proj-1/model_summary.png" %}

And let's get fitting!

```
# fit the model
history = model.fit(x_train, y_train, epochs=10, batch_size=128, validation_split=0.2, verbose=1, callbacks=[model_checkpoint_callback])

plt.plot(history.history['loss'], label='Training loss')
plt.plot(history.history['val_loss'], label='Validation loss')
plt.legend()
```

{% include image.html image="projects/proj-1/24h_validation_training.png" %}

Okay! That’s not awful! We might be overfitting towards the end around epoch 8, but that could just be a result of the stochastic nature. Either way, validation accuracy is around ~98.7%, that might just work. Let's see how it works against the testing set - following the same routine as our training set:

```
# initializing the test lists
x_test = []
y_test = []

# separating the inputs from the labels
for i in range(lookback, len(test_df) - num_future +1):
  x_test.append(test_df.iloc[i - lookback:i, 0:test_df.shape[1]])
  y_test.append(test_df.iloc[i:i + num_future, 0])

# converting lists to numpy arrays
x_test, y_test = np.array(x_test), np.array(y_test)

# predicting our test set
prediction = model.predict(x_test)    # shape: (8671, 24)
```

By the way, this was not just a simple process where the first model I tried worked. The previous models suffered badly from overfitting initially. This final model is the end result of reducing model complexity, hidden units, increasing dropout rates, and increasing/decreasing the lookback attributes.

#### Considerations, or a healthy dose of skepticism

Should we expect this model to be as effective on live data? Ehh, maybe. There are a few issues with the data. Let's talk about them.

1. Live data distribution may be different from the training data (Covariate shift)

   It looks like the general pollution values have been decreasing since 2013. This is good! Great even! But maybe not so good for our predictions, as we're used to seeing relatively high values of pollutants. Our model might not be able to predict the decreasing levels, especially since we only have data through the first quarter of 2017. We'll need to keep an eye on the distributions.

2. Relatively small dataset

     Three years really isn't a long time. We can continually add to the dataset however, by storing our current predictions and current weather/pollutant measurements in a database, and retraining the model when accuracy falls below an acceptable threshold.

3. The data is specific to the Dongsi neighborhood in Beijing, China

     It would be interesting to see if the model can predict pollution in other parts of the world, but that likely won't be the case. We have to be mindful that the model may not predict well even in other neighborhoods of Beijing.

#### Deploying the model

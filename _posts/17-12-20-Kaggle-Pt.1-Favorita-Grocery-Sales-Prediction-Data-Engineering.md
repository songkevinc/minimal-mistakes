
# My first real Kaggle competition

I entered my first Kaggle competition about a month ago (Nov. 2017). I decided to enter the Corporacion Favorita grocery sales prediction competition. It seemed relatively simple, and I wanted to work on a project where I can try out either random forest regressor or xgboost regressor. I've never worked with either of them, but I've heard they do amazingly well and they're both widely used in many data science teams.

I'll write a 3 part article on my journey to finishing my first Kaggle competition.

1. First post will talk about how I clean my data (Data Engineering)
2. Second post will be about using XGBoost + parameter tuning
3. Third post will be Results and Discussion

### This post will be about data engieering!

First, we'll load up our favorite packages (numpy, pandas, matplotlib, seaborn, sklearn). I like looking at plots, so we'll use `%matplotlib inline` to view them in our notebook.


```python
%matplotlib inline
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.externals import joblib
from sklearn.ensemble import RandomForestRegressor

sns.set_style('whitegrid')

```

Here, we'll load up all of our data using pandas. You can see that we've got train and test sets, as well as some other supplementary data regarding all transactions, oilprice (the country economy depends a lot on oil), holiday+events (Ecuadorian), item and store information.


```python
train_db = pd.read_csv('train.csv', parse_dates=['date'])
test_db = pd.read_csv('test.csv', parse_dates=['date'])
transactions_db = pd.read_csv('transactions.csv', parse_dates=['date'])
oilprice_db = pd.read_csv('oil.csv', parse_dates=['date'])
holiday_db = pd.read_csv('holidays_events.csv', parse_dates=['date'])
items_db = pd.read_csv('items.csv')
stores_db = pd.read_csv('stores.csv')
```

    /home/kcsong/.conda/envs/my_root/lib/python2.7/site-packages/IPython/core/interactiveshell.py:2717: DtypeWarning: Columns (5) have mixed types. Specify dtype option on import or set low_memory=False.
      interactivity=interactivity, compiler=compiler, result=result)


# Check if the train and test data has any null values

Here we're just going to check which columns have null values. We'll check out what they are and figure out how to deal with them later. They might be as easy as throwing them out or perhaps interpolating to fill in new values.


```python
print "Training Data"
for col in train_db.columns:
    print col, train_db[col].isnull().any()
print "="*70    
print "Test Data"
for col in test_db.columns:
    print col, test_db[col].isnull().any()

```

     Training Data
    id False
    date False
    store_nbr False
    item_nbr False
    unit_sales False
    onpromotion True
    ======================================================================
    Test Data
    id False
    date False
    store_nbr False
    item_nbr False
    onpromotion False


Seems like Training data has missing  ***onpromotion*** variables. We'll later replace missing values with 2, True = 1 and False = 0.

# Check if supplementary data has any null values

Supplementary data = anything not train or test database


```python
supData = [transactions_db, oilprice_db, holiday_db, items_db, stores_db]
for sup in supData:
    print "="*70
    for col in sup.columns:
        print col, sup[col].isnull().any()
        
del supData, sup # deleting them to make sure I have enough memory
```

    ======================================================================
    date False
    store_nbr False
    transactions False
    ======================================================================
    date False
    dcoilwtico True
    ======================================================================
    date False
    type False
    locale False
    locale_name False
    description False
    transferred False
    ======================================================================
    item_nbr False
    family False
    class False
    perishable False
    ======================================================================
    store_nbr False
    city False
    state False
    type False
    cluster False


**Looks like we have some null values in oil prices!**

## Clean data + add 'dow' and 'doy'
Here, we're going to do some data cleaning. First, we're getting rid of all unit_sales that have negative values (I think they're returned purchases) because the problem prompt says we should ignore these negative sales. 

Secondly, I'm adding new features called `'dow'` and `'doy'` becuase I think day of the week and day of the year matters a lot in terms of when purchases are made. For example, I imagine more people go to grocery stores on the weekends rather than weekdays, which is why I'm adding a `'dow'` column. Furthermore, more people purchase items as the date gets closer to the holiday seasons, which is why I'm adding the `'doy'` column.


```python
# Data Cleaning
train_db.loc[(train_db.unit_sales<0),'unit_sales'] = 0 # Cleaning all negative values to be 0

# Add 'dow' and 'doy'
train_db['dow'] = train_db['date'].dt.dayofweek # adding day of week as a feature
train_db['doy'] = train_db['date'].dt.dayofyear # adding day of year as a feature

test_db['dow'] = test_db['date'].dt.dayofweek # adding day of week as a feature
test_db['doy'] = test_db['date'].dt.dayofyear # adding day of year as a feature
```

## Cleaning onpromotion column

Now, we're going to clean up the onpromotion column up by filling NaN, true, and false values with 2,1,0, respectively. I didn't want to give NaN 0 values because False will be 0. So, I just gave it a 2 entirely on its own class.


```python
train_db.loc[:,'onpromotion'].fillna(2, inplace=True) # Replace NaNs with 2
train_db.loc[:,'onpromotion'].replace(True, 1, inplace=True) # Replace Trues with 1
train_db.loc[:,'onpromotion'].replace(False, 0, inplace=True) # Replace Falses with 0

# Do the same for test set
test_db.loc[:,'onpromotion'].fillna(2, inplace=True) # Replace NaNs with 2
test_db.loc[:,'onpromotion'].replace(True, 1, inplace=True) # Replace Trues with 1
test_db.loc[:,'onpromotion'].replace(False, 0, inplace=True) # Replace Falses with 0
print 'done'
```

    done


## Finding 'dow' unit_sales and weekly averages


```python
# Grouping columns unit sales by
# item_nbr, store_nbr, dow (day of week)
# And storing means as dataframe
ma_dw = train_db[['item_nbr','store_nbr','dow','unit_sales']]\
             .groupby(['item_nbr','store_nbr','dow'])['unit_sales']\
             .mean().to_frame('madw')
ma_dw.reset_index(inplace=True)

# Storing weekly averages
ma_wk = ma_dw[['item_nbr', 'store_nbr','madw']]\
        .groupby(['item_nbr', 'store_nbr'])['madw']\
        .mean().to_frame('mawk')
ma_wk.reset_index(inplace=True)
print 'done'
```

    done


## Interpolating to fill in NaNs in the oilprice dataset


```python
# Oilprice dataset has many missing date values
# I'm just going to create a new database with every day registered
# Fill in the values by interpolating

index = pd.date_range(start='2013-01-01', end='2017-08-31')
new_oilprice_db = pd.DataFrame(index=index, columns=['date'] )
new_oilprice_db['date'] = index
new_oilprice_db.reset_index(inplace=True)
del new_oilprice_db['index']

# Linearly interpolating and manually filling in 3 points for linear interpolation
td = oilprice_db.date.diff() # time differece vector
interp = oilprice_db.dcoilwtico.shift(1) + ((oilprice_db.dcoilwtico.shift(-1) - oilprice_db.dcoilwtico.shift(1)))\
         * td / (td.shift(-1) + td)

oilprice_db['dcoilwtico'] = oilprice_db['dcoilwtico'].fillna(interp)

# Manually added the very first point 
oilprice_db['dcoilwtico'][0] = 93.14
oilprice_db['dcoilwtico'][1174] = 46.57
oilprice_db['dcoilwtico'][1175] = 46.75

# Merge the new oil price dataframe with the old dataframe
new_oilprice_db = pd.merge(new_oilprice_db, oilprice_db, on='date', how='left')

interp = new_oilprice_db.dcoilwtico.shift(2) +\
         ((new_oilprice_db.dcoilwtico.shift(-2) - new_oilprice_db.dcoilwtico.shift(2))) \
          / 2
new_oilprice_db['dcoilwtico'] = new_oilprice_db['dcoilwtico'].fillna(interp)

# Repeating interpolating twice because the first time only filled 1 of the back-to-back
# NaN values. If I repeate the interpolation twice, it should fill in all values

interp = new_oilprice_db.dcoilwtico.shift(1) +\
         ((new_oilprice_db.dcoilwtico.shift(-1) - new_oilprice_db.dcoilwtico.shift(1))) \
          / 2
new_oilprice_db['dcoilwtico'] = new_oilprice_db['dcoilwtico'].fillna(interp)
```

    /home/kcsong/.conda/envs/my_root/lib/python2.7/site-packages/ipykernel/__main__.py:18: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame
    
    See the caveats in the documentation: http://pandas.pydata.org/pandas-docs/stable/indexing.html#indexing-view-versus-copy
    /home/kcsong/.conda/envs/my_root/lib/python2.7/site-packages/ipykernel/__main__.py:19: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame
    
    See the caveats in the documentation: http://pandas.pydata.org/pandas-docs/stable/indexing.html#indexing-view-versus-copy
    /home/kcsong/.conda/envs/my_root/lib/python2.7/site-packages/ipykernel/__main__.py:20: SettingWithCopyWarning: 
    A value is trying to be set on a copy of a slice from a DataFrame
    
    See the caveats in the documentation: http://pandas.pydata.org/pandas-docs/stable/indexing.html#indexing-view-versus-copy


## Check if we have any null values in oilprice_db


```python
new_oilprice_db[new_oilprice_db['dcoilwtico'].isnull()]
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>date</th>
      <th>dcoilwtico</th>
    </tr>
  </thead>
  <tbody>
  </tbody>
</table>
</div>



## Now we'll merge all the supplementary data with train


```python
train = pd.merge(train_db, stores_db, on='store_nbr', how='left')
train = pd.merge(train, ma_dw, on=['item_nbr', 'store_nbr', 'dow'], how='left')
train = pd.merge(train, ma_wk, on=['item_nbr', 'store_nbr'], how='left')
train = pd.merge(train, items_db, on='item_nbr', how='left')
train = pd.merge(train, oilprice_db, on='date', how='left')
train = pd.merge(train, holiday_db, on='date', how='left')

train.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>date</th>
      <th>store_nbr</th>
      <th>item_nbr</th>
      <th>unit_sales</th>
      <th>onpromotion</th>
      <th>dow</th>
      <th>doy</th>
      <th>city</th>
      <th>state</th>
      <th>...</th>
      <th>mawk</th>
      <th>family</th>
      <th>class</th>
      <th>perishable</th>
      <th>dcoilwtico</th>
      <th>type_y</th>
      <th>locale</th>
      <th>locale_name</th>
      <th>description</th>
      <th>transferred</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>2013-01-01</td>
      <td>25</td>
      <td>103665</td>
      <td>7.0</td>
      <td>2</td>
      <td>1</td>
      <td>1</td>
      <td>Salinas</td>
      <td>Santa Elena</td>
      <td>...</td>
      <td>3.371367</td>
      <td>BREAD/BAKERY</td>
      <td>2712</td>
      <td>1</td>
      <td>93.14</td>
      <td>Holiday</td>
      <td>National</td>
      <td>Ecuador</td>
      <td>Primer dia del ano</td>
      <td>False</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>2013-01-01</td>
      <td>25</td>
      <td>105574</td>
      <td>1.0</td>
      <td>2</td>
      <td>1</td>
      <td>1</td>
      <td>Salinas</td>
      <td>Santa Elena</td>
      <td>...</td>
      <td>5.308930</td>
      <td>GROCERY I</td>
      <td>1045</td>
      <td>0</td>
      <td>93.14</td>
      <td>Holiday</td>
      <td>National</td>
      <td>Ecuador</td>
      <td>Primer dia del ano</td>
      <td>False</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>2013-01-01</td>
      <td>25</td>
      <td>105575</td>
      <td>2.0</td>
      <td>2</td>
      <td>1</td>
      <td>1</td>
      <td>Salinas</td>
      <td>Santa Elena</td>
      <td>...</td>
      <td>7.005743</td>
      <td>GROCERY I</td>
      <td>1045</td>
      <td>0</td>
      <td>93.14</td>
      <td>Holiday</td>
      <td>National</td>
      <td>Ecuador</td>
      <td>Primer dia del ano</td>
      <td>False</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3</td>
      <td>2013-01-01</td>
      <td>25</td>
      <td>108079</td>
      <td>1.0</td>
      <td>2</td>
      <td>1</td>
      <td>1</td>
      <td>Salinas</td>
      <td>Santa Elena</td>
      <td>...</td>
      <td>1.535041</td>
      <td>GROCERY I</td>
      <td>1030</td>
      <td>0</td>
      <td>93.14</td>
      <td>Holiday</td>
      <td>National</td>
      <td>Ecuador</td>
      <td>Primer dia del ano</td>
      <td>False</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>2013-01-01</td>
      <td>25</td>
      <td>108701</td>
      <td>1.0</td>
      <td>2</td>
      <td>1</td>
      <td>1</td>
      <td>Salinas</td>
      <td>Santa Elena</td>
      <td>...</td>
      <td>1.619115</td>
      <td>DELI</td>
      <td>2644</td>
      <td>1</td>
      <td>93.14</td>
      <td>Holiday</td>
      <td>National</td>
      <td>Ecuador</td>
      <td>Primer dia del ano</td>
      <td>False</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 23 columns</p>
</div>



## Check for null values


```python
for i in train.columns:
    print i, train[i].isnull().any()
```

    id False
    date False
    store_nbr False
    item_nbr False
    unit_sales False
    onpromotion False
    dow False
    doy False
    city False
    state False
    type_x False
    cluster False
    madw False
    mawk False
    family False
    class False
    perishable False
    dcoilwtico True
    type_y True
    locale True
    locale_name True
    description True
    transferred True


## Merge supplementary with test data


```python
test = pd.merge(test_db, stores_db, on='store_nbr', how='left')
test = pd.merge(test, ma_dw, on=['item_nbr', 'store_nbr', 'dow'], how='left')
test = pd.merge(test, ma_wk, on=['item_nbr', 'store_nbr'], how='left')
test = pd.merge(test, items_db, on='item_nbr', how='left')
test = pd.merge(test, holiday_db, on='date', how='left') # we have 2017 holiday info
test = pd.merge(test, oilprice_db, on='date', how='left')

test.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>date</th>
      <th>store_nbr</th>
      <th>item_nbr</th>
      <th>onpromotion</th>
      <th>dow</th>
      <th>doy</th>
      <th>city</th>
      <th>state</th>
      <th>type_x</th>
      <th>...</th>
      <th>mawk</th>
      <th>family</th>
      <th>class</th>
      <th>perishable</th>
      <th>type_y</th>
      <th>locale</th>
      <th>locale_name</th>
      <th>description</th>
      <th>transferred</th>
      <th>dcoilwtico</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>125497040</td>
      <td>2017-08-16</td>
      <td>1</td>
      <td>96995</td>
      <td>False</td>
      <td>2</td>
      <td>228</td>
      <td>Quito</td>
      <td>Pichincha</td>
      <td>D</td>
      <td>...</td>
      <td>1.557651</td>
      <td>GROCERY I</td>
      <td>1093</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>46.8</td>
    </tr>
    <tr>
      <th>1</th>
      <td>125497041</td>
      <td>2017-08-16</td>
      <td>1</td>
      <td>99197</td>
      <td>False</td>
      <td>2</td>
      <td>228</td>
      <td>Quito</td>
      <td>Pichincha</td>
      <td>D</td>
      <td>...</td>
      <td>2.771435</td>
      <td>GROCERY I</td>
      <td>1067</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>46.8</td>
    </tr>
    <tr>
      <th>2</th>
      <td>125497042</td>
      <td>2017-08-16</td>
      <td>1</td>
      <td>103501</td>
      <td>False</td>
      <td>2</td>
      <td>228</td>
      <td>Quito</td>
      <td>Pichincha</td>
      <td>D</td>
      <td>...</td>
      <td>NaN</td>
      <td>CLEANING</td>
      <td>3008</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>46.8</td>
    </tr>
    <tr>
      <th>3</th>
      <td>125497043</td>
      <td>2017-08-16</td>
      <td>1</td>
      <td>103520</td>
      <td>False</td>
      <td>2</td>
      <td>228</td>
      <td>Quito</td>
      <td>Pichincha</td>
      <td>D</td>
      <td>...</td>
      <td>2.813578</td>
      <td>GROCERY I</td>
      <td>1028</td>
      <td>0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>46.8</td>
    </tr>
    <tr>
      <th>4</th>
      <td>125497044</td>
      <td>2017-08-16</td>
      <td>1</td>
      <td>103665</td>
      <td>False</td>
      <td>2</td>
      <td>228</td>
      <td>Quito</td>
      <td>Pichincha</td>
      <td>D</td>
      <td>...</td>
      <td>3.485286</td>
      <td>BREAD/BAKERY</td>
      <td>2712</td>
      <td>1</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>46.8</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 22 columns</p>
</div>



## Deleting old data to free up memory space


```python
del ma_dw, ma_wk, holiday_db, oilprice_db, stores_db, sup, supData, test_db, items_db
```


    

    NameErrorTraceback (most recent call last)

    <ipython-input-55-b9a80cdd9471> in <module>()
    ----> 1 del ma_dw, ma_wk, holiday_db, oilprice_db, stores_db, sup, supData, test_db, items_db
    

    NameError: name 'sup' is not defined


## I'm going to divide up date into day, month and year and add them as features

I'm doing this because I think year, month and day separately give more information about unit sales than them combined, especially year and month. Based on some prior analysis, the number of transactions was growing yearly (pretty linear), and the number of transactions increase dramatically towards holiday seasons (months:11 and 12).


```python
train['month'] = train['date'].dt.month
train['year'] = train['date'].dt.year
train['day'] = train['date'].dt.day

```


```python
test['month'] = test['date'].dt.month
test['year'] = test['date'].dt.year
test['day'] = test['date'].dt.day

```

# Now we're going to select features

I'm going to select for features that I think will be important for determining unit_sales:

* store_nbr
* item_nbr
* cluster 
* dow
* doy
* madw
* perishable
* dcoilwtico
* onpromotion
* day
* month
* year

These will be stored as **`x_train`** and the unit_sales values will be stored as **`y_train`**


```python
#train.dropna(inplace=True)
x_train = train[['store_nbr', 'item_nbr', 'cluster', 'dow', 'doy', 'madw',
                 'perishable', 'dcoilwtico','onpromotion', 'day', 'month', 'year']]
y_train = train['unit_sales']

del train
```

## Saving the new dataframes as 'x_train.pkl' and 'y_train.pkl' 

We're going to save these new dataframes with **`joblib`**. I've tried cPickling before, but dumping and loading took 20x longer than using joblib. I'm not sure what the magic is behind joblib, but it'll do for this project.


```python
joblib.dump(x_train, 'x_train/x_train.pkl')
joblib.dump(y_train, 'y_train/y_train.pkl')
```

### *Footnote: For training and testing*

Before I finish off, I wanted to share some info about training and testing. Since this dataset is quite large ~12 million rows of unit_sales data, I've had trouble trying to fit beyond 10 trees in random forest model to all my data. So I'm just going to write up a quick footnote on how to split the dataset into train and test, and we'll use the actual test set as validation.


```python
# Splitting x_train into 
from sklearn.cross_validation import train_test_split

xRealTrain, xRealTest, yRealTrain, yRealTest = train_test_split(x_train, y_train, train_size=0.1)
# train_size = between 0 and 1.
```

# Summary

* Merged all data into 1 dataframe
* Added features such as 'dow', 'doy', 'madw', 'mawk', 'day', 'month', 'year'
* Used linear interpolation to fill in oil data
* footnote on how to split train into another set of train, test sets


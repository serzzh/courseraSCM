<a href="https://colab.research.google.com/github/serzzh/courseraSCM/blob/master/Coursera_Supply_Chain_Planning_Assignment.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>


```
import pandas as pd
import numpy as np
%pylab inline
plt.style.use("bmh")
plt.rcParams["figure.figsize"] = (24,6)
```

    Populating the interactive namespace from numpy and matplotlib


# Prepare data

### Downloading and reading data


```
url = "https://d3c33hcgiwev3.cloudfront.net/_dc27d596205cf8e528acb844f516e28b_Supply-Chain-Planning-Assignment-Data.csv?Expires=1574467200&Signature=VPZCN3Jm~KOMcOHCAcX7dZ11zHHzTQyqqJ4mzJvnD~h-3MwljnRTJ9uTA3Xpy0j8bekZ8EQmq8JOBZjwkxOCgYTqCXaWi3scAfJe8M64nwAe8EGhv9mGLgLXC7JS5DS9vKl3MtoBzfZmemHySZvxi~SznGZiT1mHeGL4Qfc~j50_&Key-Pair-Id=APKAJLTNE6QMUY6HBC5A"
df = pd.read_csv(url, header='infer')
df = df[df.columns[:5]].iloc[1:].apply(lambda x: x.str.replace(',',''))
df.replace(' -   ', 0.0, inplace=True)
df.columns = ['Date', 'Sales of A', 'Sales of B', 'Sales of C', 'Sales of D']
df['Date'] = df['Date'].astype('datetime64[ns]')
df = df.set_index('Date').apply(pd.to_numeric)
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Sales of A</th>
      <th>Sales of B</th>
      <th>Sales of C</th>
      <th>Sales of D</th>
    </tr>
    <tr>
      <th>Date</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2011-01-03</th>
      <td>210.0</td>
      <td>702.0</td>
      <td>18.0</td>
      <td>516.0</td>
    </tr>
    <tr>
      <th>2011-01-04</th>
      <td>10294.0</td>
      <td>402.0</td>
      <td>0.0</td>
      <td>324.0</td>
    </tr>
    <tr>
      <th>2011-01-05</th>
      <td>3395.0</td>
      <td>438.0</td>
      <td>0.0</td>
      <td>3618.0</td>
    </tr>
    <tr>
      <th>2011-01-06</th>
      <td>432.0</td>
      <td>19.0</td>
      <td>0.0</td>
      <td>414.0</td>
    </tr>
    <tr>
      <th>2011-01-07</th>
      <td>106.0</td>
      <td>0.0</td>
      <td>0.0</td>
      <td>72.0</td>
    </tr>
  </tbody>
</table>
</div>



### Visualising data


```
fig, ax = plt.subplots(2, 2)
ax[0, 0].plot(df['Sales of A'])
ax[0, 0].set_title('Sales of A')
ax[0, 1].plot(df['Sales of B'])
ax[0, 1].set_title('Sales of B')
ax[1, 0].plot(df['Sales of C'])
ax[1, 0].set_title('Sales of C')
ax[1, 1].plot(df['Sales of D'])
ax[1, 1].set_title('Sales of D')
```




    Text(0.5, 1.0, 'Sales of D')




![png](Coursera_Supply_Chain_Planning_Assignment_files/Coursera_Supply_Chain_Planning_Assignment_6_1.png)


Probably we should aggregate data by month too gain more insights


```
df = df.resample('M').sum()
df2 = df.copy()
```


```
fig, ax = plt.subplots(2, 2)
ax[0, 0].plot(df['Sales of A'])
ax[0, 0].set_title('Sales of A')
ax[0, 1].plot(df['Sales of B'])
ax[0, 1].set_title('Sales of B')
ax[1, 0].plot(df['Sales of C'])
ax[1, 0].set_title('Sales of C')
ax[1, 1].plot(df['Sales of D'])
ax[1, 1].set_title('Sales of D')
```




    Text(0.5, 1.0, 'Sales of D')




![png](Coursera_Supply_Chain_Planning_Assignment_files/Coursera_Supply_Chain_Planning_Assignment_9_1.png)


# Define appropriate forecasting method

Let's try to approximate our sales timeseries with following methods:

1.   Moving average
2.   Exponential Smoothing



## Moving average

Defining function for moving average


```
def mov_av(df, n=3):
  df = df.rolling(n).mean().shift(1).round(0)[n:].astype(int)
  df.columns = ['Mov.Av. of A', 'Mov.Av. of B', 'Mov.Av. of C', 'Mov.Av. of D']
  return df
```

Finding optimal value for number of periods using MSE as main metriics


```
def err(x,y, metrics = 'mse'):
  result = {
      'me': lambda x, y: sum(x-y)/len(x), # Mean Error
      'mape': lambda x, y: sum(abs(x-y)/x)/len(x), # Mean Absolute Percent Error
      'mse': lambda x, y: sum((x-y)*(x-y))/len(x) # Mean Square Error
      }[metrics](x,y)

  return result
```


```
def find_n(df, metrics = ['me', 'mape','mse']): #N optimization function
  
  out = []
  n = range(1, 7)

  for col in df.columns:
    x = df[col]
    errors = []
    for a in n:
        y = mov_av(x, a)
        errors.append({'col': col,
                      "N":a, 
                      "me": err(x[1:-1], y[:-1], metrics='me'),
                      "mape": err(x[1:-1], y[:-1], metrics='mape'),
                      "mse": err(x[1:-1], y[:-1], metrics='mse')
                      })
        
    er1 = pd.DataFrame.from_records(errors)
    for m in metrics:
      out.append(er1.iloc[er1[m].idxmin()])

  return pd.DataFrame.from_records(out)

```


```
er1 = find_n(df[['Sales of A', 'Sales of B', 'Sales of C', 'Sales of D']])
```


```
opt = er1.iloc[er1.groupby('col')['mse'].idxmin()] # finding alpha with minimum MSE for all products
opt
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>col</th>
      <th>N</th>
      <th>me</th>
      <th>mape</th>
      <th>mse</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2</th>
      <td>Sales of A</td>
      <td>3</td>
      <td>-3030.36</td>
      <td>0.099006</td>
      <td>38546864.60</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Sales of B</td>
      <td>5</td>
      <td>-59.68</td>
      <td>0.117352</td>
      <td>734718.48</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Sales of C</td>
      <td>3</td>
      <td>18.56</td>
      <td>0.117841</td>
      <td>1121.04</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Sales of D</td>
      <td>4</td>
      <td>-343.52</td>
      <td>0.184209</td>
      <td>59802987.04</td>
    </tr>
  </tbody>
</table>
</div>



Calculating Moving Average with N calculated above


```
df = df.join(
    mov_av(df, opt[opt.col=='Sales of A'].N.values[0])['Mov.Av. of A'], how='inner'
    ).join(
        mov_av(df, opt[opt.col=='Sales of B'].N.values[0])['Mov.Av. of B'], how='inner'
    ).join(
        mov_av(df, opt[opt.col=='Sales of C'].N.values[0])['Mov.Av. of C'], how='inner'
    ).join(
        mov_av(df, opt[opt.col=='Sales of D'].N.values[0])['Mov.Av. of D'], how='inner'
    )
```


```
fig, ax = plt.subplots(2, 2)
ax[0, 0].plot(df[['Sales of A', 'Mov.Av. of A']])
ax[0, 0].set_title('Sales of A, Mov.Av. of A')
ax[0, 1].plot(df[['Sales of B', 'Mov.Av. of B']])
ax[0, 1].set_title('Sales of B, Mov.Av. of B')
ax[1, 0].plot(df[['Sales of C', 'Mov.Av. of C']])
ax[1, 0].set_title('Sales of C, Mov.Av. of C')
ax[1, 1].plot(df[['Sales of D', 'Mov.Av. of D']])
ax[1, 1].set_title('Sales of D, Mov.Av. of D')
```




    Text(0.5, 1.0, 'Sales of D, Mov.Av. of D')




![png](Coursera_Supply_Chain_Planning_Assignment_files/Coursera_Supply_Chain_Planning_Assignment_22_1.png)


We can find that moving average with optimal N parameter overall cathes trend but lack reactions on the fluctuations of demand

## Exponential Smoothing

Finding optimal alpha for smoothing


```
def round_half_up(n, decimals=0): # Excel-like rounding function
    multiplier = 10 ** decimals
    return math.floor(n*multiplier + 0.5) / multiplier
```


```
def exp_av(df, cols, alpha): # Exponential Smoothing
  ea = []
  for i in range(len(df[cols])-1):
    fc = ea[i-1] if i >0 else df[cols][0]
    ea.append(round_half_up(df[cols][i]*alpha + fc*(1-alpha)))
  return ea
```


```
def find_alpha(df, metrics = ['me', 'mape','mse']): #Alpha optimization functio
  
  out = []
  alpha = [k * 0.1 for k in range(0, 11)]
  
  for col in df.columns:
    x = df[col]
    errors = []
    for a in alpha:
        y = exp_av(df, col, a)
        errors.append({'col': col,
                      "alpha":a, 
                      "me": err(x[1:-1], y[:-1], metrics='me'),
                      "mape": err(x[1:-1], y[:-1], metrics='mape'),
                      "mse": err(x[1:-1], y[:-1], metrics='mse')
                      })
        
    er1 = pd.DataFrame.from_records(errors)
    for m in metrics:
      out.append(er1.iloc[er1[m].idxmin()])

  return pd.DataFrame.from_records(out)
```


```
er2 = find_alpha(df2[['Sales of A', 'Sales of B', 'Sales of C', 'Sales of D']])
```


```
er2.iloc[er2.groupby('col')['mse'].idxmin()] # finding alpha with minimum MSE for all products
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>col</th>
      <th>alpha</th>
      <th>me</th>
      <th>mape</th>
      <th>mse</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2</th>
      <td>Sales of A</td>
      <td>0.4</td>
      <td>-2768.88</td>
      <td>0.113479</td>
      <td>56971264.56</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Sales of B</td>
      <td>0.1</td>
      <td>173.00</td>
      <td>0.145711</td>
      <td>1241678.52</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Sales of C</td>
      <td>0.7</td>
      <td>12.84</td>
      <td>0.116684</td>
      <td>1176.20</td>
    </tr>
    <tr>
      <th>11</th>
      <td>Sales of D</td>
      <td>0.6</td>
      <td>16.32</td>
      <td>0.195464</td>
      <td>67237450.00</td>
    </tr>
  </tbody>
</table>
</div>



We can find that the results of exponential smoothing is worse than moving average if using MSE metrics for all products except C, which results is very similar.


```
opt #Below the results of moving averave forecast which is better
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>col</th>
      <th>N</th>
      <th>me</th>
      <th>mape</th>
      <th>mse</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2</th>
      <td>Sales of A</td>
      <td>3</td>
      <td>-3030.36</td>
      <td>0.099006</td>
      <td>38546864.60</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Sales of B</td>
      <td>5</td>
      <td>-59.68</td>
      <td>0.117352</td>
      <td>734718.48</td>
    </tr>
    <tr>
      <th>8</th>
      <td>Sales of C</td>
      <td>3</td>
      <td>18.56</td>
      <td>0.117841</td>
      <td>1121.04</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Sales of D</td>
      <td>4</td>
      <td>-343.52</td>
      <td>0.184209</td>
      <td>59802987.04</td>
    </tr>
  </tbody>
</table>
</div>



So on current data is better to use moving average.


```

```

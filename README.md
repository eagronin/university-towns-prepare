# Data Preparation

## Overview
This section describes clearning and processing of the data collected for testing a hypothesis whether university towns have their mean housing prices less effected by recessions.  

The previous step, which describes the process of data acquisition, is described [here](https://eagronin.github.io/university-towns-acquire/).

The next step, which provides analysis of the data, is described [here](https://eagronin.github.io/university-towns-analyze/).

## Cleaning and Processing of the University Town Data
The list of university towns includes entries of both university towns and states in which these towns are located in a single column.  State names should be removed from that column and then added as a second column to a data frame with two columns corresponding to university towns and states they are in. The format of the DataFrame is: DataFrame( [ ["Michigan", "Ann Arbor"], ["Michigan", "Yipsilanti"] ], columns=["State", "RegionName"]  )

In addition to this converstion, certain characters and portions of the text need to be removed.  Specifically:

1. For "State", removing characters from "[" to the end.
2. For "RegionName", when applicable, removing every character from either "[" or " (" or ":" to the end.
3. Finally, it is important to note that certain rows in the data represent universities and not towns.  Those rows will be subsequently dropped when merging the university town data with housing price data. 

The following function performs the conversion described above, and then uses regular expressions to identify and remove the redundant text patterns.

```python
def get_list_of_university_towns():
    university_towns = load_university_town_data()
    university_towns['State'] = np.nan
    for town in university_towns['RegionName']:
        if re.search('(\[edit\])', town): 
            university_towns.State[university_towns.RegionName == town] = town
    university_towns['State'] = university_towns['State'].fillna(method = 'ffill')
    university_towns = university_towns[university_towns.RegionName != university_towns.State]
    university_towns = university_towns.reset_index(drop = True)
    for town in university_towns.RegionName:
        town_edited = re.sub('([\(\[:].*)', '', town).rstrip()
        #town_edited = re.sub('(\[.*)', '', town_edited).rstrip()
        university_towns.RegionName[university_towns.RegionName == town] = town_edited
    for state in university_towns.State:
        state_edited = re.sub('(\[.*)', '', state).rstrip()
        university_towns.State[university_towns.State == state] = state_edited
        names = ['State', 'RegionName']
        university_towns = university_towns[names]
    return university_towns
```

The first five rows of the resulting data frame are as follows:

```
     State    RegionName
0  Alabama        Auburn
1  Alabama      Florence
2  Alabama  Jacksonville
3  Alabama    Livingston
4  Alabama    Montevallo
...
```

## Processing of the GDP Data
Because subsequent analysis requires changes in GDP in order to determine the start, the end and the bottom of the recession, the following code creates leads and lags of GDP and then calculates changes in GDP:

```python
def gdp_lead_lag():
    gdp = load_gdp_data()
    gdp['Lagged GDP'] = np.nan
    gdp['Lagged GDP'][1:] = gdp.GDP[0:-1]
    gdp['Change in GDP'] = gdp.GDP - gdp['Lagged GDP']
    gdp['Lagged Change in GDP'] = np.nan
    gdp['Lagged Change in GDP'][1:] = gdp['Change in GDP'][0:-1]
    gdp['Lead Change in GDP'] = np.nan
    gdp['Lead Change in GDP'][0:-1] = gdp['Change in GDP'][1:]
    return gdp
```

The first five rows of the resulting data frame are as follows:

```
  Quarter      GDP  Lagged GDP  Change in GDP  Lagged Change in GDP  Lead Change in GDP
0  2000Q1  12359.1         NaN            NaN                   NaN               233.4
1  2000Q2  12592.5     12359.1          233.4                   NaN                15.2   
2  2000Q3  12607.7     12592.5           15.2                 233.4                71.6   
3  2000Q4  12679.3     12607.7           71.6                  15.2               -36.0   
4  2001Q1  12643.3     12679.3          -36.0                  71.6                67.0
...
```

# Processing of the Housing Data
Finally, the monthly housing price data needs to be converted to quarters before it is analyzed along with quarterly GDP figures.  The following function averages the monthly prices within each quarter.  The resulting dataframe has columns for 2000Q1 through 2016Q3, and a multi-index in the shape of ["State", "RegionName"].  Then the function below merges housing data with state names using state codes.  This step is necessary in order to subsequently merge the housing data with the university town data, which includes state names but does not include state codes:

```python
def convert_housing_data_to_quarters():
    #print(housing_data.shape)
    housing_data = load_housing_data()
    x = housing_data.drop_duplicates(subset = 'RegionID', keep = 'first')
    #print(x.shape)    
    housing_data = housing_data.set_index(['State', 'RegionName', 'RegionID'], drop = True) 
    housing_data.columns.name = 'Month'
    x = housing_data.stack()
    x.name = 'Value'
    x = x.reset_index()
    x.Month = pd.to_datetime(x.Month)
    x['Year'] = x.Month.dt.year
    x['Quarter'] = x.Month.dt.quarter
    x = x.groupby(['State', 'RegionName', 'RegionID', 'Year', 'Quarter'],).mean()['Value']
    x = x.reset_index()
    #print(x.dtypes)
    x['Date'] = x.Year.apply(str) + 'Q' + x.Quarter.apply(str)
    x = x[(x.Year >= 2000)]
    x = x.drop(['Year', 'Quarter'], axis = 1)
    x = x.set_index(['State', 'RegionName', 'RegionID', 'Date'])
    x = x['Value']
    x = x.unstack('Date')
    x = x.reset_index()
    x.rename(columns = {'State': 'Code'}, inplace = True)

    # Merge housing data with state names using state codes
    st_codes = pd.DataFrame.from_dict(states, orient = 'index')
    st_codes = st_codes.reset_index()
    st_codes.columns = ['Code', 'State']
    x = x.merge(st_codes, how = 'inner', on = 'Code')
    x = x.drop('Code', axis = 1)
    x = x.drop('RegionID', axis = 1)
    x = x.set_index(['State', 'RegionName'])

    return x
```

The first 10 rows and three columns of the resulting data frame are as follows:

```
                               2000Q1         2000Q2         2000Q3
State   RegionName                                                 
Alaska  Anchorage       174633.333333  175266.666667  179566.666667
        Fairbanks       163200.000000  165033.333333  169300.000000
        Homer                     NaN            NaN            NaN
Alabama Birmingham       54033.333333   54400.000000   54966.666667
        Brookwood        92566.666667   95100.000000   98866.666667
        Decatur                   NaN            NaN            NaN
        Duncanville     108100.000000  112033.333333  116133.333333
        Forestdale       88966.666667   89500.000000   89600.000000
        Grayson Valley   88100.000000   89366.666667   90033.333333
...
```

Next step:  [Analysis](https://eagronin.github.io/university-towns-analyze/).

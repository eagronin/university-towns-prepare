# Prepare data

Convert the university_towns.txt list to a DataFrame of towns and the states they are in. 
The format of the DataFrame is: DataFrame( [ ["Michigan", "Ann Arbor"], ["Michigan", "Yipsilanti"] ], 
columns=["State", "RegionName"]  )

The following cleaning has been done:

1. For "State", removing characters from "[" to the end.
2. For "RegionName", when applicable, removing every character from either "[" or " (" or ":" to the end.
3. Note: certain rows in the data represent universities and not towns. However, those rows will get dropped
   after merging with housing price data. 
   
```
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

# Create leads and lags of GDP for determining changes in GDP
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


# Convert the housing data to quarters and return it as mean values in a dataframe. 
# This dataframe has columns for 2000Q1 through 2016Q3, and a multi-index in the shape of ["State", "RegionName"].
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

    # Merge university towns and state codes
    st_codes = pd.DataFrame.from_dict(states, orient = 'index')
    st_codes = st_codes.reset_index()
    st_codes.columns = ['Code', 'State']
    x = x.merge(st_codes, how = 'inner', on = 'Code')
    x = x.drop('Code', axis = 1)
    x = x.drop('RegionID', axis = 1)
    x = x.set_index(['State', 'RegionName'])

    return x

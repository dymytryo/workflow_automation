<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#library-import">Library Import</a></li>
        <li><a href="#reading-from-google-sheet">Reading From Google Sheet</a></li>
        <li><a href="#writing-into-google-sheet">Writing Into Google Sheet</a></li>
        <li><a href="#creating-the-table-in-athena">Creating the Table in Athena</a></li>
        <li><a href="#writing-into-the-table-in-athena">Writing into the Table in Athena</a></li>
      </ul>
    </li>
  </ol>
</details>



<!-- ABOUT THE PROJECT -->
## About The Project

**Problem at hand**: the Operations team is working with merchants and payors daily resolving their issues, but most of this brand new effort is kept in Google Sheets. Indeed, it is difficult to maintain the work and have all the necessary information at hand.  

**Solution**: create an API call for the sheets and connect with the datalake to:
* remove the need for separate queries and analysis - read from the sheet, then fill in automatically all the necessary columns from the datalake; 
* ingest the results, wrangle in Python, write into a table into a datalake, vizualize and send out to the stakeholders.

![alt text](https://github.com/dymytryo/google-sheets-api/blob/e83b42ea77531e90e146c745c98ac4354f966f1d/images/automation_workflow.png?raw=true)

Steps:
- [x] Manual input by the operations team. 
- [x] Create a notebook in `Sagemaker` to ingest the data from the sheet.
- [x] Setup the connection to `Athena` using `PyAthena` connector and cursor to execute the requests.
- [x] Gather the data requested, wrangle, write into the sheet. 
- [x] Upload the skeleton version of the sheet as `.csv` into bucket in `S3` *(this is done only once)*.
- [x] Create a scratch table from the .csv *(this is done only once)*.
- [x] Convert the table in the datalake into the `Parqet` format to ensure its persistance *(this is done only once)*.
- [x] Write into the table the necessary data. 
- [x] Pull the data from newly created table and data lake to visualize in `Quicksight`. 
- [x] Set up the report to be automatically mailed to the management with KPIs.
- [x] Set up cron job with `CloudWatch` to automatically triger the execution of the notebook in `SageMaker` to repeat the flow daily.  

<!-- GETTING STARTED -->

## Getting Started
Since a lot of the data contains PII information about the customer base, I will be only sharing the generic query logic and libraries used to complete this project. 

### Library Import
To set-up the environment, we will need the following:
  ```python 
  # arrays and dataframes
  import pandas as pd
  import numpy as np
  
  # general functionality
  import datetime as dt
  import os
  import decimal 
  
  # preserve google api secret
  import pickle 
  
  # error logging 
  import logging
  import sys
  import traceback as tb

  # graphing 
  import matplotlib as mpl
  import matplotlib.pyplot as plt
  import seaborn as sns

  # expand the functionality of any function that takes only 2 arguments
  from functools import reduce
  
  # settings
  %matplotlib inline
  mpl.style.use('ggplot')
  pd.set_option('display.max_columns', None)
  pd.options.display.float_format = '{:.2f}'.format # remove scientific notation 
  
  # settings
  %matplotlib inline
  mpl.style.use('ggplot')
  pd.set_option('display.max_columns', None)
  pd.options.display.float_format = '{:.2f}'.format # remove scientific notation 
  
  # establish connection
  from pyathena import connect 
  from pyathena.pandas.cursor import PandasCursor
  
  # fetch the data from the query result
  cursor = connect(s3_staging_dir='s3://aws-athena/',
                 region_name='us-east-1',
                 work_group='Analyst',
                 cursor_class=PandasCursor).cursor()
  ```

### Reading from Google Sheet

This portion is actually wrapped in a function `read_sheet(url, range)`, but is broken down here for readability:

```python
# OAuth 2.0
SCOPES = ['https://www.googleapis.com/auth/spreadsheets']

# sheet of interest 
url = 'https://docs.google.com/spreadsheets/d/1cVkdf-aEQpvp8BfBd_BDKNDVD7rkOfhslI5JdQ/edit#gid=4569754212'

# extract id 
sheet_id = url.split('/')[5]
print(sheet_id)
```

Little reference cheat-sheet (from `Sheets API` documentation: 
* `Sheet1!A1:B2` refers to the first two cells in the top two rows of Sheet1.
* `Sheet1!A:A` refers to all the cells in the first column of Sheet1.
* `Sheet1!1:2` refers to all the cells in the first two rows of Sheet1.
* `Sheet1!A5:A` refers to all the cells of the first column of Sheet 1, from row 5 onward.
* `A1:B2` refers to the first two cells in the top two rows of the first visible sheet.
* `Sheet1` refers to all the cells in Sheet1.
* `'My Custom Sheet'!A:A` refers to all the cells in the first column of a sheet named "My Custom Sheet." Single quotes are required for sheet names with spaces, special characters, or an alphanumeric combination.
* `'My Custom Sheet'` refers to all the cells in 'My Custom Sheet'.

```python
# set the range
sheet_range = 'Operations Support Campaign' # give me the whole sheet 
sheet_name = 'Operations Support Campaign' # using this later to modify only certain columns

# credential
creds = None

# assing secret name 
secret = 'dmytro_secret.json'

# get the secret authenticated 
if os.path.exists('token.rw'):
    with open('token.rw', 'rb') as token:
        creds = pickle.load(token)
        
if not creds or not creds.valid:
    if creds and creds.expired and creds.refresh_token:
        creds.refresh(Request())
    else:
        flow = InstalledAppFlow.from_client_secrets_file(secret, SCOPES)
        creds = flow.run_local_server(port=0)
    # save the credentials for the next run
    with open('token.rw', 'wb') as token:
        pickle.dump(creds, token)
        
# read the values 
try:
    service = build('sheets', 'v4', credentials=creds)

    # api call
    sheet = service.spreadsheets()
    result = sheet.values().get(spreadsheetId=sheet_id,
                                range=sheet_range).execute()
    values = result.get('values', [])

    if not values:
        print('No data found.')

except HttpError as err:
    print(err)
```

Next, I would want to make sense of value outputs and work with it in the dataframe format: 

```python
# build the data frame 
df = pd.DataFrame(values)
df.head()

# assign header
header = df.iloc[0]
df = df[1:]
df.columns = header
df 
```

### Writing into Google Sheet
Next, we identify the information to fill in. The steps here are to create a tuple with values, insert it into query, and then work with the dataframe further:
```python
merchant_ids = tuple(df.loc[df['transactions last month'].isnull()]['merchant id']) 

merchant_transactions_query="""
 SELECT 
       ...
  WHERE 
       True 
       AND merchant_id IN {}
       ....
              
                            """.format(merchant_ids)

# get the dataframe with data of interest 
merchant_transactions_result = cursor.execute(merchant_transactions_query).as_pandas()

# close the cursor 
cursor.close()
```
Ultimately, we would arrive at the point of combining all the outputs:

```python
# create the list of all the dataframe that we worked with and now we would like to combine 
data_frames = [df, merchant_transactions_result, payor_details_result, customer_success_kpis_result, contacts_result, merchant_details_result]

# iterate to merge into one comprehensive dataframe 
agg = reduce(lambda  left, right: pd.merge(left,
                                     right,
                                     on=['record_id'],
                                     how='inner'),
                                     data_frames)
````
Now, since we have done all the wrangling, we would want to write back into the Sheet that Operations are working with. There is a custom function `sheet_write(range,values)`, but showing decomposed code snippets here for visibility:

```python
try:
    service = build('sheets', 'v4', credentials=creds)
    # insert the df to write 
    list = enrolled_bom.values.tolist()
    resource = {
      "majorDimension": "ROWS",
      "values": list
    }

    # clear cells
    range = sheet_name +'!K168:Q1368';
    service.spreadsheets().values().clear(spreadsheetId=sheet_id, range=range).execute()      
    
    # fill cells
    range = sheet_name +'!K168:Q1368';
    service.spreadsheets().values().append(
        spreadsheetId=sheet_id,
        range=range,
        body=resource,
        valueInputOption="USER_ENTERED"
    ).execute()
    
except HttpError as err:
    print(err)
```

### Creating the Table in Athena 
First, I will export my bare bones table from the notebook using `pd.to_csv('output.csv', index=False)`. Next, I will upload `.csv` file into the bucket. Now, I will go into Athena and run the following snippet.
I used `.dtypes`,'df.columns', and `for` loop to streamline the process of generating the following snippet:

Presto SQL(Athena): 
```sql
   CREATE EXTERNAL TABLE IF NOT EXISTS 
   AWSDataCatalog.ops_merchant_contact_demo(
                                              lastContactDate DATE,
                                              merchantId STRING,
                                              repId STRING,
                                              payorId STRING,
                                              contactType INT,
                                              callOutcome INT,
                                              emailContact INT,
                                              cityName STRING,
                                              state STRING,
                                              transactionVolumeLifetime REAL,
                                              ...
                                             )
-- specify the type of SerDe (Serializer/Deserializer) to define the table schema with 
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
-- specify the delimiters
WITH SERDEPROPERTIES (
                      'serialization.format' = ',',
                      'field.delim' = ',',
                      'collection.delim' = '|',
                      'mapkey.delim' = ':',
                      'escape.delim' = '\\'
                      ) 
-- give the bucket location 
LOCATION 's3://athena-datalake/dmytro/google_api_sheets_gdrf/'
TBLPROPERTIES (
                'classification' = 'csv',
                'has_encrypted_data'='false',
                "skip.header.line.count"="1", -- ignore header 
               );
```
  
Since the created table is dependent on the `.csv` file  uploaded and the idea is to have this table slowly ingesting more information, we would want to convert to parquet to lose the dependency on the S3 bucket.

Presto SQL(Athena): 
```sql
   CREATE TABLE IF NOT EXISTS AWSDataCatalog.ops_merchant_contact_info 
WITH (
      format = 'Parquet',
      parquet_compression = 'SNAPPY'
      )
AS SELECT 
    *
FROM 
    AWSDataCatalog.ops_merchant_contact_demo ;
```

### Writing Into the Table in Athena 
Finally, we can write from the dataframe into the table. I used the logging for the errors (not shown) should the connection fail and wrap the whole process into function `write_athena(table_name, df)`, showing simplified snippet here:

```python
# constant for the table 
ATHENA_TABLE = "AWSDataCatalog.ops_merchant_contact_info"

# execute a DML statement 
cursor.execute(
    f"""
INSERT INTO {ATHENA_TABLE}
VALUES 
      ('2022-08-09', 'mid045slpoiklamljg', ...),
      ...
      (...)
      """
)
# close the connection
cursor.close()
```
Now, all is done we can set up the dashboard in Quicksight and execute the Alert functionality within Quicksight. 

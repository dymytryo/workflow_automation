<!-- Improved compatibility of back to top link: See: https://github.com/othneildrew/Best-README-Template/pull/73 -->
<a name="readme-top"></a>
<!--
*** Thanks for checking out the Best-README-Template. If you have a suggestion
*** that would make this better, please fork the repo and create a pull request
*** or simply open an issue with the tag "enhancement".
*** Don't forget to give the project a star!
*** Thanks again! Now go create something AMAZING! :D
-->



<!-- PROJECT SHIELDS -->
<!--
*** I'm using markdown "reference style" links for readability.
*** Reference links are enclosed in brackets [ ] instead of parentheses ( ).
*** See the bottom of this document for the declaration of the reference variables
*** for contributors-url, forks-url, etc. This is an optional, concise syntax you may use.
*** https://www.markdownguide.org/basic-syntax/#reference-style-links
-->
[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![MIT License][license-shield]][license-url]
[![LinkedIn][linkedin-shield]][linkedin-url]



<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/othneildrew/Best-README-Template">
    <img src="images/logo.png" alt="Logo" width="80" height="80">
  </a>

  <h3 align="center">Best-README-Template</h3>

  <p align="center">
    An awesome README template to jumpstart your projects!
    <br />
    <a href="https://github.com/othneildrew/Best-README-Template"><strong>Explore the docs »</strong></a>
    <br />
    <br />
    <a href="https://github.com/othneildrew/Best-README-Template">View Demo</a>
    ·
    <a href="https://github.com/othneildrew/Best-README-Template/issues">Report Bug</a>
    ·
    <a href="https://github.com/othneildrew/Best-README-Template/issues">Request Feature</a>
  </p>
</div>



<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
      <ul>
        <li><a href="#built-with">Built With</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="pquisites">Prerequisites</a></li>
        <li><a href="#installation">Installation</a></li>
      </ul>
    </li>
    <li><a href="#usage">Usage</a></li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li><a href="#contributing">Contributing</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
    <li><a href="#acknowledgments">Acknowledgments</a></li>
  </ol>
</details>



<!-- ABOUT THE PROJECT -->
## About The Project

**Problem at hand**: the Operations team is working with merchants and payors daily resolving their issues, but most of this brand new effort is kept in Google Sheets. Indeed, it is difficult to maintain the work and have all the necessary information at hand. 
**Solution**: create an API call for the sheets and connect with the datalake to:
* remove the need for separate queries and analysis - read from the sheet, then fill in automatically all the necessary columns from the datalake 
* ingest the results, wrangle in Python, write into a table into a datalake, vizualize and send out to the stakeholders.

![alt text](https://github.com/dymytryo/google-sheets-api/blob/e83b42ea77531e90e146c745c98ac4354f966f1d/images/automation_workflow.png?raw=true)

Steps:
- [x] Manual input by the operations team. 
- [x] Create a notebook in Sagemaker to ingest the data from the sheet.
- [x] Setup the connection to Athena using PyAthena connector and cursor to execute the requests.
- [x] Gather the data requested, wrangle, write into the sheet. 
- [x] Upload the scheleton version of the sheet as .csv into bucket in S3 *(this is done only once)*.
- [x] Create a scratch table from the .csv (this is done only once).
- [x] Convert the table in the datalake into the Parqet format to ensure its persistance *(this is done only once)*.
- [x] Write into the table the necessary data. 
- [x] Pull the data from newly created table and data lake to visualize in Quicksight. 
- [x] Set up the report to be automatically mailed to the management with KPIs.
- [x] Set up cron job with CloudWatch to automatically triger the execution of the notebook in SageMaker to repeat the flow daily.  

<!-- GETTING STARTED -->

## Getting Started

We will cover some major snippets of the code. 

### Library Import
To set-up the environment for us to successfully execute this project, we will need the following:
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

### Reading Google Sheet

This portion is actually wrapped in a function `read_sheet(url, range)`, but is broken down here for readibility:

```python
# OAuth 2.0
SCOPES = ['https://www.googleapis.com/auth/spreadsheets']

# sheet of interest 
url = 'https://docs.google.com/spreadsheets/d/1cVkdf-aEQpvp8BfBd_BDKNDVD7rkOfhslI5JdQ/edit#gid=4569754212'

# extract id 
sheet_id = url.split('/')[5]
print(sheet_id)
```

Little reference cheat-sheet: 
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
Next, we identify the information that is need to fill in. The steps here is to create a tuple with values, insert it into query and then work with the dataframe further:
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
Now, since we have done all the wrangling, we would want to write back into the Sheet that Operations are working with. There is a custom function `sheet_write(range,values0`, but showing decomposed here for visibility:

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


_Below is an example of how you can instruct your audience on installing and setting up your app. This template doesn't rely on any external dependencies or services._

1. Get a free API Key at [https://example.com](https://example.com)
2. Clone the repo
3. 

```sql
   CREATE EXTERNAL TABLE IF NOT EXISTS 
   database.pre_payment_info (yr INT,
    quarter INT,
    month INT,
    dayofmonth INT,
    dayofweek INT,
    flightdate STRING,
    uniquecarrier STRING,
    airlineid INT,
    carrier STRING,
    tailnum STRING,
    flightnum STRING,
    originairportid INT,
    originairportseqid INT,
    origincitymarketid INT,
    origin STRING,
    origincityname STRING,
    originstate STRING,
    originstatefips STRING,
    originstatename STRING,
    originwac INT,
    destairportid INT,
    destairportseqid INT,
    destcitymarketid INT,
    dest STRING,
    destcityname STRING,
    deststate STRING,
    deststatefips STRING,
    deststatename STRING,
    destwac INT,
    crsdeptime STRING,
    deptime STRING,
    depdelay INT,
    depdelayminutes INT,
    depdel15 INT,
    departuredelaygroups INT,
    deptimeblk STRING,
    taxiout INT,
    wheelsoff STRING,
    wheelson STRING
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
LOCATION 's3://athena-datalake-us-east-1/dymytryo/google_api_sheets_gdrf/'
TBLPROPERTIES ('classification' = 'csv',
'has_encrypted_data'='false',
"skip.header.line.count"="1", -- ignore header 
);
```
  
3. 
Issue: created table is dependent on the csv file and the idea is having this table slowly ingesting more information
Solution: convert to parquet to lose the dependency on the S3 bucket

Athena SQL:
```sql
   CREATE TABLE IF NOT EXISTS database.payment_info 
WITH (
      format = 'Parquet',
      parquet_compression = 'SNAPPY')
AS SELECT *
FROM database.pre_payment_info ;
```


4. Enter your API in `config.js`
   ```js
   const API_KEY = 'ENTER YOUR API';
   ```

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- USAGE EXAMPLES -->
## Usage

Use this space to show useful examples of how a project can be used. Additional screenshots, code examples and demos work well in this space. You may also link to more resources.

_For more examples, please refer to the [Documentation](https://example.com)_

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- ROADMAP -->
## Roadmap

- [x] Add Changelog
- [x] Add back to top links
- [ ] Add Additional Templates w/ Examples
- [ ] Add "components" document to easily copy & paste sections of the readme
- [ ] Multi-language Support
    - [ ] Chinese
    - [ ] Spanish

See the [open issues](https://github.com/othneildrew/Best-README-Template/issues) for a full list of proposed features (and known issues).

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- CONTRIBUTING -->
## Contributing

Contributions are what make the open source community such an amazing place to learn, inspire, and create. Any contributions you make are **greatly appreciated**.

If you have a suggestion that would make this better, please fork the repo and create a pull request. You can also simply open an issue with the tag "enhancement".
Don't forget to give the project a star! Thanks again!

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- LICENSE -->
## License

Distributed under the MIT License. See `LICENSE.txt` for more information.

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- CONTACT -->
## Contact

Your Name - [@your_twitter](https://twitter.com/your_username) - email@example.com

Project Link: [https://github.com/your_username/repo_name](https://github.com/your_username/repo_name)

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- ACKNOWLEDGMENTS -->
## Acknowledgments

Use this space to list resources you find helpful and would like to give credit to. I've included a few of my favorites to kick things off!

* [Choose an Open Source License](https://choosealicense.com)
* [GitHub Emoji Cheat Sheet](https://www.webpagefx.com/tools/emoji-cheat-sheet)
* [Malven's Flexbox Cheatsheet](https://flexbox.malven.co/)
* [Malven's Grid Cheatsheet](https://grid.malven.co/)
* [Img Shields](https://shields.io)
* [GitHub Pages](https://pages.github.com)
* [Font Awesome](https://fontawesome.com)
* [React Icons](https://react-icons.github.io/react-icons/search)

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->
[contributors-shield]: https://img.shields.io/github/contributors/othneildrew/Best-README-Template.svg?style=for-the-badge
[contributors-url]: https://github.com/othneildrew/Best-README-Template/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/othneildrew/Best-README-Template.svg?style=for-the-badge
[forks-url]: https://github.com/othneildrew/Best-README-Template/network/members
[stars-shield]: https://img.shields.io/github/stars/othneildrew/Best-README-Template.svg?style=for-the-badge
[stars-url]: https://github.com/othneildrew/Best-README-Template/stargazers
[issues-shield]: https://img.shields.io/github/issues/othneildrew/Best-README-Template.svg?style=for-the-badge
[issues-url]: https://github.com/othneildrew/Best-README-Template/issues
[license-shield]: https://img.shields.io/github/license/othneildrew/Best-README-Template.svg?style=for-the-badge
[license-url]: https://github.com/othneildrew/Best-README-Template/blob/master/LICENSE.txt
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=for-the-badge&logo=linkedin&colorB=555
[linkedin-url]: https://linkedin.com/in/othneildrew
[product-screenshot]: images/screenshot.png
[Next.js]: https://img.shields.io/badge/next.js-000000?style=for-the-badge&logo=nextdotjs&logoColor=white
[Next-url]: https://nextjs.org/
[React.js]: https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB
[React-url]: https://reactjs.org/
[Vue.js]: https://img.shields.io/badge/Vue.js-35495E?style=for-the-badge&logo=vuedotjs&logoColor=4FC08D
[Vue-url]: https://vuejs.org/
[Angular.io]: https://img.shields.io/badge/Angular-DD0031?style=for-the-badge&logo=angular&logoColor=white
[Angular-url]: https://angular.io/
[Svelte.dev]: https://img.shields.io/badge/Svelte-4A4A55?style=for-the-badge&logo=svelte&logoColor=FF3E00
[Svelte-url]: https://svelte.dev/
[Laravel.com]: https://img.shields.io/badge/Laravel-FF2D20?style=for-the-badge&logo=laravel&logoColor=white
[Laravel-url]: https://laravel.com
[Bootstrap.com]: https://img.shields.io/badge/Bootstrap-563D7C?style=for-the-badge&logo=bootstrap&logoColor=white
[Bootstrap-url]: https://getbootstrap.com
[JQuery.com]: https://img.shields.io/badge/jQuery-0769AD?style=for-the-badge&logo=jquery&logoColor=white
[JQuery-url]: https://jquery.com 


# google-sheets-api
# Daux.io

[![Latest Version](https://img.shields.io/github/release/dauxio/daux.io.svg?style=flat-square)](https://github.com/dauxio/daux.io/releases)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](https://github.com/dauxio/daux.io/blob/master/LICENSE.md)
![GitHub Workflow Status](https://img.shields.io/github/workflow/status/dauxio/daux.io/CI?style=flat-square)
[![Coverage](https://sonarcloud.io/api/project_badges/measure?project=dauxio_daux.io&metric=coverage)](https://sonarcloud.io/dashboard?id=dauxio_daux.io)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=dauxio_daux.io&metric=alert_status)](https://sonarcloud.io/dashboard?id=dauxio_daux.io)
[![Total Downloads](https://img.shields.io/packagist/dt/daux/daux.io.svg?style=flat-square)](https://packagist.org/packages/daux/daux.io)

**Daux.io** is a documentation generator that uses a simple folder structure and Markdown files to create custom documentation on the fly. It helps you create great looking documentation in a developer friendly way.

## Features

-   100% Mobile Responsive
-   CommonMark compliant (a Markdown specification)
-   Supports Markdown tables
-   Auto created homepage/landing page
-   Auto Syntax Highlighting
-   Auto Generated Navigation
-   4 Built-In Themes or roll your own
-   Functional, Flat Design Style
-   Shareable/Linkable SEO Friendly URLs
-   Built On Bootstrap
-   No Build Step
-   Git/SVN Friendly
-   Supports Google Analytics and Piwik Analytics
-   Static Output Generation

## Demos

This is a list of sites using Daux.io:

-   With a custom theme:
    -   [Crafty](https://swissquote.github.io/crafty)
    -   [Pixolution flow](https://docs.pixolution.org) \* [Soisy](https://doc.soisy.it/)
    -   [Vulkan Tutorial](https://vulkan-tutorial.com) \* [3Q](https://docs.3q.video/)
    -   [The Advanced RSS Environment](https://thearsse.com/manual/)
-   With the default Theme
    -   [Daux.io](https://daux.io/)
        _ [DoctrineWatcher](https://dsentker.github.io/WatcherDocumentation/)
        _ [DrupalGap](http://docs.drupalgap.org/8/)
    -   [ICADMIN: An admin panel powered by CodeIgniter.](http://istocode.com/shared/ic-admin/)
    -   [Munee: Standalone PHP 5.3 Asset Optimisation & Manipulation](http://mun.ee)
    -   [Nuntius: A PHP framework for bots](https://roysegall.github.io/nuntius-bot/)

Do you use Daux.io? Send me a pull request or open an [issue](https://github.com/dauxio/daux.io/issues) and I will add you to the list.

## Install

### PHP and Composer

If you have PHP and Composer installed, you can install the dependency

```bash
composer global require daux/daux.io

# Next to your `docs` folder, run
daux generate
```

You can then use the `daux` command line to generate your documentation.

If the command isn't found, ensure your `$PATH` contains `~/.composer/vendor/bin`

### Docker

Or if you wish to use Docker, the start of the command will be :

```bash
docker run --rm -it -w /build -v "$PWD":/build -u "$(id -u):$(id -g)" daux/daux.io daux
```

## Run on a server

Download this repository as a zip, unpack, and put your documentation in the `docs` folder, you can then serve it with Apache or Nginx.

## `daux`

The command line tool has two commands: `generate` and `serve`, running Daux.io without an argument will automatically run the `generate` command.

You can run `daux --help` to get more details about each command.

## Folders

By default, the generator will look for folders in the `docs` folder. Add your folders inside the `docs` folder. This project contains some example folders and files to get you started.

You can nest folders any number of levels to get the exact structure you want. The folder structure will be converted to the nested navigation.

If you'd prefer to keep your docs somewhere else (like outside of the daux.io root directory) you can specify your docs path in the `global.json` file.

## Files

The generator will look for Markdown files (`*.md` and `*.markdown`) inside the `docs` folder and any of the subfolders within `docs`.

You must use underscores instead of spaces. Here are some example file names and what they will be converted to:

**Good:**

-   01_Getting_Started.md = Getting Started
-   API_Calls.md = API Calls
-   200_Something_Else-Cool.md = Something Else-Cool
-   \_5_Ways_to_Be_Happy.md = 5 Ways To Be Happy

**Bad:**

-   File Name With Space.md = FAIL

## Sorting

To sort your files and folders in a specific way, you can prefix them with a number and underscore, e.g. `/docs/01_Hello_World.md` and `/docs/05_Features.md` This will list _Hello World_ before _Features_, overriding the default alpha-numeric sorting. The numbers will be stripped out of the navigation and urls. For the file `6 Ways to Get Rich`, you can use `/docs/_6_Ways_to_Get_Rich.md`

## Landing page

If you want to create a beautiful landing page for your project, simply create a `index.md` file in the root of the `/docs` folder. This file will then be used to create a landing page. You can also add a tagline and image to this page using the config file like this:

```json
{
    "title": "Daux.io",
    "tagline": "The Easiest Way To Document Your Project",
    "image": "app.png"
}
```

Note: The image can be a local or remote image. Use the convention `<base_url>` to refer to the root directory of the Daux instance.

## Section landing page

If you are interested in having a landing page for a subsection of your docs, all you need to do is add an `index.md` file to the folder. For example, `/docs/01_Examples` has a landing page for that section since there exists a `/docs/01_Examples/index.md` file. If you wish to have an index page for a section without a landing page format, use the name `_index.md`

## Configuration

To customize the look and feel of your documentation, you can create a `config.json` file in the of the `/docs` folder.
The `config.json` file is a simple JSON object that you can use to change some of the basic settings of the documentation.

### Title

Change the title bar in the docs

```json
{
    "title": "Daux.io"
}
```

### Themes

We have 4 built-in Bootstrap themes. To use one of the themes, just set the `theme` option to one of the following:

-   daux-blue
-   daux-green
-   daux-navy
-   daux-red

```json
{
    "html": { "theme": "daux-green" }
}
```

### More options

Many other options are available:

-   [Global options](http://daux.io/Configuration/index)
-   [HTML Options](http://daux.io/Configuration/Html_export)
-   [Confluence options](http://daux.io/Configuration/Confluence_upload)

## Running Remotely

Copy the files from the repo to a web server that can run PHP 8.1.0 or newer.

## Running Locally

There are several ways to run the docs locally.
The recommended way is to run `daux serve` which will execute PHP's embedded server.

By default the server will run at: <a href="http://localhost:8085" target="_blank">http://localhost:8085</a>

This is really only intended be used when you are writing/updating a ton of docs and want to preview the changes locally.

## Generating a set of static files

These can be uploaded to a static site hosting service such as pages.github.com

Generating a complete set of pages, with navigation

```bash
daux --source=docs --destination=static
```

## Running on IIS

If you have set up a local or remote IIS web site, you may need a `web.config` with:

-   A rewrite configuration, for handling clean urls.
-   A mime type handler for less files, if using a custom theme.

### Clean URLs

The `web.config` needs an entry for `<rewrite>` under `<system.webServer>`:

```xml
<configuration>
    <system.webServer>
        <rewrite>
            <rules>
                <rule name="Main Rule" stopProcessing="true">
                    <match url=".*" />
                    <conditions logicalGrouping="MatchAll">
                        <add
                            input="{REQUEST_FILENAME}"
                            matchType="IsFile"
                            negate="true"
                        />
                        <add
                            input="{REQUEST_FILENAME}"
                            matchType="IsDirectory"
                            negate="true"
                        />
                    </conditions>
                    <action
                        type="Rewrite"
                        url="index.php"
                        appendQueryString="false"
                    />
                </rule>
            </rules>
        </rewrite>
    </system.webServer>
</configuration>
```

To use clean URLs on IIS 6, you will need to use a custom URL rewrite module, such as [URL Rewriter](http://urlrewriter.net/).

## PHP Requirements

Daux.io is compatible with the [officially supported](https://www.php.net/supported-versions.php) PHP versions; 8.1.0 and up.

### Extensions

Daux.io needs the following PHP extensions to work : `php-mbstring` and `php-xml`.

If you use non-english characters in your page names, it is recommended to install the `php-intl` extension as well.

## Support

If you need help using Daux.io, or have found a bug, please create an issue on the <a href="https://github.com/dauxio/daux.io/issues" target="_blank">GitHub repo</a>.

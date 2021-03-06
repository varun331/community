# Community
This is a place for gathering comments and other social media content across major platforms.

## Step 1. Scoping the project and gathering data

### Identifying and gathering data
The data gathered for this project consist of (1) Reddit articles and comments; and (2) YouTube video statistics.

For Reddit, the data is gathered via the PRAW API. The `reddit_lambda.py` file at the top-level of the repository connects to submissions for particular subreddits and gathers submission information alongwith top-level comments. This file was written with the intent of converting it to a lambda function for scheduled running on AWS.

By hooking the `lambda_handler` function within `reddit_lambda.py` I was able to gather over 1.7M Reddit comments along with thousands of observations of posts over the course of one weekend. This data was stored in JSON files on S3, which is described in section two (2). Proof that this much data was in fact collected is shown via querying the final Redshift tables. See figure below.

![Data model](https://github.com/wsankey/community/blob/master/capstone_datastats.png)

YouTube data collection was also done via API. The top-level `youtube_lambda.py` has a command-line interface that draws out YouTube video metadata for an input topic and saves that information to S3 via CSV.

### Use cases
There are numerous use cases for this data.

The spirit of the community repository is to identify what particular communities of individuals are talking about. So under the "gaming" community broadly, what are the discussion points for a particular point in time? What is the community excited about, and what are some of the challenges? These questions can be answered in part by collecting data on community discussions.

Towards this end these data are gathered and pipelined into a postgres database on AWS Redshift to become analysis ready.


## Step 2. Exploring and assessing the data
For both the Reddit and YouTube data the intent is to save as raw as possible JSON files into S3 and then clean/manipulate that data into RedShift via CSV files.

The raw data is structured such that the article id is the key to the dictionary of information. Here is one example of one Reddit article in one S3 .json log file.

`
{"dden4u": {"title": "The Indie Industry: Defying the Odds", "score": 57,
`
`"url": "https://www.youtube.com/watch?v=VwV_59LBF4E", "name": "t3_dden4u", "author": "bisquick_quick",
`
`
"is_video": false, "over_18": false, "selftext": "", "shortlink": "https://redd.it/dden4u", "subreddit_type": "public", "subreddit_subscribers": 41733, "thumbnail": "https://b.thumbs.redditmedia.com/K3Jx0cc0NY1gpNgyB3mrF_U-WU8FpZIVXlzzQeDXnAU.jpg", "ups": 57, "created_utc": "2019-10-04T22:29:50", "archived": "2019-10-05T12:26:36.970597", "subreddit": "indiegames"
`

This is a typical example of the articles data being collected from Reddit. The data is of various types, of various lengths, and may contain escape characters (e.g. \n) incompatible with Redshift. Data may be missing (the example is missing `selftext`) and datetimes will need to be converted to dates. The Reddit comment and YouTube data have similar issues as the ones just described.

The steps taken to clean this data was (1) identify only the fields we want to bring into Redshift and only keep that information; (2) remove all illegal characters for Redshift; and (3) save the transformed data as a CSV.

## Step 3. The Data Model
The data model consists of two main tables: Reddit top comments and general YouTube videos. By combining two distinct tables, Reddit articles and comments, we can find top voted comments and have a reference to the article they were submitted on. The article id that is saved for both the article and the comment is what links the articles and comments together so we can make the top comment table.

Meanwhile the YouTube general table cannot be merged with these Reddit data sources at this time and must sit distinct.

![Data model](https://github.com/wsankey/community/blob/master/capstone_datamodel.png)

To get the data into this model we need to take the raw json logs and manipulate them. We first keep only the data we want from the logs, then we remove any potentially damanging character, such as an escape delimiter. Then we save that entire log file as a CSV. These data cleaning steps are completed in `etl/logs_to_csv.py`.

Opening a terminal and running `pyhton etl/logs_to_csv.py` will manipulate all logs files living with the S3 bucket and, after cleaning them via the aforementioned method, save them to another S3 bucket of S3s. The program is smart enough to pick up where the user may have left off.

Once the CSV files are constructed we can move them via RedShift COPY to the cluster. The articles and comments tables are created first, then the `top_comments` table is created afterwards using fields from those tables. The YouTube general table is made last.

## Step 4. ETL and modeling the data
The code for each step of this process, after actually gathering the data, are:
1. Run `etl/create_tables.py` to create the tables in the Redshift cluster;
2. Run `etl/etl.py` that will take the processed CSVs and insert them into the cluster.

The following data dictionary shows the columns, data types, and tables on the Redshift cluster.

![Data model](https://github.com/wsankey/community/blob/master/capstone_datadictionary.png)

To ensure the integrity of our data pipeline we will need to write tests. We can write unit tests around cleaning the data specifically for CSV creation. It is more difficult to unittest the correct variable type on the clsuter. However, we can go on the querying portion of Redshift and see that all of the variables for our tables match our description of those variables in the code. Final quality checks are done by spot-checking some of the JSON logs against the Redshift data records.

## Step 5. What's the goal again? And other considerations
Having a database of comments for various themes allows us to see what a community is thinking down at the user level and over time. One potential use-case is to provide insight into how a community is responding to a particular issue or set of issues across any give time. We start to achieve these goals with this repository.

This architecture can easily support large amounts of new data, scheduling tasks to run regularly, and involving other team members.

### Large amounts of new data
Since this pipeline is built on top of AWS it can scale. Adding nodes to the RedShift cluster is done via the AWS interface and S3 can automatically scale with additional data moving into it.

### Running regularly
The AWS lambdas that are built off of this repository can be scheduled to run at any hour of the day. Scheduling the Redshift COPY can be achieved via integrating this pipeline with Airflow.

### Team member access
Add team members to this project via IAM roles in AWS.

# spotify-api-etl
This is a project that involves different technologies that extract data from a spotify API, transform it and load it to snowflake.

The Technologies involved in this project includes:
- Python (AWS Lambda function): AWS Lambda is a serverless computing service that allows you to run code without manging any server.
- AWS Glue (Spark): It is a tool to transform data using Apache Spark
- S3 (Storage): It is a highly scalable object storage that can store files in the form of Objects. We could also define it as a Data Lake in it's raw form.
- SQL
- Snowflake (Snowpipe): Snowflake is a cloud serverless data warehouse which is very well used for stored the transformed data and do some aggregations from there.
- Airflow (Orchestration)

  Let's take a look at a Diagram using these different tools and how they connect:
  ![Architecture Diagram](https://github.com/alblcq/spotify-api-etl/blob/main/spotify_architecture.png)

Making this program run:

API: We will need to go to the Spotify API and create an account, once we have our account we will need the client id and client_secret stored somehwere

The first step is to get into AWS and create a IAM user and assign the correct permissions to that user (AmazonS3FullAccess, AWSLambda_FullAccess, AWSGlueServiceRole) once the user is created we will need 2 things here,
Store the AWS Access Key ID and generate an AWS Secret Access Key we will store somewhere

0 - Create a S3 Bucket with the name and you might need to replace this name in the different scripts or create. Inside of the S3 Bucket we will create a Folder Structure which will look something like
raw_data, transformed_data, and inside of raw_data we will have 2 other folders, To_processed and processed, inside of transformed_data we will have album_data, songs_data, artist_data folders.

1 - Go to Lambda Functions in AWS and create a file using the code we have in the spotipy-api-etl/scripts/spotify_api_data_extract.py

2- then you can also download the code from glue_job/spotify_transformation_job.py and create a script by uploading that code.

3- We will be running this by creating an orchestration job, In your airflow installed by either Docker or local. Go to your DAG Folder and paste in there the script from Dag/spotipy_transformation_job.py

4- We will create a database in Snowflake and we will implement a Snowpipe that will capture the changes in our S3 Bucket and once there is a new change then we will grab the data from the S3 Bucket and add the data to the tables created in Snowflake.

SNOWFLAKE

1- We will first create a database named SPOTIFYDB: Create database spotifydb;

use SPOTIFYDB;

after that we will need to create our tables:

CREATE TABLE tblAlbum (
    album_id STRING,
    album_name STRING,
    album_release_date DATE,
    album_total_tracks INT,
    album_url STRING
)

CREATE TABLE tblArtist (
    artist_id STRING,
    artist_name STRING,
    external_url STRING
)

CREATE TABLE tblSongs (
    song_id STRING,
    song_name STRING,
    song_duration INT,
    song_url STRING,
    song_popularity INT,
    song_added TIMESTAMP_TZ
)

Once this is done we can create our snowpipe and storage integration as well as file formats and stages:

CREATE OR REPLACE storage integration s3_int
    TYPE = EXTERNAL_STAGE
    STORAGE_PROVIDER = S3
    ENABLED = TRUE
    STORAGE_AWS_ROLE_ARN = "arn:aws:iam::471583395564:role/snowflake-s3-connection"
    STORAGE_ALLOWED_LOCATIONS = ('s3://alblcq-bucket-spotify')
    COMMENT = 'S3 Connection'


DESC integration s3_int;

CREATE OR REPLACE SCHEMA file_formats

CREATE OR REPLACE file format file_formats.csv_fileformat
    type = csv
    field_delimiter = ','
    skip_header = 1
    null_if = ('NULL', 'null')
    empty_field_as_null = TRUE;

create schema external_stages

CREATE OR REPLACE stage external_stages.spotify_album_stage
    URL = 's3://alblcq-bucket-spotify/transformed_data/'
    STORAGE_INTEGRATION = s3_int
    FILE_FORMAT = file_formats.csv_fileformat

CREATE OR REPLACE stage spotify_artist_stage
    URL = 's3://alblcq-bucket-spotify/transformed_data/artist_data/'
    STORAGE_INTEGRATION = s3_int
    FILE_FORMAT = csv_fileformat

CREATE OR REPLACE stage spotify_songs_stage
    URL = 's3://alblcq-bucket-spotify/transformed_data/songs_data/'
    STORAGE_INTEGRATION = s3_int
    FILE_FORMAT = csv_fileformat

LIST @external_stages.spotify_album_stage/album_data;
LIST @spotify_artist_stage;
LIST @spotify_songs_stage;

CREATE OR REPLACE SCHEMA pipes

CREATE OR REPLACE pipe pipes.album_pipe
auto_ingest = TRUE
AS
COPY INTO spotify_songs.TBLALBUM
FROM @external_stages.spotify_album_stage/album_data;

CREATE OR REPLACE pipe pipes.artist_pipe
auto_ingest = TRUE
AS
COPY INTO spotify_songs.TBLARTIST
FROM @spotify_songs.spotify_artist_stage;

CREATE OR REPLACE pipe pipes.songs_pipe
auto_ingest = TRUE
AS
COPY INTO spotify_songs.TBLSONGS
FROM @spotify_songs.spotify_songs_stage;

DESC pipe pipes.album_pipe;
DESC pipe pipes.artist_pipe;
DESC pipe pipes.songs_pipe;

To finish setting up the snowpipe we will need to go to our bucket in Amazon S3

Click on Properties -> scroll down to Event Notifications and Create a New event named snowpipe_album this will need to be done for each one including snowpipe_songs and snowpipe_artist

To create it we will need to click on All Object create events in the event types
and then go to the bottom and select SQS queue in Destination

and then in Specify SQS queue select enter SQS queue ARN
and paste the ARN SQS that you get from Describing the pipes.

And now you should be able to run from Airflow this project automatically daily, It will extract the data and save it in S3, then will take the data from there and transform it with GLUE and then storing that data in S3 again and it will also take this data and read it in Snowflake from the S3 and add this data to the Data Warehouse so we can then use it for analytics purposes using tools like Power BI





  


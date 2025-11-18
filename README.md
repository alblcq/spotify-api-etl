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

  


# Project 2: Tracking User Activity

## Alissa Stover

# Project Summary

*Goal of this project:* In this project, you work at an ed tech firm. You've created a service that
delivers assessments, and now lots of different customers (e.g., Pearson) want
to publish their assessments on it. You need to get ready for data scientists
who work for these customers to run queries on the data. 

*Outcome of this project:* In this project, I used Docker, Kafka, Spark, and Hadoop to get the assessment data described above ready to be used by data scientists. 

# Tasks

I prepared the infrastructure to land the data in the form and structure it needs to be to be queried.  To do this, I needed to: 

- Publish and consume messages with Kafka
- Use Spark to transform the messages. 
- Use Spark to transform the messages so that I could land them in HDFS

I have included my `docker-compose.yml` used for spinning the pipeline. 

I also included the history of my console. This history file in in this repo under the name `alisto92-history.txt` and was generated using the following command:
```
history > alisto92-history.txt
```
The history file has not been altered -- I submitted it as is. I selected the lines relevant to how I spun the pipeline in the `commands.md` file. Those relevant to my report are embedded in `report.md`. 

We could either run Spark through the command line or use a notebook. I used Spark through the command line and have included an annotated markdown file (`report.md`) to present my results, tools used, and explanations of the commands I used.

I have included my Spark history in the file `alisto92-sparkhistory.txt`. To get the history from Spark, after quitting Spark I ran:

```
docker-compose exec spark cat /root/.python_history > alisto92-sparkhistory.txt
```

At the end of this pipeline, I am able to query the data. 

In order to show the data scientists at these other companies the kinds of data that they will have access to, I answered 3 basic business questions (at the end of `report.md` that I believe they might need to answer about these data. I also gave instructions for how they might run other types of queries and also save the results of these to HDFS.

## What is provided in this repo

- My history file, `alisto92-history.txt`

- A report either as a markdown file, `report.md`
  The report describes my queries and spark SQL to answer business questions. 
  I describe my assumptions, my thinking, what the parts of the
  pipeline do. What was given? What did I set up myself? I discuss the data.
  What issues did I find with the data? Did I find ways to solve it? How?
  If not, I describe what the problem is.

- Other files needed for the project, e.g., docker-compose.yml, scripts used, etc

  * `commands.md` provides the commands needed to: prepare and run the Docker containers; publish & consume the messages in  Kafka; and use Spark to transform the messages as well as to transform them to land them into HDFS.
  
  * `alisto92-sparkhistory.txt` provides my history from Spark.


## Data

To get the data, I ran 
```
curl -L -o assessment-attempts-20180128-121051-nested.json https://goo.gl/ME6hjp`
```

I was given this note on the data: This dataset is much more complicated than the one we'll cover in the live sessions. It is a nested JSON file, where you need to unwrap it carefully to understand what's really being displayed. There are many fields that are not that important for your analysis, so don't be afraid of not using them. The main problem will be the multiple questions field. Think about what the  problem is here and give your thoughts. We recommend for you to read schema implementation in Spark [Here is the documenation from Apache](https://spark.apache.org/docs/2.3.0/sql-programming-guide.html).
It is NOT necessary to get to that data.


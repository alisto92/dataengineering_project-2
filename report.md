## Project 2: Report 

### Data pipeline

Please see the `command.md` file contained in this repository for a set of commands used for processing these data so that they could be queried using the code below. 

On a high level, the following technologies were used to get the data to this point:

* Docker -- forms containers the provide an environment to work in 
* Kafka -- ingests the data in a topic
    * What's the name of this Kafka topic? **assessments**
    * I chose this name under the advice of our Data Engineering instructors, as it is an informative name about what is contained the data (information about assessments)
* Spark -- reads the data from Kafka in JSON and extracts it
* Hadoop -- stores the data in a format to be used by other data scientists to answer business questions that we can answer with the assessment data
 
### Structure of assessments json file

We were given the following information about these data: 
*The assessments json file is nested in a complex multi-valued way.  It has nested dictionaries that nest lists that nest dictionaries that nest lists that nest dictionaries.*

Based on the Jupyter Notebook provided that displayed the data structure, we can see that assessments contains the following. 

I have made some guesses about what each field indicates as we were not provided a data dictionary.

* base_exam_id: an exam ID
* certification: whether certification was achieved (or, perhaps whether certification was attempted)
* exam_name: exam name 
* keen_created_at: timestamp for when "keen" was created; I do not know what "keen" refers to.
* keen_id: "keen" ID; perhaps the profile created by assessment-takers to record an assessment attempt
* keen_timestamp: a second timestamp for "keen"; I do not know what action this timestamp refers to.
* max_attempts: number of maximum attempts allowed 
* sequences: a nested structure that contains the following sub-values:
  * attempt: the number of the attempt
  * counts: another nested structure that contains the following sub-values:
    * all_correct: whether all questions were answered correctly or not
    * correct: number of correctly answered questions
    * incomplete: number of questions left incomplete
    * incorrect: number of incorrectly answered questions
    * submitted: number of submitted questions
    * total: total number of questions that could be attempted
    * unanswered: number of questions left unanswered (it is unclear what the difference is between this field and the incomplete filed)
  * id: an ID; unclear what this refers to but it could be one section of questions
  * questions: a nested structure tracking the questions in the certification exam
    * id: ID of the question
    * options: a nested structure tracking data specific to each question
      * at: timestamp recording when the question was attempted
      * checked: tracking whether the question was checked
      * correct: tracking whether the question was answered correctly
      * id: another ID; it is unclear what this refers to
      * submitted: tracking whether the question was submitted 
    * user_correct: tracking whether the user got all the questions correct
    * user_incomplete: tracking whether the user completed all of the questions
    * user_result: tracking the outcome of the user's assesment
    * user_submitted: tracking whether the user submitted the assessment 
 * started_at: recording timestamp for when the exam was started
 * user_exam_id: ID for the user's exam attempt

### Example code for processing nested json file

The following sections contain code we were provided in advance to help with processing the json file. 

#### Examples of unrolling the assessments data

In the following example, we use our dot notation with the [] operator to pull out a single item from a list.  Note that sequences.questions is a list (multi-valued).
```python
raw_assessments = spark.read.format("kafka").option("kafka.bootstrap.servers", "kafka:29092").option("subscribe","assessments").option("startingOffsets", "earliest").option("endingOffsets", "latest").load() 

raw_assessments.cache()

assessments = raw_assessments.select(raw_assessments.value.cast('string'))

import json

from pyspark.sql import Row

extracted_assessments = assessments.rdd.map(lambda x: Row(**json.loads(x.value))).toDF()

extracted_assessments.registerTempTable('assessments')

spark.sql("select keen_id from assessments limit 10").show()

spark.sql("select keen_timestamp, sequences.questions[0].user_incomplete from assessments limit 10").show()
```

Missing Values in some json objects - Spark allows some flexibility in inferring schema for json in the case of some of the json objects have a value and others don't have the value.  It infers a null for those.  Here is an example of an obviously made up column called "abc123" and see that it infers null for the column:
```python
spark.sql("select sequences.abc123 from assessments limit 10").show()
```

#### Nested multi-value as a dictionary

Let's see an example of a nested multi-value as a dictionary.  First note that the following will NOT work because sequences value is a dictionary, so id is a key of the nested dictionary:
```python
# does NOT work!
spark.sql("select sequence.id from assessments limit 10").show()
```

We can extract sequence.id by writing a custom lambda transform, creating a separate data frame, registering it as a temp table, and use spark SQL to join it to the outer nesting layer:
```python
def my_lambda_sequences_id(x):
    raw_dict = json.loads(x.value)
    my_dict = {"keen_id" : raw_dict["keen_id"], "sequences_id" : raw_dict["sequences"]["id"]}
    return Row(**my_dict)

my_sequences = assessments.rdd.map(my_lambda_sequences_id).toDF()

my_sequences.registerTempTable('sequences')

spark.sql("select sequences_id from sequences limit 10").show()

spark.sql("select a.keen_id, a.keen_timestamp, s.sequences_id from assessments a join sequences s on a.keen_id = s.keen_id limit 10").show()
```

#### Nested multi-valued as a list

Let's see an example of a multi-valued in the form of a list.  Previously, we saw that we can pull out 1 item using the [] operator. In this example, we will pull out all values from the list by writing a custom labmda transform, creating a another data frame, registering it as a temp table, and joining it to data frames of outer nesting layers.

```python
def my_lambda_questions(x):
    raw_dict = json.loads(x.value)
    my_list = []
    my_count = 0
    for l in raw_dict["sequences"]["questions"]:
        my_count += 1
        my_dict = {"keen_id" : raw_dict["keen_id"], "my_count" : my_count, "id" : l["id"]}
        my_list.append(Row(**my_dict))
    return my_list

my_questions = assessments.rdd.flatMap(my_lambda_questions).toDF()

my_questions.registerTempTable('questions')

spark.sql("select id, my_count from questions limit 10").show()

spark.sql("select q.keen_id, a.keen_timestamp, q.id from assessments a join questions q on a.keen_id = q.keen_id limit 10").show()
```

#### How to handle "holes" in json data

When unrolling the json for the assessments dataset, if you are trying to unroll a key in a dictionary that does not exist for all the items, it will generate an error when you try to reference in the cases it does not exist.

Below is some example code for raw_dict["sequences"]["counts"]["correct"] which exists for some but not all of the json objects.  To keep it from generating errors, you would need to check it piece meal to make sure it exists before referencing it. 

We could just default it to 0 if it doesn't exist.  

However, suppose we want to find the average or standard deviatation, the 0's would skew the data too low.  In order to not include it, I add a level of indirection on top of the dictionary and only add the dictionary if it has meaningful data.  Instead of using the "map()" spark functional transformation, I use the "flatMap()" functional transformation, which removes a level of indirection at the end.

Here is how the flat map works in this case.  Suppose A, B, C, and D are all dictionaries:

```( (A), (), (B), (), (), (C), (D), () )```

flat maps to:

```( A, B, C, D)```

Here is the full pyspark code:

```python
def my_lambda_correct_total(x):
    
    raw_dict = json.loads(x.value)
    my_list = []
    
    if "sequences" in raw_dict:
        
        if "counts" in raw_dict["sequences"]:
            
            if "correct" in raw_dict["sequences"]["counts"] and "total" in raw_dict["sequences"]["counts"]:
                    
                my_dict = {"correct": raw_dict["sequences"]["counts"]["correct"], 
                           "total": raw_dict["sequences"]["counts"]["total"]}
                my_list.append(Row(**my_dict))
    
    return my_list

my_correct_total = assessments.rdd.flatMap(my_lambda_correct_total).toDF()

my_correct_total.registerTempTable('ct')

spark.sql("select * from ct limit 10").show()

spark.sql("select correct / total as score from ct limit 10").show()

spark.sql("select avg(correct / total)*100 as avg_score from ct limit 10").show()

spark.sql("select stddev(correct / total) as standard_deviation from ct limit 10").show()
```

### Business questions answerable with the assessment data

What are my assumptions in going about answering these questions?

* I am assuming that the rough data definitions listed above are approximately accurate, meaning that items like `correct` indicate number of questions an exam-taker gets correct on a given assessment

* I am also assuming that these rows are unique assessments -- that there are no duplicates. This is not a very safe assumption to make, but rectifying duplicates is a task that I believe is outside the scope of this project. 

When thinking about what types of business questions might be important for data scientists to answer, I imagined that those interested in these data would be interested in the overall volume of assessments, and which types were more or less popular as well as which seemed to be more challenging for people. I also imagined that many of these data scientists would be interested in queries that dove into multiple levels of the nested JSON file. 

The first two questions rely on queries that pull from the temporary table created using the following code:

```
extracted_assessments.registerTempTable('assessments')
```

*(1) How many assessments are in the dataset?*

```
spark.sql("select count(keen_id) from assessments").show()
+--------------+
|count(keen_id)|
+--------------+
|          3280|
+--------------+
```

When examining `base_exam_id` and `exam_name`, it appeared that there were different numbers of ids versus names. I could have misinterpreted the meanings of these columns, however it could be that there are differnent versions of the exams that are not recorded in the name although they are in the names. It is difficult to remedy this in the scope of this project, so these results should be interpreted with this in mind. 

*(2) What are the 3 most common and 3 least common courses taken?*

```
# pull out 3 most common
spark.sql("select exam_name, count(exam_name)  from assessments group by exam_name order by count(exam_name) desc").show(3)

# output 
+--------------------+----------------+
|           exam_name|count(exam_name)|
+--------------------+----------------+
|        Learning Git|             394|
|Introduction to P...|             162|
|Introduction to J...|             158|
+--------------------+----------------+

# pull out 3 least common 
spark.sql("select exam_name, count(exam_name)  from assessments group by exam_name order by count(exam_name)").show(3)

# output
+--------------------+----------------+
|           exam_name|count(exam_name)|
+--------------------+----------------+
|Native Web Apps f...|               1|
|Learning to Visua...|               1|
|Nulls, Three-valu...|               1|
+--------------------+----------------+
```

Some of the assessments are missing information about the total number of questions (and number correct or incorrect). In the following, we drop these in our counts of average scores. These missing data are not resolved here, so interpret this output with caution. The third question creates a new temporary table to extract the information needed. Its solution relies that first you import the following:

```
from pyspark.sql import Row
```

*(3) Which 3 exams have the least scores on average? Highest scores?*

```python
# define lambda function
def my_lambda_correct_total(x):
    # load json
    raw_dict = json.loads(x.value)
    # initialize empty list
    my_list = []
    
    # if sequences in dictionary
    if "sequences" in raw_dict:
        # if counts are in sequences
        if "counts" in raw_dict["sequences"]:
            # if correct & total are in counts
            if "correct" in raw_dict["sequences"]["counts"] and "total" in raw_dict["sequences"]["counts"]:
                # pull exam name, count of correctly answered questions, and total into new dictionary    
                my_dict = {"exam_name": raw_dict["exam_name"],
                           "correct": raw_dict["sequences"]["counts"]["correct"], 
                           "total": raw_dict["sequences"]["counts"]["total"]}
                # append row to list
                my_list.append(Row(**my_dict))
    # return list
    return my_list

# use RDD to create table 
my_correct_total = assessments.rdd.flatMap(my_lambda_correct_total).toDF()

# create temp table
my_correct_total.registerTempTable('ct')

# pull 3 highest average scored exams
spark.sql("select exam_name, avg(correct / total) as avg_score from ct group by exam_name order by avg_score desc").show(3)
# output 
+--------------------+---------+                                                
|           exam_name|avg_score|
+--------------------+---------+
|The Closed World ...|      1.0|
|Nulls, Three-valu...|      1.0|
|Learning to Visua...|      1.0|
+--------------------+---------+


# pull 3 lowest average scored exams
spark.sql("select exam_name, avg(correct / total) as avg_score from ct group by exam_name order by avg_score").show(3)
# output
+--------------------+---------+                                                
|           exam_name|avg_score|
+--------------------+---------+
|Client-Side Data ...|      0.2|
|Native Web Apps f...|     0.25|
|       View Updating|     0.25|
+--------------------+---------+

```

### How to use these data

You can create other temporary tables and queries, as shown above. You can also store these in hdfs, as shown in the example below: 

*Grab the desired information*

```
some_assessment_info = spark.sql("select keen_id, exam_name from assessments limit 10")
```

*Write to HDFS*

```
some_assessment_info.write.parquet("/tmp/some_assessment_info")
```

You can check the results in Hadoop by using the following code:

```
# check all files
docker-compose exec cloudera hadoop fs -ls /tmp/
# check specific files
docker-compose exec cloudera hadoop fs -ls /tmp/some_assessment_info/
```

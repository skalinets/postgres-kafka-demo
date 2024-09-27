# postgres-kafka-demo

![img](assets/data-stream.jpg)

Fully reproducible step-by-step demo on how to stream tables from Postgres
to Kafka/KSQL back to Postgres.

I walk through this tutorial and others here on GitHub and on my [Medium blog](https://maria-patterson.medium.com/).  Here is a friend link for open access to the article: [*Data Stream Processing for Newbies with Kafka, KSQL, and Postgres*](https://medium.com/high-alpha/data-stream-processing-for-newbies-with-kafka-ksql-and-postgres-c30309cfaaf8?sk=3da652f7ab08ef3a138241569857e110).  I'll always add friend links on my GitHub tutorials for free Medium access if you don't have a paid Medium membership [(referral link)](https://maria-patterson.medium.com/membership).  

If you find any of this useful, I always appreciate contributions to my Saturday morning [fancy coffee fund](https://github.com/sponsors/mtpatter)!

All components are containerized so that the only things you need to run
through this demo are Docker and docker-compose.

## Data

The data used here was originally taken from the
[Graduate Admissions](https://www.kaggle.com/mohansacharya/graduate-admissions)
open dataset available on Kaggle.
The admit csv files are records of students and test scores with their chances
of college admission.  The research csv files contain a flag per student
for whether or not they have research experience.

## Components

The following technologies are used through Docker containers:
* Kafka, the streaming platform
* Zookeeper, Kafka's best friend
* KSQL server, which we will use to create real-time updating tables
* Kafka's schema registry, needed to use the Avro data format
* Kafka Connect, pulled from [debezium](https://debezium.io/), which will
source and sink data back and forth through Kafka
* Postgres, pulled from debezium, tailored for use with Connect

Most of the containers are pulled directly from official Docker Hub images.
The debezium connect image used here needs some additional packages, so I've
built a [debezium connect image](https://cloud.docker.com/repository/docker/mtpatter/debezium-connect) that I've made available on DockerHub.
It can also be built from the included Dockerfile.

### Build the connect image (optional)

```
docker build -t debezium-connect -f debezium.Dockerfile .
```

### Bring up the entire environment

```
docker-compose up -d
```

## Loading data into Postgres

We will bring up a container with a psql command line, mount our local data
files inside, create a database called `students`, and load the data on
students' chance of admission into the `admission` table.

```
docker run -it --rm --network=postgres-kafka-demo_default \
         -v $PWD:/home/data/ \
         postgres:11.0 psql -h postgres -U postgres
```

Password = postgres

At the command line:

```
CREATE DATABASE students;
\connect students;
```

Load our admission data table:

```
CREATE TABLE admission
(student_id INTEGER, gre INTEGER, toefl INTEGER, cpga DOUBLE PRECISION, admit_chance DOUBLE PRECISION,
CONSTRAINT student_id_pk PRIMARY KEY (student_id));

\copy admission FROM '/home/data/admit_1.csv' DELIMITER ',' CSV HEADER
```

Load the research data table with:

```
CREATE TABLE research
(student_id INTEGER, rating INTEGER, research INTEGER,
PRIMARY KEY (student_id));

\copy research FROM '/home/data/research_1.csv' DELIMITER ',' CSV HEADER
```

## Connect Postgres database as a source to Kafka

The postgres-source.json file contains the configuration settings needed to
sink all of the students database to Kafka.

```
curl -X POST -H "Accept:application/json" -H "Content-Type: application/json" \
      --data @postgres-source.json http://localhost:8083/connectors
```

The connector 'postgres-source' should show up when curling for the list
of existing connectors:

```
curl -H "Accept:application/json" localhost:8083/connectors/
```

The two tables in the `students` database will now show up as topics in Kafka.
You can check this by entering the Kafka container:

```
docker exec -it <kafka-container-id> /bin/bash
```

and listing the available topics:

```
/usr/bin/kafka-topics --list --zookeeper zookeeper:2181
```

## Check Data in Kafka

```
docker run --network=postgres-kafka-demo_default -it --rm confluentinc/cp-kafkacat kafkacat -b kafka:9092 -C -t dbserver1.public.research -J
```

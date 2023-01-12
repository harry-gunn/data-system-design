# data-system-design

## Data Ingestion

#### Postgres

We have a 'Postgres Ingester' job to bring the data from our Postgres databases (in RDS) to the bronze layer of our data lake.
The jobs are kubernetes cron jobs deployed using EKS.
These jobs are scheduled based on the frequency of the data updates - for example every 7 days for the vessel information.

#### Kafka

There is a similar Kafka ingester, but this is a streaming job - also deployed using EKS. We listen to the vessel data positions topic, and use the kafka to aws connector (with a little extra, as this doesn't output in parquet as it comes), and publish into the bronze s3 layer.

#### Weather Data Source

There is an assumption made here that the vessel data positions from the kafka topic is needed information to get the weather data for vessel positions.

This is a similar streaming job that also listens to the vessel data positions kafka topic, and uses that data to call the weather data api. This is then published into the bronze s3 layer.


## The 'data lake'

We use the mesallion architecture to store the data here.

#### Bronze

This is the raw data, stored 'as it comes' but transformed into parquet for ease. We can perform any data quality validation on this to check if the inputs are as expected (e.g. if we expect a field to be non null integer, but it is either null or a string then we would like to know about this change of schema). Each dataset is ingested here into it's own location.

#### Silver

Described as the 'Enterprise view', silver is slightly transformed data that could be used for more data analysis - whether manual reporting or new services. The data here is cleansed but not changed too much - the aim is for this to be completed quickly, to allow it to be the source for other services.

#### Gold

Fully curated data, this is ready to be the source for our reports/dashboards. It should have been transformed enough so that the dashboards need not perform any major processing themselves, ensuring they are as fast and responsive as possible.

#### 'Layer processing' jobs

The bronze to Silver and Silver to Gold jobs take our data between the layers by transforming them and joining with other datasets. An example of a transformation to a specific gold table could be rolling data up from daily to monthly - summing some of it, averaging other parts and so on. These, as with the ingesters, are streaming jobs deployed in EKS.

## Vessel Consumption Prediction

The source for this dataset is our silver layer - it provides 'good enough' data but gets it to that point very quickly. This means we are running our model on as up to date data as possible.

## Dashboards

The source for the dashboards are gold tables. These are heavily processed to supply as much of the data in the form it will be used in as possible - this prevents the user sitting around and waiting for the dashboard to load as it tries to transform data itself.
I have chosen QuickSight as the dashboarding tool as it looks to play very well / easily with s3, but this can be swapped out for other dashboarding tools such as grafana if needed.



Learn about Apache Hudi Transformers with Hands on Lab

![1](https://user-images.githubusercontent.com/39345855/231207046-5fd5992a-7797-487a-9014-2c7605ccacfa.JPG)

What is Apache Hudi Transformers?

* Apache Hudi Transformers is a library that provides data transformation capabilities for Apache Hudi. It provides a set of functions that can be used to transform data within a Hudi table. These transformations can be performed either during ingestion or during update operations. Some of the transformations that can be performed include filtering, mapping, and aggregation.

* The library is designed to work with Hudi's delta streamer, which is a component that allows for incremental data processing on large datasets. When used in conjunction with Hudi Transformers, the delta streamer can apply transformations to the data as it is being ingested, allowing for faster and more efficient processing.

### Sample Dataset
* https://drive.google.com/drive/folders/1BwNEK649hErbsWcYLZhqCWnaXFX3mIsg?usp=share_link
```
try:
    import json
    import uuid
    import os
    import boto3
    from dotenv import load_dotenv

    load_dotenv("../.env")
except Exception as e:
    pass

global AWS_ACCESS_KEY
global AWS_SECRET_KEY
global AWS_REGION_NAME

AWS_ACCESS_KEY = os.getenv("DEV_ACCESS_KEY")
AWS_SECRET_KEY = os.getenv("DEV_SECRET_KEY")
AWS_REGION_NAME = os.getenv("DEV_REGION")

client = boto3.client("emr-serverless",
                      aws_access_key_id=AWS_ACCESS_KEY,
                      aws_secret_access_key=AWS_SECRET_KEY,
                      region_name=AWS_REGION_NAME)


def lambda_handler_test_emr(event, context):
    # ------------------Hudi settings ---------------------------------------------
    glue_db = "hudi_db"
    table_name = "invoice"
    op = "UPSERT"
    table_type = "COPY_ON_WRITE"

    record_key = 'invoiceid'
    precombine = "replicadmstimestamp"
    partition_feild = 'year,month,day'
    source_ordering_field = 'replicadmstimestamp'

    delta_streamer_source = 's3://XXXXX/raw'
    hudi_target_path = 's3://XXXXXX/hudi'
    query = """SELECT * ,extract(year from replicadmstimestamp) as year, extract(month from replicadmstimestamp) as month, extract(day from replicadmstimestamp) as day FROM <SRC> a;"""


    # ---------------------------------------------------------------------------------
    #                                       EMR
    # --------------------------------------------------------------------------------
    ApplicationId = "XXXXXXXXXX"
    ExecutionTime = 600
    ExecutionArn = "arn:aXXXXXXXXXX1645"
    JobName = 'delta_streamer_{}'.format(table_name)

    # --------------------------------------------------------------------------------

    spark_submit_parameters = ' --conf spark.jars=/usr/lib/hudi/hudi-utilities-bundle.jar'
    spark_submit_parameters += ' --conf spark.serializer=org.apache.spark.serializer.KryoSerializer'
    spark_submit_parameters += ' --conf spark.sql.extensions=org.apache.spark.sql.hudi.HoodieSparkSessionExtension'
    spark_submit_parameters += ' --conf spark.sql.catalog.spark_catalog=org.apache.spark.sql.hudi.catalog.HoodieCatalog'
    spark_submit_parameters += ' --conf spark.sql.hive.convertMetastoreParquet=false'
    spark_submit_parameters += ' --conf mapreduce.fileoutputcommitter.marksuccessfuljobs=false'
    spark_submit_parameters += ' --conf spark.hadoop.hive.metastore.client.factory.class=com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory'
    spark_submit_parameters += ' --class org.apache.hudi.utilities.deltastreamer.HoodieDeltaStreamer'

    arguments = [
        "--table-type", table_type,
        "--op", op,
        "--enable-sync",
        "--source-ordering-field", source_ordering_field,
        "--source-class", "org.apache.hudi.utilities.sources.ParquetDFSSource",
        "--target-table", table_name,
        "--target-base-path", hudi_target_path,
        "--payload-class", "org.apache.hudi.common.model.AWSDmsAvroPayload",

        "--hoodie-conf", "hoodie.datasource.write.keygenerator.class=org.apache.hudi.keygen.ComplexKeyGenerator",

        "--hoodie-conf", "hoodie.datasource.write.recordkey.field={}".format(record_key),

        "--hoodie-conf", "hoodie.deltastreamer.source.dfs.root={}".format(delta_streamer_source),
        "--hoodie-conf", "hoodie.datasource.write.precombine.field={}".format(precombine),

        "--hoodie-conf", "hoodie.database.name={}".format(glue_db),
        "--hoodie-conf", "hoodie.datasource.hive_sync.enable=true",
        "--hoodie-conf", "hoodie.datasource.hive_sync.table={}".format(table_name),

        "--transformer-class", "org.apache.hudi.utilities.transform.SqlQueryBasedTransformer",
        "--hoodie-conf", "hoodie.deltastreamer.transformer.sql={}".format(query),
        "--hoodie-conf", "hoodie.datasource.write.partitionpath.field={}".format(partition_feild),
        "--hoodie-conf", "hoodie.datasource.hive_sync.partition_fields={}".format(partition_feild),
    ]

    response = client.start_job_run(
        applicationId=ApplicationId,
        clientToken=uuid.uuid4().__str__(),
        executionRoleArn=ExecutionArn,
        jobDriver={
            'sparkSubmit': {
                'entryPoint': "command-runner.jar",
                'entryPointArguments': arguments,
                'sparkSubmitParameters': spark_submit_parameters
            },
        },
        executionTimeoutMinutes=ExecutionTime,
        name=JobName,
    )
    print("response", end="\n")
    print(response)


lambda_handler_test_emr(context=None, event=None)

```

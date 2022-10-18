# ASEAN-Data-Roadshow
Hanoi Oct.18
Lab 1: EMR Serverless
Amazon EMR Serverless is a new deployment option for Amazon EMR. EMR Serverless provides a serverless runtime environment that simplifies the operation of analytics applications that use the latest open source frameworks, such as Apache Spark and Apache Hive. With EMR Serverless, you don’t have to configure, optimize, secure, or operate clusters to run applications with these frameworks.

EMR Serverless helps you avoid over- or under-provisioning resources for your data processing jobs. EMR Serverless automatically determines the resources that the application needs, gets these resources to process your jobs, and releases the resources when the jobs finish. For use cases where applications need a response within seconds, such as interactive data analysis, you can pre-initialize the resources that the application needs when you create the application.

With EMR Serverless, you'll continue to get the benefits of Amazon EMR, such as open source compatibility, concurrency, and optimized runtime performance for popular frameworks.

EMR Serverless is suitable for customers who want ease in operating applications using open source frameworks. It offers quick job startup, automatic capacity management, and straightforward cost controls.

You can select the Amazon EMR release version for the open source framework version that you want to use. You can also select the specific runtime that you want your application to use, such as Apache Spark or Apache Hive. After you create an application, you can submit data-processing jobs or interactive requests to your application.

Each EMR Serverless application runs on a secure Amazon Virtual Private Cloud (VPC) strictly apart from other applications. Additionally, you can use AWS Identity and Access Management (IAM) policies to define which IAM users and roles can access the application. You can also specify limits to control and track usage costs incurred by the application.

Consider creating multiple applications when you need to do the following:

Use different open source frameworks

Use different versions of open source frameworks for different use cases

Perform A/B testing when upgrading from one version to another

Maintain separate logical environments for test and production scenarios

Provide separate logical environments for different teams with independent cost controls and usage tracking

Separate different line-of-business applications

Find more information about EMR Serverless 

Lab
In this Workshop you are going to launch and EMR Serverless PySpark job with EMR Studio.

We are going upload a CSV file with Taxi rides transactions and convert it into a Parquet file.

Converting files to columnar format, like Parquet or ORC, is one of the best practices to optimize analytics workloads with Amazon EMR.

Create Folders inside S3 bucket
Locate the bucket created for this lab in the S3 Console .
The bucket for this lab should start with the prefix emrserverless. Note the name of the bucket because you are going to use it during the next tasks of the lab.

Now click Create folder. In the box type in script and click Save. This will create a new folder called script inside the bucket.

Repeat the above steps to create folders logs, input and output. You should have the following structure in your bucket:

script
logs (optional)
input
output
Get Sample Data
This workshop will use the New York Taxi dataset which is publicly available. The data is available in CSV format and is around 1.8 MB in size. We will download this file to our local machine and upload it to the "input" folder we created above inside the s3 bucket.

Download the TripData dataset 

Once the file is downloaded we will upload the file to s3 via the AWS Management Console. Inside the AWS Management Console under s3 bucket click on the folder "input".

Upload the file by clicking "Upload". In the upload wizard click "Add files" to browse the file which is downloaded in the step above or drag and drop the file into this window.

Once the file is selected click on "Upload" to upload the file.

Create the code
In your preferred code editor in your local machine, create a file spark-etl.py, copy the following snippet and save it. Alternatively, you can download the script here 
The following code infers the schema on read, adds new column current_date and saves it back in your bucket in parquet format.

import sys
from datetime import datetime

from pyspark.sql import SparkSession
from pyspark.sql.functions import *

if __name__ == "__main__":

    print(len(sys.argv))
    if (len(sys.argv) != 3):
        sys.exit(0)

    spark = SparkSession\
        .builder\
        .appName("SparkETL")\
        .getOrCreate()

    nyTaxi = spark.read.option("inferSchema", "true").option("header", "true").csv(sys.argv[1])

    updatedNYTaxi = nyTaxi.withColumn("current_date", lit(datetime.now()))

    updatedNYTaxi.write.parquet(sys.argv[2]+"/parquet/nyTaxi.parquet")

Note that the script contains a couple of arguments that are going to be provided during the job creation.

You can upload the the python file into your bucket under the script folder.
Create the necessary permissions
To launch our job, you would require an IAM role with the necessary permissions.

Navigate to IAM console  and navigate to Policies located in the left menu.

First, let's create 2 policies that will allow your role to access your s3 Bucket and AWS Glue Catalog. Open in a new tab Create policy.



Let's start first with AWS glue Catalog policy to create and read your Glue Catalog. Select JSON tab selection and insert the following statement.
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "GlueCreateAndReadDataCatalog",
            "Effect": "Allow",
            "Action": [
                "glue:GetDatabase",
                "glue:GetDataBases",
                "glue:CreateTable",
                "glue:GetTable",
                "glue:GetTables",
                "glue:GetPartition",
                "glue:GetPartitions",
                "glue:CreatePartition",
                "glue:BatchCreatePartition",
                "glue:GetUserDefinedFunctions"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}

Click Next, and navigate to the Review tab.
Assign a glue-access-emr a name or assign your preferred name and click Create Policy.


Repeat the same process for the s3 policy. Name your policy s3-access-emr and attach the following json statement. Make sure you replace <INSERT-YOUR-BUCKET-NAME> with your bucket name.
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ReadFromOutputAndInputBuckets",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::<INSERT-YOUR-BUCKET-NAME>",
                "arn:aws:s3:::<INSERT-YOUR-BUCKET-NAME>/*"
            ]
        },
        {
            "Sid": "WriteToOutputDataBucket",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::<INSERT-YOUR-BUCKET-NAME>/*"
            ]
        }
    ]
}

Now, navigate againg to IAM console  and navigate to Roles located in the left menu.

Create role and select Custom Trust Policy. Copy the following Trust Policy that allows EMR Serverlessthis assume your role. The clickNext`.

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "emr-serverless.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}

Select the policies you have created. If you do not see the policies you can refresh the list by clicking in the refresh button. Then click Next.

Assign the name to your role. You can use emr-serverless-job-role as a suggestion.



Your role has been created. if you access your role, you will visualize the following information.



Create the EMR Serverless Job
Now it is time to go to our EMR studio interface .

On the left panel select EMR Serverless and click Get Started.


After clicking on Get Started, click Create and launch EMR Studio.


Automatically you will be redirect to EMR Serverless Application console.



Assign a name to your application and leave the rest of parameters as per default. You can use emr-serverless as a suggestion. Click Create application. (Please note that the application creation would take few seconds)


You can note that our application will start to provision resources for your job once it is submitted. And it will free those resources after 15 minutes of idle time.

Click on the name of your application and choose Submit a Job.
Assign a name ex: taxi-rides
Select the role you have created previously
Navigate to input folder in the s3 bucket and select the script. ``s3:///script/spark-etl.py
And provided the script parameters with input folder and the output path (add a suffix if you require). ["s3://<YOUR-BUCKET-NAME>/input/","s3://<YOUR-BUCKET-NAME>/output/spark"]
Click on Submit Job.


Now the job is submitted and as per configuration the application will start the resources allocation, then it will run the job.



The job will take around ~3 minutes to run. However it can vary depending of the queuing time during the pending status. Find more about the different EMR Serverless job status .



You can navigate to your s3 bucket output and check the generation of the new parquet file



However, since it is a parquet file I would recommend to use AWS Glue Catalog to visualize it.

Crawl and inspect your Data
Navigate to AWS Glue console . Create a AWS Glue Data Crawler. Select Crawlers on the left panel;


Create Crawler
Assign a unique name to your crawler taxi-rides. Then click Next.
On the next screen click on Add a data source. Add a s3 as your Data source and select the path to your output folder in your bucket. Let the rest of the options as per default. Once the data source is added click Next.
Select Create new IAM role and and assign a suffix to the AWSGlueServiceRole-. You can use emr-serverless as a suggestion.
Select Update chosen IAM role to enable access to your s3 bucket. Then click Next.
Choose a Target Database. If you do not have any you can create one by clicking add database. You can assign serverless as your database name. Refresh the database list after its creation and click `Create database.
Select Run Crawler On Demand. Then click Next and finally click Create Crawler.


Once the crawler is created you can run it by selecting the crawler and clicking on Run.
This action will take around ~1 minute.

If the crawler has been successfull in the section Tables you will see a new table in the serverless database with the folder name (output).


If you access the table, you will see the Table details, schema, partitions, indexes. Confirm that the new tables format is parquet. If you navigate at the end of the page you will see the new column current_date is now part of the schema.




Good job! In this lab you have learned how to run a ETL job with EMR Serverless and EMR Studio.

You can proceed now to the next lab about EMR and Sagemaker for Machine Learning use-cases.


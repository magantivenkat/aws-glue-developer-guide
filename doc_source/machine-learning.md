# Machine Learning Transforms in AWS Glue<a name="machine-learning"></a>

AWS Glue provides machine learning capabilities to create custom transforms to cleanse your data\. There is currently one available transform named FindMatches\. The FindMatches transform enables you to identify duplicate or matching records in your dataset, even when the records do not have a common unique identifier and no fields match exactly\. This will not require writing any code or knowing how machine learning works\. FindMatches can be useful in many different problems, such as: 
+ **Matching Customers**: Linking customer records across different customer databases, even when many customer fields do not match exactly across the databases \(e\.g\. different name spelling, address differences, missing or inaccurate data, etc\)\.
+ **Matching Products**: Matching products in your catalog against other product sources, such as product catalog against a competitor's catalog, where entries are structured differently\.
+ **Improving Fraud Detection**: Identifying duplicate customer accounts, determining when a newly created account is \(or might be\) a match for a previously known fraudulent user\.
+ **Other Matching Problems**: Match addresses, movies, parts lists, etc etc\. In general, if a human being could look at your database rows and determine that they were a match, there is a really good chance that the FindMatches transform can help you\.

 You can create these transforms when you create a job\. The transform that you create is based on a source data store schema and example data that you label \(we call this process “teaching” a transform\)\. In this process we generate a file which you label and then upload back which the transform would in a manner learn from\)\. After you teach your transform, you can call it from your Spark\-based AWS Glue job \(PySpark or Scala Spark\) and use it in other scripts with a compatible source data store\. 

After the transform is created, it is stored in AWS Glue\. On the AWS Glue console, you can manage the transforms that you create\. On the AWS Glue **ML transforms** tab, you can edit and continue to teach your machine learning transform\. For more information about managing transforms on the console, see [Working with Machine Learning Transforms on the AWS Glue Console](console-machine-learning-transforms.md)\.

## Types of Machine Learning Transforms<a name="machine-learning-transforms"></a>

AWS Glue gives you the capability to create machine learning transforms to cleanse your data\. You can call these transforms from your ETL script\. Your data passes from transform to transform in a data structure called a *DynamicFrame*, which is an extension to an Apache Spark SQL `DataFrame`\. The `DynamicFrame` contains your data, and you reference its schema to process your data\.

AWS Glue provides the following types of machine learning transforms:

*Find matches*  
Finds duplicate records in the source data\. You teach this machine learning transform by labeling example datasets to indicate which rows match\. The AWS Glue machine learning transform learns which rows should be matches the more you teach it with example labeled data\. Depending on how you configure the transform, the output is one of the following:  
+ A copy of the input table plus a `match_id` column filled in with values that indicate matching sets of records\. The `match_id` column is an arbitrary identifier\. Any records which have the same `match_id` have been identified as matching to each other\. Records with different `match_id`'s do not match\.
+ A copy of the input table with duplicate rows removed\. If multiple duplicates are found, then the record with the lowest primary key is kept\.

### Find Matches Transform<a name="machine-learning-find-matches"></a>

You can use the `FindMatches` transform to find duplicate records in the source data\. A labeling file is generated or provided to help teach the transform\.

#### Getting Started Using the Find Matches Transform<a name="machine-learning-find-mathes-workflow"></a>

Follow these steps to get started with the `FindMatches` transform:

1. Create a table in the AWS Glue Data Catalog for the source data that is to be cleaned\. For information about how to create a crawler, see [Working with Crawlers on the AWS Glue Console](https://docs.aws.amazon.com/glue/latest/dg/console-crawlers.html)\.

   If your source data is a text\-based file such as a comma\-separated values \(CSV\) file, consider the following: 
   + Keep your input record CSV file and labeling file in separate folders\. Otherwise, the AWS Glue crawler might consider them as multiple parts of the same table and create tables in the Data Catalog incorrectly\. 
   + Unless your CSV file includes ASCII characters only, ensure that UTF\-8 without BOM \(byte order mark\) encoding is used for the CSV files\. Microsoft Excel often adds a BOM in the beginning of UTF\-8 CSV files\. To remove it, open the CSV file in a text editor, and resave the file as **UTF\-8 without BOM**\. 

1. On the AWS Glue console, create a job, and choose the **Find matches** transform type\.
**Important**  
The data source table that you choose for the job can't have more than 100 columns\.

1. Tell AWS Glue to generate a labeling file by choosing **Generate labeling file**\. AWS Glue takes the first pass at grouping similar records for each `labeling_set_id` so that you can review those groupings\. You label matches in the `label` column\.
   + If you already have a labeling file, that is, an example of records that indicate matching rows, upload the file to Amazon Simple Storage Service \(Amazon S3\)\. For information about the format of the labeling file, see [Labeling File Format](#machine-learning-labeling-file)\. Proceed to step 4\.

1. Download the labeling file and label the file as described in the [Labeling](#machine-learning-labeling) section\.

1. Upload the corrected labelled file\. AWS Glue runs tasks to teach the transform how to find matches\.

   On the **Machine learning transforms** list page, choose the **History** tab\. This page indicates when AWS Glue performs the following tasks:
   + **Import labels**
   + **Export labels**
   + **Generate labels**
   + **Estimate quality**

1. To create a better transform, you can iteratively download, label, and upload the labelled file\. In the initial runs, a lot more records might be mismatched\. But AWS Glue learns as you continue to teach it by verifying the labeling file\.

1. Evaluate and tune your transform by evaluating performance and results of finding matches\. For more information, see [Tuning Machine Learning Transforms in AWS Glue](add-job-machine-learning-transform-tuning.md)\.

#### Labeling<a name="machine-learning-labeling"></a>

When AWS Glue generates a labeling file, records are selected from your source table\. Based on previous training, AWS Glue identifies the most valuable records to learn from\.

The act of *labeling* is editing a labeling file \(such as in a spreadsheet\) and adding identifiers, or labels, into the `label` column that identifies matching and nonmatching records\. It is important to have a clear and consistent definition of a match in your source data\. AWS Glue learns from which records you designate as matches \(or not\) and uses your decisions to learn how to find duplicate records\.

When a labeling file is generated by AWS Glue, approximately 100 records are generated\. There are roughly 10 records per unique `labeling_set_id` generated\. You label each set of records for each `labeling_set_id`\.

##### Tips for Editing Labeling Files in a Spreadsheet<a name="machine-learning-labeling-tips"></a>

When editing the labeling file in a spreadsheet application, consider the following:
+ The file might not open with column fields fully expanded\. You might need to expand the `labeling_set_id` and `label` columns to see content in those cells\.
+ If the primary key column is a number, such as a `long` data type, the spreadsheet might interpret it as a number and change the value\. This key value must be treated as text\. To correct this problem, format all the cells in the primary key column as **Text data**\.

#### Labeling File Format<a name="machine-learning-labeling-file"></a>

The labeling file that is generated or provided to AWS Glue to teach your `find matches` transform follows this format:
+ It is a comma\-separated values \(CSV\) file\. 
+ It must be encoded in `UTF-8` without BOM \(byte order mark\)\. If you edit the file using Microsoft Windows, it might be encoded with `cp1252`\.
+ It must be in an Amazon S3 location to pass it to AWS Glue\.
**Note**  
 Because it has additional columns, the labeling file has a different schema from a file that contains your source data\. Place the labeling file in a different folder from any transform input CSV file so that the AWS Glue crawler does not consider it when it creates tables in the Data Catalog\. Otherwise, the tables created by the AWS Glue crawler might not correctly represent your data\. 
+ The first two columns \(`labeling_set_id`, `label`\) are required by AWS Glue\. The remaining columns must match the schema of the data that is to be processed\.
+ For each `labeling_set_id`, you identify all matching records using the same label\. A label is a unique string placed in `label`\. We recommend using labels that contain simple characters, such as A, B, C, and so on\. Labels are case sensitive and are entered in the `label` column\. The following is an example of labeling the data:    
<a name="table-labeling-data"></a>[\[See the AWS documentation website for more details\]](http://docs.aws.amazon.com/glue/latest/dg/machine-learning.html)
+ The scope of the labels is limited to the `labeling_set_id`\. So labels do not cross `labeling_set_id` boundaries\. For example, a label "A" in `labeling_set_id` 1 does not have any relation to label "A" in `labeling_set_id` 2\.
+ Labels should include examples of both matching and nonmatching records for the `labeling_set_id`\.
+ If a record does not have any matches, then assign it a unique label\.

**Important**  
Confirm that the IAM role that you pass to AWS Glue has access to the Amazon S3 bucket that contains the labeling file\. By convention, AWS Glue policies grant permission to Amazon S3 buckets or folders whose names are prefixed with **aws\-glue\-**\. If your labeling files are in a different location, add permission to that location in the IAM role\.
# Step 1: Create an S3 Bucket and IAM Policy in AWS
Log in to the AWS Management Console and navigate to the IAM service.

In the left-hand navigation pane, select Policies > Create Policy.

Choose the JSON tab and paste the following policy, replacing your-s3-bucket-name with your actual bucket name:
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowSnowflakeAccessToBucket",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::your-s3-bucket-name",
                "arn:aws:s3:::your-s3-bucket-name/*"
            ]
        }
    ]
}
Click Next, name your policy (e.g., snowflake_s3_access_policy), and click Create policy.

# Step 2: Create an AWS IAM Role
In the IAM console, select Roles > Create role.

Select AWS account as the trusted entity type, then select Another AWS account.   

For the Account ID, temporarily enter your own AWS account ID (you will update this with Snowflake's specific ARN in a later step).   

Check Require external id, and enter a placeholder value like 0000.   

Click Next, search for and select the IAM policy you created in Step 1 (snowflake_s3_access_policy), and click Next.

Give the role a name (e.g., snowflake_s3_role) and click Create role.

Open the newly created role and copy its ARN (e.g., arn:aws:iam::123456789012:role/snowflake_s3_role).

Step 3: Create a Storage Integration in Snowflake
Log into Snowflake using a role with ACCOUNTADMIN privileges (or a role granted sufficient privileges) and run the following SQL command:
CREATE OR REPLACE STORAGE INTEGRATION s3_init
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'S3'
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = 'arn:aws:iam::123456789012:role/snowflake_s3_role'
  STORAGE_ALLOWED_LOCATIONS = ('s3://your-s3-bucket-name/');

# Step 4: Retrieve Snowflake's AWS Parameters & Update AWS Trust Policy
Run the DESCRIBE command in Snowflake to view the integration details: 
<<DESC STORAGE INTEGRATION s3_init;>>

Look for two specific properties in the output:

STORAGE_AWS_IAM_USER_ARN (The AWS IAM user created for your Snowflake account)

STORAGE_AWS_EXTERNAL_ID (The unique security token required for trust)

Go back to the AWS Console > IAM > Roles and open your snowflake_s3_role.

Go to the Trust relationships tab and click Edit trust policy.

Update the JSON policy to substitute the temporary account ID with Snowflake's STORAGE_AWS_IAM_USER_ARN and add the sts:ExternalId condition using your STORAGE_AWS_EXTERNAL_ID:

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "<STORAGE_AWS_IAM_USER_ARN_FROM_SNOWFLAKE>"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "sts:ExternalId": "<STORAGE_AWS_EXTERNAL_ID_FROM_SNOWFLAKE>"
                }
            }
        }
    ]
}
Click Update policy.
# Step 5: Create a File Format and External Stage in Snowflake
Define how your data files are structured (e.g., CSV, JSON) and create the named external stage pointing to your S3 bucket:
-- Create a file format object
CREATE OR REPLACE FILE FORMAT my_csv_format
  TYPE = CSV
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1;

-- Create the external stage using the storage integration
CREATE OR REPLACE STAGE my_s3_stage
  STORAGE_INTEGRATION = s3_init
  URL = 's3://your-s3-bucket-name/'
  FILE_FORMAT = my_csv_format;

# Step 6: Ingest Data Into a Table
You can now ingest data from your staged S3 files directly into a Snowflake target table using the COPY INTO command:
-- Optional: Test file parsing and check for errors before loading
COPY INTO my_target_table
  FROM @my_s3_stage
  VALIDATION_MODE = RETURN_ERRORS;

-- Execute the actual data load
COPY INTO my_target_table
  FROM @my_s3_stage
  FILE_FORMAT = (FORMAT_NAME = 'my_csv_format')
  ON_ERROR = 'SKIP_FILE';
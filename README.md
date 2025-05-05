# Use Case for this Project
Perform the ETL (Extract, Transform, Load) process using AWS Glue along with Amazon S3, RDS, Redshift.

# AWS Services
- **AWS Glue**
- **Amazon S3** (Simple Storage Service)
  - Created a bucket for `aws-bucket-for-athena-results` to store Athena Query Results.
- **Amazon Athena**
- **Amazon RDS** (Amazon Aurora and RDS - Relational Database Service)
  - Required MySQL Workbench to extract data from csv files and load it to RDS db.
- **Amazon EC2** (Elastic Compute Cloud)
  - Created Security group for connection between RDS and MySQL Workbench, RDS and AWS Glue.
- **Amazon VPC**
  - Set up VPC Endpoints for connection between RDS and AWS Glue
- **Amazon Redshift**
  
# How does it work?
### Step 1: Create an IAM role for AWS Glue to get full permission
- Navigate to IAM > Roles > Create Role
  - Trusted entity type: `AWS Service`
  - Service/Use case: `Glue`
  - Add permissions: `Administrator Access`
  - Role name: `GlueFullAccessRole`
  - Description: `Allows Glue to call AWS services on your behalf.`
 
### Step 2: Create data source 1
#### Step 2.1 Create S3 bucket
  - Navigate to S3 > Create bucket
    - Bucket name: `banking-customer-info-mjk`
    - *Bucket name has to be unique globally.
  - Click on the bucket
    - Under the `Objects` tab, click on `Upload` > Add files > Upload `accounts.csv`
#### Step 2.2 Create AWS Glue Database and add tables using Crawler
  - Navigate to AWS Glue > Data Catalog > Databases > Add database
  - Database name: `accounts-glue-db` > Create database
  - Click on the database
    - Click on `Add tables using crawler`
      - *Crawler is a tool to extract data/metadata from a csv file.
    - Crawler name: `accounts-crawler`
      - Is your data already mapped to Glue tables? `No`
      - Data sources > `Add a data source`
        - Data source: `S3`
        - Location of S3 data: `In this account`
        - S3 path: `Browse S3` and select a file (e.g. `s3://banking-customer-info-mjk`) > `Add an S3 data source`
      - Choose an IAM role: `GlueFullAccessRole`
      - Target database: `accounts-glue-db`
      - Crawler schedule - Frequency: `On demand`
      - Create Crawler
    - Run crawler `accounts-crawler`
      - This would create a table in the database with the data extracted from csv files.

#### Step 2.3: Query AWS Glue Database Table with Athena
  - Navigate to Amazon Athena > Go to Workgroup > Primary > Edit
    - Query result configuration: `s3://aws-bucket-for-athena-results/results/`
      - This will create a folder called results if it's not already existing.
  - Go to Launch Query Editor > Run query

### Step 3: Create data source 2
#### Step 3.1 Create RDS database
  - Navigate to Aurora and RDS > Databases > Create a database
    - Choose a database creation method: `Easy create`
    - Configuration: `MySQL`
    - DB instance size: `Free tier`
    - DB instance identifier: `customers-info-rds-db-instance`
    - Master username: admin
    - Master password: {your_password}
    - Create database
#### Step 3.2 Set up Security Groups to connect from MySQL Workbench
  - Modify RDS DB instance
    - Additional configuration: select `Publicly accessible` to connect from MySQL Workbench
  - Open EC2 > Security Groups > Create security group
    - Security group name: `SG-Open-MySQL`
    - Description: `Allows MySQL Access to Developers`
    - VPC Info: select the default one
    - Inbound rules
      - > Add rule (to allow MySQL Connection)
        - Type: `Custom TCP`
        - Port range: `3306`
        - Source: Anywhere-IPv4
        - Create security group
  - Navigate to RDS > `customers-info-rds-db-instance` > Modify
    - Connectivity > Select security group `SG-Open-MySQL`
  - Open MySQL Workbench
    - Setup new connection
      - Connection Name: `RDSConnection`
      - Connection Method: `Standard (TCP/IP)`
      - Hostname: This is the endpoint of the RDS db instance `customers-info-rds-db-instance`
      - Port: This can be found from the RDS db instance
      - Username: `admin`
      - Test Connection > Password: {your_password}
        - You should be able to see the popup saying "Successfully made the MySQL connection"

#### Step 3.3 Create a table in MySQL Workbench
- Double click on `RDSConnection`
  - Run the query `create database customerinfodb`
- Click on `customerinfodb` newly created
  - Right click on `Tables` > Select `Table Data Import Wizard`
    - Create new table: `customerinfodb`, `customers`  

#### Step 3.4 Set up Security Groups and VPC Endpoints for connection between AWS Glue and RDS
  - Navigate to AWS Glue > Connectors > Create connection
    - Choose data source: `Amazon Aurora (RDS)`
    - Database instances: `customers-info-rds-db-instance`
    - Database name: `customerinfodb` (what was created in MySQL Workbench)
    - Username: `admin`
    - Password: {your_password}
    - Connection name: `RDSConnection`
  - Open EC2 > Security Groups > `SG-Open-MySQL` security group > Add Inbound rule
    - Inbound rules
      - > Add rule (to allow AWS Glue Connection)
        - Type: `All TCP`
        - Source: Anywhere-IPv4
  - Navigate to VPC > Endpoints > Create endpoint
    - Name tag: `VPC Endpoint to S3`
    - Service category: `AWS services`
    - Services: `Service Name = com.amazonaws.us-east-1.s3` > Select `Gateway` type
    - VPC: select the default one
    - Route tables: select the default one
    - Create endpoint
  - Test connection of `RDSConnection`

#### Step 3.5 Add tables using Crawler
  - Navigate to AWS Glue > Databases > `customers-glue-db`
  - Click on `Add tables using crawler`
  - Crawler name: `customers-crawler`
      - Is your data already mapped to Glue tables? `No`
      - Data sources > `Add a data source`
        - Data source: `JDBC`
        - Connection: `RDSConnection`
        - Include path: `customerinfodb/%`
        - Add a JDBC data source
      - Choose an IAM role: `GlueFullAccessRole`
      - Target database: `customers-glue-db`
      - Crawler schedule - Frequency: `On demand`
      - Create Crawler
    - Run crawler `customers-crawler`
      - This would create a table in the database with the data extracted from RDS db table within MySQL Workbench.

### Step 4: Create target - Amazon Redshift
#### Step 4.1 Create IAM role
  - Navigate to IAM > Roles > Create role
    - Trusted entity type: `AWS Service`
    - Service/Use case: `Redshift`
      - Use case: `Redshift - Customizable`
    - Add permissions: `Administrator Access`
    - Role name: `RedshiftFullAccessRole`
  - Description: `Allows Redshift clusters to call AWS services on your behalf.`
  - Create role
#### Step 4.2 Set up Redshift
  - Navigate to Amazon Redshift > Select `Try Redshift Serverless free trial`
    - Configuration: `Customize settings`
    - Target namespace: `mynamespace`
    - Associated IAM roles: `RedshiftFullAccessRole`
    - Workgroup name: `myworkgroup`
    - Save configuration

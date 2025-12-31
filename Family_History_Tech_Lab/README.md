### Note: This is a readme I contributed to in my research assistant position. I wrote over 50% of this documentation. The section “Running the backend locally” until the subsection “Normal Start” was written exclusively by me. Any sensitive information has been replaced with placeholders.

# Table of Contents
- [API Documentation](#api-documentation)
- [Understanding RelativeFinder's backend](#understanding-relativefinders-backend)
  - [AWS S3](#aws-s3)
  - [PostgreSQL (UGP)](#postgresql-ugp)
  - [FamilySearch](#familysearch)
  - [Acquiring Data](#acquiring-data)
- [Running the backend locally](#running-the-backend-locally)
  - [How to connect to the S3 database](#how-to-connect-to-the-s3-database)
  - [How to connect to the RDS Postgres database](#how-to-connect-to-the-rds-postgres-database)
    - [How to make the tunnel_prd.sh script work](#how-to-make-the-tunnel_prdsh-script-work)
  - [If the "rf-backend-beta-tunnel" EC2 instance is off or gone](#if-the-rf-backend-beta-tunnel-ec2-instance-is-off-or-gone)
  - [How to configure the local frontend to use the local backend rather than the deployed backend](#how-to-configure-the-local-frontend-to-use-the-local-backend-rather-than-the-deployed-backend)
  - [If you've done everything here but it's still not working](#if-youve-done-everything-here-but-its-still-not-working)
  - [If you want to locally test API endpoints...](#if-you-want-to-locally-test-api-endpoints)
  - [Normal Start](#normal-start)
  - [Debug](#debug)
  - [Testing](#testing)
- [Project Structure](#project-structure)
  - [Routes](#routes)
  - [Services](#services)
  - [DAO](#dao)
  - [DB](#db)
  - [Error Handling](#error-handling)
- [Future Code Maintenance Tips](#future-code-maintenance-tips)
  - [Avoiding Bugs](#avoiding-bugs)
  - [API Documentation](#api-documentation)
  - [Database Migration](#database-migration)

# API Documentation
The full documentation for the API is found in docs/API.md
The backend is hosted on `[removed].amazonaws.com`

# Understanding RelativeFinder's backend
This backend is designed to bridge communication between three databases: AWS S3, PostgreSQL (hosted on AWS RDS), and FamilySearch. It also manages calculation between trees to find how/if people are related!

## AWS S3
This s3 database, called rf-trees in the aws prd account, holds only users' family trees. Each tree contains basic information namely the pid (person id) of the user, the length of their tree, and their tree as a 2D array. The JSON object of one such tree would look something like the following:
```
{
  pid: 'ABC-DEFG',
  length: 52384,
  tree: [
    [1,'ABC-DEFG'],
    [2,'BBC-DEFG'],
    [3,'CBC-DEFG'],
    [4,'DBC-DEFG'],
    ...
    ]
}
```
Each subarry within the tree array represents an individual person. The first element is their Ancestral Reference Number (ANUM). This number simply represent the position the person is within the user's tree; i.e. user = 1, father = 2, mother = 3, father's father = 4, father's mother = 5, etc. (see https://www.familysearch.org/en/wiki/Genealogy_Numbering_Systems_(National_Institute) ). The second element is that person's pid.

## PostgreSQL (rf-database)
PostgreSQL is a relational database which holds the bulk of RelativeFinder's data. This database, called rf-database in the aws prd account under RDS instances, is sometimes called 'UGP' for short because it holds information on all users, groups, and people (amongst other information). The schemas for all the tables found for this database can be found in src/db/ugp/schema.txt.

## FamilySearch
Almost all data is obtained from FamilySearch. FamilySearch has provided FHTL (RelativeFinder) with a privateKey that allows RelativeFinder to retain a FamilySearch session token for a user. These APIs are then called using a user's FamilySearch token.

## Acquiring Data
RelativeFinder currently supports comparing users' trees up to 16 generations. Unfortunately, however, FamilySearch's API only allows us to retrieve 8 generations at any time. So, to get 16 generations, RelativeFinder first calls the API endpoint for the user, and then calls the API endpoint for each ancestor in the 8th generation. FamilySearch provides the aforementioned ANUMs which are recalculated to portray relationship to the user.

When the tree information is obtained, the tree is uploaded to AWS S3. The people in the tree are also upserted to Postgres (UGP) (upsert means that they are added if they don't exist or updated if they do).

# Running the backend locally
This project runs using node.js. Make sure Node is installed, at least version 16.

In order to run this backend locally, you need to connect to these two databases: 
1. The `"rf-trees"` s3 database
2. The `"rf-database"` RDS Postgres database. 

## How to connect to the S3 database
To access the s3 database, the backend knows where to get find it, but you need to authorize yourself with the AWS prd acount environment variables. To do this:

1. Create a file in the src directory named `.env` for Mac or Linux, or `env.bat` for Windows.
2. Login to aws.byu.edu.
3. Click on the `[removed]` account.
4. Select `command line or programmatic access`.
5. Finally, select the appropriate `macOS and Linux` or `Windows` tab, copy the AWS environment variables (option 1), and paste them to the .env file. This allows the server to access AWS services.

## How to connect to the RDS Postgres database
Start by pasting the following into your .env file. If you are on Mac or Linux, change each `SET` to `export` and put quotation marks (`"`) around each string value. (you can find the postgres password in the AWS prd account Parameter Store under "/relative-finder/rf-postgres-password"):
```
SET POSTGRES_ENDPOINT=localhost
SET POSTGRES_USERNAME=relativefinder
SET POSTGRES_NAME=relativefinder
SET POSTGRES_PASSWORD=<REPLACE WITH POSTGRES PASSWORD>
SET DEBUG=rf-backend:*
```

The RDS postgres database is configured to only allow connections coming from particular sources within its own network (VPC) in AWS. There is an EC2 instance that is configured to have access to the database. The way this EC2 has access to the database is by putting the EC2's private IPv4 address as an incoming rule to the `[removed]` security group, which is attached to the database.

For you to access the database when you are running this backend locally, you will need to tunnel all database requests from your machine through the EC2 and then onto the RDS database. If you haven't set up the tunnel to work, follow the steps below. Otherwise, the way to do this is easy, just run the tunnel_prd.sh script inside /src and leave it running. 
To connect to the database locally through aws run the command psql -h [AWS DATABASE URL] -p [PORT] -U [username] (all the info can be found in the .pgpass file)
### How to make the tunnel_prd.sh script work
The tunnel_prd.sh script contains the following:
```
#!/bin/bash

ssh rfdb -v -NL localhost:5432:[removed].rds.amazonaws.com:5432
```

You'll notice that is says "rfdb". rfdb is the name of an ssh configuration that you'll need to create. Making this will make it easier to connect to your EC2 instance than the traditional pasting in the url link after the ssh command along with the key.pem file. Here's what you'll need to do to create this configuration:

1.  Go to your home directory with `cd ~`

2. Go to your .ssh directory with `cd .ssh`, if you don't have one, create one with `mkdir .ssh`

3. Create a file called config using `touch config` on Mac/Linux or `New-Item config` on Windows, or if you already have one, skip this step.

4. Paste this text (using `vim config` on Mac/Linux, or `Set-Content config '<text to past in>'` on Windows) in that config file and replace the Hostname with the public IPv4 address of the EC2 instance (User ubuntu may change to "ec2-user" if EC2 instance is not the Ubuntu OS, if the EC2 tunneling instance is not running or is deleted, complete the next section of instructions titled '`If the "rf-backend-beta-tunnel" EC2 instance is off or gone`' below before continuing these steps):
```
Host rfdb
    User ubuntu
    Hostname <REPLACE WITH PUBLIC IPv4 ADDRESS OF EC2 INSTANCE>
    IdentityFile ~/.ssh/rf_db_key
    Port 22
```

5. Create a file called rf_db_key using `touch rf_db_key` on Mac/Linux or `New-Item rf_db_key` on Windows, or if you already have one, skip this step.

6. Paste in (using `vim rf_db_key` on Mac/Linux, or `Set-Content rf_db_key '<text to past in>'` on Windows) the private key for accessing the EC2 into the rf_db_key file. You can find the key in an S3 bucket in the AWS prd account called `[removed]`. Inside that bucket, the name of the key is called `[removed].pem`. Download it and paste the text into the rf_db_key file.

7. If you are using Mac or Linux, run `chmod 400 rf_db_key`

Now you should be able to connect to the EC2 just by typing `ssh rfdb`, and now the tunnel_prd.sh script should work.

### Once that's all set up

In summary, when you use npm start to run the backend in one terminal, open another terminal to run the tunnel_prd.sh script.

Please note that the config.yml file in the /source/config folder contains information for database endpoints. These can be changed if wanting to run another database or a local version of the database.

## If the "rf-backend-beta-tunnel" EC2 instance is off or gone
If the EC2 instance is off, sweet. Just turn it on.

If the EC2 instance is gone, you need to make a new one (Name it something along the lines of `rf-backend-beta-tunnel`). Create a new EC2 instance. The default OS for launching new EC2s is the Amazon Linux OS. You can use that, but sometimes it has a hard time downloading the postgres CLI (psql) because that OS uses yum to install packages. If you're struggling with downloading psql with yum, just create an EC2 with the Ubuntu OS. Ubuntu uses apt-get, and it's more reliable for downloading psql.

Once you've created a new EC2 instance:

1. Add the `ssh-[removed]` security group to the EC2 instance

2. Add the private IPv4 address of the EC2 instance to the `[removed]` security group (you can find that security group by looking through the security groups on the rf-database in aws RDS) as an incoming rule. When creating the rule, make sure the type of connection is PostgreSQL, and in the custom IP address add a /32 subnet mask to the end of the IP address.

3. Install psql on the ec2 instance

4. Create a file called `[removed]` on the ec2 instance and paste this text into it (you can find the postgres password in the AWS prd account Parameter Store under "/[removed]"):

`[removed].rds.amazonaws.com:5432:[removed]:<REPLACE WITH POSTGRES PASSWORD>`

5. On your local machine, change your ~/.ssh/config to use the public IPv4 address of the new EC2 instance

## How to configure the local frontend to use the local backend rather than the deployed backend
In the `[removed]` repo, in /src/src/environments/environment.ts, change the RF_BACKEND variable to be `"http://localhost:3000"` rather than `"https://[removed].amazon.byu.edu"`, then save and start the frontend with "ng serve".

## If you've done everything here but it's still not working
1. Verify that you're not running anything else on port 5432 of your local machine. If you've installed Postgres/psql on your own machine, it's highly likely that your local Postgres server is intercepting the tunnel script on port 5432. If you are running a local Postgres server, kill that process.
If your tunnel.sh script is running correctly the last line of output will look something like this " listening port 5432 for [removed].rds.amazonaws.com port 5432, connect from ::1 port 54793 to ::1 port 5432, nchannels 3"

>- For Mac or Linux:
>
>>>Run this command to find the process
>
>>> `sudo lsof -i :5432`
>
>>> Run this command to kill that process
>
>>> `sudo kill -9 <PID>`
>
>- For Windows:
>>> Navigate to /c/Program Files/PostgreSQL/15. The path may vary slightly. Once you're there, you should see a directory called `data`. 
>
>>>Once there, run
>
>>>`pg_ctl -D data stop`
>
>>>to turn off the server. You may need to be running your terminal in Admin mode, and you may need to change the path to a version other than 15.
>
>>> If you are on Windows, and a process other than a Postgres server is running on that port, find some online solution to kill that process.

2. Look over the aws security groups, make sure you have an inbound rule to the EC2 instance with your IP address, and make sure the EC2 has an inbound rule to the RDS Postgres database with its private IP.

3. List of front end errors and likely solutions
- Can't read the user's tree: probably can't connect to the S3 bucket
- Can't see if user exists: Probably an issue with your .env file's environment variables for AWS access
- Can't see if user exists: May also be that you're running a local Postgres server which interferes with the tunnel script.
- (Not a message) Nothing loads for a long time, then gives an error message finally: Probably a security group problem with connection to the RDS Postgres database. It takes a while to load since the backend is trying to access the database but the IP isn't whitelisted to access the RDS database.

## If you want to locally test API endpoints...
If you are locally testing API endpoints, be sure to include your JWT as an `Authorization` header (the JWT stores FamilySearch login token and other information). The JWT can be obtained by running the fs-auth frontend found in the fs-auth directory. It can be run with `http-server` or `npx http-server` (if you are in the directory) and is hosted on `localhost:8080`. Also, `https://hoppscotch.io/` is a great site to use for API testing (for example GET `http://localhost:3000/health` will retrieve the health status. Be sure to use `http` and NOT `https`).

Before starting, be sure to run the following within the src directory:
```
npm install
```

## Normal Start

The server is set to run on port 3000. To start the server, run the following command within the src directory:
```
npm start
```
## Debug
The following will run the server in debug mode
```
npm run dev
``` 

## Testing
Test cases are found in the services/test. These are run in the test API endpoint (admin restricted access). If any number of tests fail for any service, first fix the first failing test since some tests are dependent on the success of others (i.e. `UserService.remove()` depends on `UserService.add()` actually working).

# Project Structure
To avoid confusion, this project follows a strict layered architecture design. This means that layers only call their immediate lower level layer. I.e., routes only call services, services only call DAOs, and DAOs only call DBs. Equally, routes do not call each other, neither services, nor DAOs, nor DBs. This helps maintain the code allowing bugs to be quickly found since the code is not very interdependent.

Note that routes can call multiple services and services can call multiple DAOs. I.e. the TreeService receives tree information from the treeDAO that receives its information from fs-api (db). That information contains people within the tree that need to be added to the people table. So the TreeService gives that information to the personDAO which gives it to the postgreSQL db.

## Routes
The routes are the basic API endpoints. Each route connects to an appropriate service.
## Services
The services are the logic center of the backend. They are called by the routes, use DAOs to get appropriate information, and process information by themselves.
## DAO
The DAOs interact with the databases to get needed information and give the information to the services. They also change the information into what is expected by the service.
## DB
The databases hold the needed information.
## Error Handling
Errors are handled within app.js using express. Express is able to catch all thrown errors from app.js without the use of any try/catches. However, express cannot catch asynchronous errors by itself. Each asynchronous route uses asyncHandler which basically wraps the function so any error can be caught by express. Thus, errors can be thrown anywhere and will be caught in app.js unless there is an underlying try/catch.

# Future Code Maintenance Tips
## Avoiding Bugs
### Exception Handling
Do NOT use `res.send()` to handle errors within the route functions. `res.send(400)` would send the status of 400 as a response BUT code later would still be executed since there is no `return` statement. Instead, throw an exception like: `throw new Exception(400, Exception.SERVER_ERROR)`.

The code is written such that throwing an exception anywhere, specifically the Exception object found within the model directory, will be returned as the response unless it is explicitly handled.

## API Documentation
The API documentation (found in docs/API.md) should be up to date. If you update, delete, or add any route please update the documentation.

## Database Migration
Documentation for the database migration from the old relative finder can be found in /docs

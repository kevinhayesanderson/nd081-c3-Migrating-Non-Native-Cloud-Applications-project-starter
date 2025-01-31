# TechConf Registration Website

## Project Overview
The TechConf website allows attendees to register for an upcoming conference. Administrators can also view the list of attendees and notify all attendees via a personalized email message.

The application is currently working but the following pain points have triggered the need for migration to Azure:
 - The web application is not scalable to handle user load at peak
 - When the admin sends out notifications, it's currently taking a long time because it's looping through all attendees, resulting in some HTTP timeout exceptions
 - The current architecture is not cost-effective 

In this project, you are tasked to do the following:
- Migrate and deploy the pre-existing web app to an Azure App Service
- Migrate a PostgreSQL database backup to an Azure Postgres database instance
- Refactor the notification logic to an Azure Function via a service bus queue message

## Dependencies

You will need to install the following locally:
- [Postgres](https://www.postgresql.org/download/)
- [Visual Studio Code](https://code.visualstudio.com/download)
- [Azure Function tools V3](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=windows%2Ccsharp%2Cbash#install-the-azure-functions-core-tools)
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
- [Azure Tools for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-node-azure-pack)

## Project Instructions

### Part 1: Create Azure Resources and Deploy Web App
1. Create a Resource group
2. Create an Azure Postgres Database single server
   - Add a new database `techconfdb`
   - Allow all IPs to connect to database server
   - Restore the database with the backup located in the data folder
3. Create a Service Bus resource with a `notificationqueue` that will be used to communicate between the web and the function
   - Open the web folder and update the following in the `config.py` file
      - `POSTGRES_URL`
      - `POSTGRES_USER`
      - `POSTGRES_PW`
      - `POSTGRES_DB`
      - `SERVICE_BUS_CONNECTION_STRING`
4. Create App Service plan
5. Create a storage account
6. Deploy the web app

### Part 2: Create and Publish Azure Function
1. Create an Azure Function in the `function` folder that is triggered by the service bus queue created in Part 1.

      **Note**: Skeleton code has been provided in the **README** file located in the `function` folder. You will need to copy/paste this code into the `__init.py__` file in the `function` folder.
      - The Azure Function should do the following:
         - Process the message which is the `notification_id`
         - Query the database using `psycopg2` library for the given notification to retrieve the subject and message
         - Query the database to retrieve a list of attendees (**email** and **first name**)
         - Loop through each attendee and send a personalized subject message
         - After the notification, update the notification status with the total number of attendees notified
2. Publish the Azure Function

### Part 3: Refactor `routes.py`
1. Refactor the post logic in `web/app/routes.py -> notification()` using servicebus `queue_client`:
   - The notification method on POST should save the notification object and queue the notification id for the function to pick it up
2. Re-deploy the web app to publish changes

## Monthly Cost Analysis
Complete a month cost analysis of each Azure resource to give an estimate total cost using the table below:

| Azure Resource | Service Tier | Monthly Cost |
| ------------ | ------------ | ------------ |
| *Azure Postgres Database* | Single server    | 3241.05 INR per month |
| *Azure Service Bus*   | Standard        | $10.00 per month            |
| *Azure Function App* | See Azure App Service Plan        |              |
| *Azure Web App* | See Azure App Service Plan        |              |
| *Azure App Service Plan* | B1        | 1031.943 INR per month             |
|*Total Monthly Cost*||5200 INR or $63 per month |

## Architecture Explanation
The architecture consists of four major parts: a PostgreSQL database, a webapp, a service bus queue, and a service bus trigger function app.

The database is used for persisting data.

The webapp is used for displaying and altering data.

The service bus queue is used for decoupling the updating of data from sending out notification emails.

For Azure Function, the consumption plan is chosen to minimize cost and as the it is a simple notification task.

For Azure Function App, Azure Web App and Azure App service, the selected plan is cheaper than on-premise solution and we don't have to worry about the infrastructure, since the app doest require much computaional resourses, I have gone with B1 plan with scale instances of 3.

For Azure Postgres Database, i have chosen Single Server instead of Flexible Server option to reduce cost since in flexible server the cost is 2x for both compute and storage.

For Azure Service Bus, i have chosed the standard tier to accomodate for 13M ops per month and it has max message size of 256kb, which are sufficient for given requirements.




The architecture works as follows: 

1.An administrator fills out a web form that contains the message they wish to send to conference participants.

2.This kicks off two things: the database is updated and a message is placed on the queue

3.The service bus trigger function app is constantly listening to the queue.

4.When there is a message on the queue, the function app picks it up and sends emails to participants.

5.The database is then updated to inform that messages were sent out successfully.

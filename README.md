# Covid-19 Vaccine Appointment Notification System

Twilio Serverless Function to monitor vaccine appointment availability changes by querying [MA Covid Vaccines](https://www.macovidvaccines.com/) scraper endpoint.

## Overview

![Service Diagram](assets/services.png)

### Process

### 1. Cron Scheduler
Per a cron schedule, cronhooks makes a call to a webhook hosted by Twilio serverless functions.

### 2. Twilio Serverless Function Invocation
Twilio function is invocated and a script runs to:
1. Check MA Covid Vaccines for the latest status
2. Query Airtable for list of users
3. For each user, filter availabilities based on their preferences.
4. If there are availabilities, trigger a phone & email notification if enough time has passed since we last notified them (based on their preference). 
    - Twilio API to Call
    - SendGrid API to Email
5. Update Airtable DB with new timestamps for users if we notified them.

## "DB" Schema

User Object

| Column      | Airtable Field Name | Description |
| ----------- | ----------- | ----------- |
| User Name      | `Name`       | Name of user is used in the phone call speech script |
| User Email   | `Email`        | SendGrid recipient email destination |
| Phone Number   | `Phone Number`        | Twilio calls this number to notify |
| Last Call   | `Last Call`        | DateTime of when we last called the user's phone number |
| Last Email   | `Last Email`        | DateTime of when we last emailed the user |
| Call Time Threshold   | `Min Minutes Between Calls`        | User preference for how many minutes should pass before we call them again. (We can check for available updates every 5 minutes and call a user if something becomes available but don't call them again until an hour later, if this value is set to `60`. This prevents repeated calls every time the cron job runs if there are consistently available appointments.) |
| Email Time Threshold   | `Min Minutes Between Emails`        | Similar to Call Time Threshold, but for emails. |
| Locations   | `Locations`        | Array of strings of the name of locations this user wants to monitor |

## External Services

### Twilio Serverless Functions
https://www.twilio.com/docs/runtime/functions

Twilio Functions are used to query the Vaccine Availability endpoint, gather a list of users, filter vaccine availabilities based on user preferences, and call out to external services (Twilio, SendGrid) to trigger notifications if availabilities open that match the user's clinic criteria and notification preferences.

We write the function in Node/Javascript as a "serverless function" and Twilio exposes an endpoint that can be used to trigger the function on-demand.

To get started, you will need to create a Twilio account and register a phone number to use for your service. 

Twilio Pricing
- $1/month for PAYG phone number
- [Voice PAYG Pricing](https://www.twilio.com/voice/pricing/us)
- [Runtime Pricing](https://www.twilio.com/runtime/pricing)
  > First 10,000 invocations per month are free

### Twilio SendGrid 
https://sendgrid.com/

SendGrid is used to send email notifications to users. Although SendGrid is now under Twilio, you need to create a separate account under SendGrid with a SendGrid API token. If you anticipate less than 100 emails/day, you can get by with just the SendGrid free tier.

[SendGrid Pricing (Free Plan)](https://sendgrid.com/pricing/)
> 100 emails/day free forever

### Airtable
https://airtable.com/

Airtable is used as a simple database that we can interface with over a REST API to manage users. This can be adapted to use Google Sheets or another db storage mechanism. Airtable provides a simple way to use a spreadsheet-like interface to maintain records and should be free if you anticipate <1,200 user records.

A sample Airtable "Base" schema can be found in [airtable_sample.csv](assets/airtable_sample.csv).

[Airtable Pricing (Free Plan)](https://airtable.com/pricing)
> Free Plan: 1,200 records per base

### cronhooks
https://cronhooks.io/

cronhooks.io is used as a recurring scheduling mechanism to trigger our Twilio function. Let's say we want to run our Twilio function every 5 mins to repeatedly check whether or not availabilities have opened up and if so, send users a notification. We'll need a mechanism to call the Twilio function endpoint on a recurring basis.

From the Twilio blog, I went with cronhooks.io from their recommended list of easy ways to schedule functions: https://www.twilio.com/blog/4-ways-to-schedule-node-code 

[cronhooks pricing (Basic Plan)](https://cronhooks.io/#pricing)
- $3/month Basic Plan gives you 50 webhooks and the ability to schedule a recurring webhook. The Free plan does not let you schedule a recurring webhook.

## Local Development

To develop and run the function locally, update the sample `.env` file with your service api keys and credentials. 

Ensure you have `npm` and `node` on your machine. Twilio Serverless production runs node 10.x so it would be better to use [nvm](https://github.com/nvm-sh/nvm) to ensure you're running 10.x locally to develop in line with how functions are executed in Twilio prod environments. 

### Start Twilio Serverless Toolkit Development Server
Once you run `npm i`, you can leverage the `twilio-run` dev dependency to run a local server that mimics the Twilio serverless function environment. You don't need the full `twilio-cli` tool for the purposes of this project.

Start the Twilio Development Server
```
node_modules/.bin/twilio-run
```

Use an API program ([Postman](https://www.postman.com/), [Insomnia](https://insomnia.rest/), etc.) or just a cURL command to trigger the function endpoint
```
curl --location --request GET 'http://localhost:3000/ma-notify'
```

The cURL command should just return a basic "Success" payload
```
$ curl --location --request GET 'http://localhost:3000/ma-notify'

Success%
```

The Twilio Dev Server should show a request with 200 OK Response
```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Twilio functions available:                                            │
│   └── /ma-notify | http://localhost:3000/ma-notify                       │
│                                                                          │
│   Twilio assets available:                                               │
│   ├── /airtable_sample.csv | http://localhost:3000/airtable_sample.csv   │
│   └── /covid-alert.xml | http://localhost:3000/covid-alert.xml           │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
200 GET /ma-notify │ Response Type text/plain; charset=utf-8
```

#### Debug Mode
Set the `DEBUG_MODE` environment variable to true if you want to send a notification to test, even if there are no availabilities. 
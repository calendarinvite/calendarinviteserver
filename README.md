# calendarinviteserver
The Calendar Invite Server
The Calendar Invite Server is backed by to pieces of Technology. The Front End and the Backend.

The Front End is show cased by a Free Application called Calendarsnack.com. It is a VUE APP. It connects to the calendar invite server which is a set of APIs and Workflows built on AWS Serverless.

The Calendar Event Data is injected into the AWS Serverless stack using a custom Lambda (Lambda 1 Machine). That machine ETL's the data by taking the data from a calendar invite that is sent to an AWS Email Box from a customers client.

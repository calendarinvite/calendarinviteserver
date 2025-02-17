# calendarinviteserver
Two pieces of technology in the Calendar Invite Server: the VUE.JS Front End and the AWS Backend.

The Front End is showcased by a Free Application called Calendarsnack.com. It is a VUE app that connects to the calendar invite server, a set of APIs and Workflows built on AWS Serverless for the Calendar Invite Server.

**CRUD Data Injection by the Calendar Client**
The Calendar Event Data is injected into the AWS Serverless stack using a custom Lambda (Lambda 1 Machine). That machine ETLs the data by taking it from a calendar invite sent to an AWS Email Box from a customer client and assigns a UID attached to the Email Name of Client which becomes the "Organizer". Anytime the organizer updates the Calendar Client the data is updated in the Calendar Invite Server and to anyone that has received the calendar invite from the Calendar Invite Server. 

**Calendar Invite Server - Onboarding Instructions Technical Overview**
Overview	
The Onboarding Instructions guide provides a structured approach for setting up and managing the Calendar Invite Server (CIS). This document details the steps for repository setup, AWS profile configuration, CI/CD pipeline integration, and code deployment.
AWS Services Utilized
●	AWS IAM Identity Center: Manages authentication and profile access.
●	AWS S3: Stores event data and email notifications.
●	AWS SNS & SQS: Handles messaging and event processing.
●	AWS Lambda: Processes event updates and interacts with the database.
●	Amazon DynamoDB: Stores event metadata.
●	AWS CodeBuild & GitHub Actions: Automates CI/CD workflows.
System Setup
1. Establish GitHub Code Repositories
To begin, set up the necessary GitHub repositories:
1.	Create a GitHub account and generate a GitHub secret for authentication.
2.	Create and upload the following repositories:
○	calendar-invite-cicd
○	calendar-invite-event-management
○	calendar-invite-dashboard
○	calendar-invite-shared-library
2. AWS Profile Configuration for Sceptre
1.	Log in to AWS IAM Identity Center.
Configure AWS CLI SSO:
 aws configure sso
2.	
○	Provide AWS IAM Identity Center start URL.
○	Set region to us-west-2.
○	Authenticate via browser and select the appropriate account (dev or prod).
○	Ensure IAM roles (AdministratorAccess) are properly assigned.
3.	Update profile names to match:
○	infra/config.yaml → calendar-invite-dev
○	prod/config.yaml → calendar-invite-prod
CI/CD Pipeline Integration
1. GitHub Actions and AWS CodeBuild Setup
●	Update GitHub references within CI/CD configurations:
○	Modify sceptre-launch.yaml to reference correct GitHub accounts and repositories.
○	Update .gitmodules and .github/workflows/main.yaml in relevant repositories.
2. Running Sceptre Deployments
To launch infrastructure components:
sceptre launch infra -y
Code Management and Deployment
1. Version Control & Branching Strategy
●	Use short-lived feature branches for development.
●	All changes must be committed through Pull Requests (PRs).
●	PRs trigger GitHub Actions for linting and testing.
2. Deployment Workflow
To deploy a new application version:
git tag -am "Release vX.x.x" X.x.x
git push origin X.x.x

This triggers AWS CodeBuild, updating the AWS Serverless Application Repository.
Repository Structure
Repository	Description
.aws/ and .github/	CI/CD configuration files.
calendar-invite-cicd/	Core infrastructure and deployment configurations.
calendar-invite-dashboard/	Dashboard UI and supporting services.
calendar-invite-event-management/	Event processing and calendar logic.
calendar-invite-shared-library/	Common shared functions and utilities.
Event Processing Flow
1.	Email Handling:
○	SES receives an email and stores it in S3.
2.	Event Notification:
○	SNS triggers a SQS queue, passing event data.
3.	Lambda Processing:
○	Lambda retrieves SQS messages and updates DynamoDB.
○	Triggers notifications via SES.
Security Considerations
●	IAM Role Restrictions: Access is limited to authorized roles.
●	Data Encryption: Ensures secure data transmission and storage.
●	Access Control: Tenant-based filtering enforces data isolation.
Summary
The Onboarding Instructions guide provides an efficient and scalable method for configuring and managing the Calendar Invite Server (CIS). A structured approach enables seamless integration into AWS environments, ensuring robust event processing and automated CI/CD deployment.

**Verify New Event Invite Request - Technical Overview**
Overview
The Verify New Event Invite Request function in the Calendar Invite Server (CIS) ensures that event invitations are authorized before they are sent to attendees. It validates organizer status, invite limits, and potential restrictions to prevent unauthorized or excessive event invites.
AWS Services Utilized
●	AWS Lambda: Handles invite verification processing.
●	Amazon SQS (Simple Queue Service): Manages event invite verification requests.
●	Amazon DynamoDB: Stores event and organizer data for validation.
●	Amazon SNS (Simple Notification Service): Publishes notifications for verified invites.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event notification.
2.	Retrieve Event Details:
○	Queries DynamoDB to fetch event and organizer information.
3.	Validation Checks:
○	Ensures the invitee has not blocked the organizer.
○	Verifies that the organizer account is not suspended.
○	Checks bulk invite authorization and invite limits.
4.	Approve or Deny Invite:
○	If all validations pass, the request is approved and published to SNS.
○	If any validation fails, the request is denied.
5.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Iterates through each record in the SQS event.
●	Calls process_request_from to validate the event invite request.
process_request_from(record)
●	Extract event invite details.
●	Queries DynamoDB to fetch event information.
●	Runs validation checks to determine whether the invite should be sent.
●	Deletes the processed message from SQS.
verified_to_send(request, event)
●	Checks if the invitee has blocked the organizer.
●	Verifies that the organizer's account is active.
●	Ensures bulk sending requests meet authorization and invite limits.
attendee_has_blocked(organizer, attendee)
●	Queries DynamoDB to check if an attendee has blocked the organizer.
account_is_suspended_for(organizer)
●	Checks if the organizer's account is marked as suspended in DynamoDB.
bulk_validation_failed(origin, organizer, event)
●	Ensures bulk invite requests comply with organizer permissions and event invite limits.
send_notification_of_verified(request, event)
●	Publishes an SNS message to approve and process the event invite.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent retries.
3. Data Handling
DynamoDB Records
●	Event Record:
○	pk: event#<uid>
○	Stores event metadata including invite limits and organizer details.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	DYNAMODB_TABLE: Name of the DynamoDB table.
●	SQS_URL: URL of the SQS queue for processing event invite verification.
●	NEW_EVENT_INVITE: SNS topic ARN for approved invites.
Error Handling
●	Invalid Invite Requests: If a required field is missing, the request is logged and skipped.
●	Authorization Failures: The request is denied and logged if an organizer or invitee is restricted.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Verify New Event Invite Request function ensures that event invites are properly validated before sending. By leveraging AWS Lambda, SQS, DynamoDB, and SNS, this function prevents unauthorized or excessive invite sending while maintaining event integrity.

**Update Event Attendee Record - Technical Overview**
Overview
The Update Event Attendee Record function in the Calendar Invite Server (CIS) updates attendee records based on RSVP responses. It ensures that event participation status is accurately maintained and statistics are updated accordingly.
AWS Services Utilized
●	AWS Lambda: Handles RSVP response processing.
●	Amazon SQS (Simple Queue Service): Manages attendee update requests.
●	Amazon DynamoDB: Stores and updates attendee records.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event notification.
2.	Retrieve Attendee Record:
○	Queries DynamoDB to fetch existing attendee details.
3.	Determine Update Action:
○	If the RSVP status has changed, update the record.
○	If the attendee does not exist, create a new record.
4.	Update Event Statistics:
○	Modify event and organizer statistics based on the new RSVP status.
5.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Iterates through each record in the SQS event.
●	Calls process_request_from to update the attendee record.
process_request_from(record)
●	Extract RSVP details from the request.
●	Queries DynamoDB to fetch the attendee record.
●	Updates the attendee record and event statistics if necessary.
●	Deletes the processed message from SQS.
update_event_attendee_record_with(event_reply)
●	Check if the attendee’s RSVP status has changed.
●	Updates the record if a change is detected.
●	Creates a new attendee record if none exists.
get_current_attendee_record_for(event_reply)
●	Queries DynamoDB to retrieve the attendee’s current status.
submit_updated_event_attend_record_for(event_reply, previous_status)
●	Updates the attendee record and modifies event statistics accordingly.
submit_shared_event_attend_record_for(event_reply)
●	Creates a new attendee record if the RSVP response is from a shared invitation.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent retries.
3. Data Handling
DynamoDB Records
●	Attendee Record:
○	pk: event#<uid>
○	sk: attendee#<email>
○	Stores attendee RSVP status and history.
●	Event Statistics:
○	Tracks the number of RSVP responses by status.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	DYNAMODB_TABLE: Name of the DynamoDB table.
●	SQS_URL: URL of the SQS queue for processing attendee updates.
Error Handling
●	Invalid RSVP Requests: The request is logged and skipped if a required field is missing.
●	DynamoDB Update Failures: Logged for troubleshooting but do not halt execution.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Update Event Attendee Record function ensures that RSVP responses are accurately reflected in the event database. By leveraging AWS Lambda, SQS, and DynamoDB, this function automates the update process while maintaining data integrity.

**Update Event - Technical Overview**
Overview
The Update Event function in the Calendar Invite Server (CIS) modifies existing event details, ensures that updated event data is recorded in the system, and sends notifications to affected attendees.
AWS Services Utilized
●	AWS Lambda: Handles event update processing.
●	Amazon SQS (Simple Queue Service): Manages event update requests.
●	Amazon DynamoDB: Stores and updates event records.
●	Amazon SNS (Simple Notification Service): Publishes update notifications.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event notification.
2.	Retrieve Existing Event:
○	Queries DynamoDB to fetch the original event details.
3.	Compare and Update:
○	Check if any event details have changed.
○	Updates the event record if modifications exist.
4.	Send Notifications:
○	Publishes update requests to SNS.
5.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Iterates through each record in the SQS event.
●	Calls process_request_from to update the event record.
process_request_from(record)
●	Extracts updated details from the request.
●	Queries DynamoDB to fetch the original event.
●	Updates the event if necessary.
●	Publishes an SNS notification.
●	Deletes the processed message from SQS.
get_event_uid_from(uid)
●	Queries DynamoDB to retrieve the original event record.
event_updated(original_event, new_event)
●	Compares the new and original event to determine if an update is needed.
update_event(original_event, new_event)
●	Updates the event record in DynamoDB.
●	Increments the event sequence number.
send_notification_of_updated(event)
●	Publishes an SNS message to notify attendees of the update.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent retries.
3. Data Handling
DynamoDB Records
●	Event Record:
○	pk: event#<uid>
○	sk: event#<uid>
○	Stores event metadata, including timestamps, status, and organizer details.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	DYNAMODB_TABLE: Name of the DynamoDB table.
●	SQS_URL: URL of the SQS queue for processing event updates.
●	EVENT_UPDATED: SNS topic ARN for update notifications.
Error Handling
●	Invalid Update Requests: The request is logged and skipped if a required field is missing.
●	DynamoDB Update Failures: Logged for troubleshooting but do not halt execution.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Update Event function ensures that changes to event details are recorded and communicated efficiently. This function automates the update process by leveraging AWS Lambda, SQS, DynamoDB, and SNS while maintaining data integrity and attendee awareness.

**Stage Attendees for Updated Event - Technical Overview Overview**
The Stage Attendees for Updated Event function in the Calendar Invite Server (CIS) is responsible for identifying all attendees of an updated event and queuing update notifications for processing. This ensures that all invitees receive proper notifications when event details change.
AWS Services Utilized
●	AWS Lambda: Handles event attendee staging.
●	Amazon SQS (Simple Queue Service): Manages event update requests.
●	Amazon DynamoDB: Retrieves attendee records for a specific event.
●	Amazon SNS (Simple Notification Service): Publishes update notifications.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event notification.
2.	Retrieve Attendee List:
○	Queries DynamoDB to fetch attendees of the updated event.
3.	Queue Updates:
○	Format attendee details for notification.
○	Publishes update requests to SNS.
4.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Iterates through each record in the SQS event.
●	Calls process_request_from to stage attendee update requests.
process_request_from(record)
●	Extract event update details.
●	Queries attendee records from DynamoDB.
●	Queues updates for all affected attendees.
●	Deletes the processed message from SQS.
get_attendee_list(uid)
●	Queries DynamoDB to retrieve all attendees for the updated event.
queue_updated_event_for_processing(event, attendees)
●	Publishes an SNS message for each attendee to notify them of the update.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent retries.
3. Data Handling
DynamoDB Records
●	Attendee Record:
○	pk: event#<uid>
○	sk: attendee#<email>
○	Stores attendee information and participation status.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	DYNAMODB_TABLE: Name of the DynamoDB table.
●	SQS_URL: URL of the SQS queue for processing event updates.
●	EVENT_UPDATE_REQUEST: SNS topic ARN for update notifications.
Error Handling
●	Missing Attendee Records: If no attendees are found, the function logs the issue and exits.
●	SNS Publishing Failures: Logged for troubleshooting but do not halt execution.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Stage Attendees for Updated Event function ensures that all affected attendees are notified when an event is updated. By leveraging AWS Lambda, SQS, DynamoDB, and SNS, this function automates the update notification process for efficiency and reliability.

**Stage Attendees for Cancelled Event - Technical Overview**
Overview
The Stage Attendees for Cancelled Event function in the Calendar Invite Server (CIS) is responsible for identifying all attendees of a canceled event and queuing cancellation notifications for processing. This ensures that all invitees receive proper cancellation notifications in a structured and automated manner.
AWS Services Utilized
●	AWS Lambda: Handles event attendee staging.
●	Amazon SQS (Simple Queue Service): Manages event cancellation requests.
●	Amazon DynamoDB: Retrieves attendee records for a specific event.
●	Amazon SNS (Simple Notification Service): Publishes cancellation notifications.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event notification.
2.	Retrieve Attendee List:
○	Queries DynamoDB to fetch attendees of the cancelled event.
3.	Queue Cancellations:
○	Formats attendee details for cancellation.
○	Publishes cancellation requests to SNS.
4.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Iterates through each record in the SQS event.
●	Calls process_request_from to stage attendee cancellation requests.
process_request_from(record)
●	Extracts event cancellation details.
●	Queries attendee records from DynamoDB.
●	Queues cancellations for all affected attendees.
●	Deletes the processed message from SQS.
get_attendee_list(uid)
●	Queries DynamoDB to retrieve all attendees for the cancelled event.
queue_cancellation_for_processing(event, attendees)
●	Publishes an SNS message for each attendee to notify them of the cancellation.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent retries.
3. Data Handling
DynamoDB Records
●	Attendee Record:
○	pk: event#<uid>
○	sk: attendee#<email>
○	Stores attendee information and participation status.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	DYNAMODB_TABLE: Name of the DynamoDB table.
●	SQS_URL: URL of the SQS queue for processing event cancellations.
●	EVENT_CANCELLATION_REQUEST: SNS topic ARN for cancellation notifications.
Error Handling
●	Missing Attendee Records: If no attendees are found, the function logs the issue and exits.
●	SNS Publishing Failures: Logged for troubleshooting but do not halt execution.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Stage Attendees for Cancelled Event function ensures that all affected attendees are notified when an event is cancelled. By leveraging AWS Lambda, SQS, DynamoDB, and SNS, this function automates the cancellation notification process for efficiency and reliability.

**Send Event Invite - Technical Overview**
Overview
The Send Event Invite function in the Calendar Invite Server (CIS) processes and sends calendar invitations to attendees. It ensures that each invitee receives a properly formatted iCal invite while tracking event statistics.
AWS Services Utilized
●	AWS Lambda: Handles event invite processing.
●	Amazon SES (Simple Email Service): Sends calendar invites to attendees.
●	Amazon SQS (Simple Queue Service): Manages event invite requests.
●	Amazon DynamoDB: Stores attendee records and event statistics.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event notification.
2.	Extract Invite Details:
○	Retrieves invite data from the SQS message.
○	Checks if the attendee has already been sent an invite.
3.	Send Invite:
○	Generates a properly formatted iCal invitation.
○	Sends the invite via SES.
4.	Update Event Records:
○	Stores invitee details in DynamoDB.
○	Updates invite statistics for the event.
5.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Iterates through each record in the SQS event.
●	Calls process_request_from to handle each invitation.
process_request_from(record)
●	Extracts invitee details.
●	Checks if the invitee has already been sent an invite.
●	Sends an invite if necessary.
●	Deletes the processed message from SQS.
attendee_has_not_been_sent(event_invite)
●	Checks DynamoDB to verify if the attendee has already been invited.
send_invite_to_attendee(invite)
●	Formats and sends an iCal invite via SES.
get_ical_for(invite)
●	Generates an iCal formatted invite.
create_record_of_attendee(invite)
●	Stores the invitee’s information in DynamoDB.
●	Updates event statistics and invite limits.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent retries.
3. Data Handling
DynamoDB Records
●	Attendee Record:
○	pk: event#<uid>
○	sk: attendee#<email>
○	Stores attendee information, RSVP status, and history.
●	Event Statistics:
○	Tracks the number of invites sent, RSVP responses, and other key metrics.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	DYNAMODB_TABLE: Name of the DynamoDB table.
●	SENDER: Email address used for sending invites.
●	RSVP_EMAIL: Email used for collecting RSVP responses.
●	SQS_URL: URL of the SQS queue for processing event invites.
Error Handling
●	Duplicate Invites: If an invitee has already been sent an invite, no duplicate is sent.
●	Email Sending Failures: Logged for troubleshooting but do not halt execution.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Send Event Invite function enables efficient handling of event invitations. It integrates with AWS Lambda, SES, SQS, and DynamoDB to send calendar invites, track invitations, and update event statistics.

**Send Event Update - Technical Overview**
Overview
The Send Event Update function in the Calendar Invite Server (CIS) is responsible for processing and sending updated calendar invitations to attendees. It ensures that attendees receive properly formatted iCal updates when event details change.
AWS Services Utilized
●	AWS Lambda: Handles event update processing.
●	Amazon SES (Simple Email Service): Sends calendar update emails to attendees.
●	Amazon SQS (Simple Queue Service): Manages event update requests.
●	Amazon DynamoDB: Stores attendee records and event update history.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event notification.
2.	Extract Update Details:
○	Retrieves event update data from the SQS message.
3.	Send Update:
○	Generates a properly formatted iCal update.
○	Sends the update via SES.
4.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Iterates through each record in the SQS event.
●	Calls process_request_from to handle each update request.
process_request_from(record)
●	Extracts event update details.
●	Sends the update notice to the attendee.
●	Deletes the processed message from SQS.
send_update(request)
●	Formats and sends an iCal update via SES.
●	Uses the subject CalendarSnack Event Updated: <event summary>.
get_ical(request)
●	Generates an iCal formatted event update.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent retries.
3. Data Handling
DynamoDB Records
●	Attendee Record:
○	pk: event#<uid>
○	sk: attendee#<email>
○	Stores attendee information and update history.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	DYNAMODB_TABLE: Name of the DynamoDB table.
●	SENDER: Email address used for sending updates.
●	RSVP_EMAIL: Email used for collecting RSVP responses.
●	SQS_URL: URL of the SQS queue for processing event updates.
Error Handling
●	Invalid Update Requests: The request is logged and skipped if a required field is missing.
●	Email Sending Failures: Logged for troubleshooting but do not halt execution.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Send Event Update function ensures attendees receive timely and accurate event updates. AWS Lambda, SES, SQS, and DynamoDB work together to deliver and track event modifications efficiently.

**Send Event Cancellation - Technical Overview**
Overview
The Send Event Cancellation function in the Calendar Invite Server (CIS) notifies attendees about canceled events. It ensures cancellation emails are sent in iCal format and updates the event status accordingly.
AWS Services Utilized
●	AWS Lambda: Handles event cancellation processing.
●	Amazon SES (Simple Email Service): Sends cancellation emails to attendees.
●	Amazon SQS (Simple Queue Service): Manages cancellation requests.
●	Amazon DynamoDB: Stores attendee and event records.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event notification.
2.	Extract Event Details:
○	Retrieves cancellation request details from the SQS message.
3.	Send Cancellation:
○	Generates an iCal formatted cancellation notice.
○	Sends the cancellation email via SES.
4.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Iterates through each record in the SQS event.
●	Calls process_request_from to handle each cancellation request.
process_request_from(record)
●	Extracts cancellation details.
●	Sends the cancellation notice to the attendee.
●	Deletes the processed message from SQS.
send_cancellation(request)
●	Formats and sends an iCal cancellation via SES.
●	Uses the subject CalendarSnack Event Cancelled: <event summary>.
get_ical(request)
●	Generates an iCal formatted cancellation invite.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent retries.
3. Data Handling
DynamoDB Records
●	Attendee Record:
○	pk: event#<uid>
○	sk: attendee#<email>
○	Stores attendee information and cancellation status.

Environmental Variables Used
●	REGION: AWS Region for execution.
●	DYNAMODB_TABLE: Name of the DynamoDB table.
●	SENDER: Email address used for sending cancellations.
●	RSVP_EMAIL: Email used for collecting RSVP responses.
●	SQS_URL: URL of the SQS queue for processing event cancellations.
Error Handling
●	Invalid Cancellation Requests: If a required field is missing, the request is logged and skipped.
●	Email Sending Failures: Logged for troubleshooting but do not halt execution.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Send Event Cancellation function enables seamless processing of event cancellations, ensuring attendees are promptly notified. AWS Lambda, SES, SQS, and DynamoDB work together to deliver and track cancellation notifications efficiently.

**Send Bulk Event Invites - Technical Overview**
Overview
The Send Bulk Event Invites function in the Calendar Invite Server (CIS) processes bulk event invitations. It ensures that attendees receive properly formatted calendar invites while tracking invite history and updating event statistics.
AWS Services Utilized
●	AWS Lambda: Handles bulk invite processing.
●	Amazon SES (Simple Email Service): Sends calendar invites to attendees.
●	Amazon SQS (Simple Queue Service): Manages bulk invite requests.
●	Amazon DynamoDB: Stores attendee records and event statistics.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event notification.
2.	Extract Invite Details:
○	Retrieves invite data from the SQS message.
○	Checks if the attendee has already been sent an invite.
3.	Send Invite:
○	Generates a properly formatted iCal invitation.
○	Sends the invite via SES.
4.	Update Event Records:
○	Stores invitee details in DynamoDB.
○	Updates invite statistics for the event.
5.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Iterates through each record in the SQS event.
●	Calls process_request_from to handle each invitation.
process_request_from(record)
●	Extracts invitee details.
●	Checks if the invitee has already been sent an invite.
●	Sends an invite if necessary.
●	Deletes the processed message from SQS.
attendee_has_not_been_sent(event_invite)
●	Checks DynamoDB to verify if the attendee has already been invited.
send_invite_to_attendee(invite)
●	Formats and sends an iCal invite via SES.
get_ical_for(invite)
●	Generates an iCal formatted invite.
create_record_of_attendee(invite)
●	Stores the invitee’s information in DynamoDB.
●	Updates event statistics and invite limits.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent retries.
3. Data Handling
DynamoDB Records
●	Attendee Record:
○	pk: event#<uid>
○	sk: attendee#<email>
○	Stores attendee information, RSVP status, and history.
●	Event Statistics:
○	Tracks the number of invites sent, RSVP responses, and other key metrics.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	DYNAMODB_TABLE: Name of the DynamoDB table.
●	SENDER: Email address used for sending invites.
●	RSVP_EMAIL: Email used for collecting RSVP responses.
●	SQS_URL: URL of the SQS queue for processing event invites.
Error Handling
●	Duplicate Invites: If an invitee has already been sent an invite, no duplicate is sent.
●	Email Sending Failures: Logged for troubleshooting but do not halt execution.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Send Bulk Event Invites function enables efficient handling of large-scale event invitations. It integrates with AWS Lambda, SES, SQS, and DynamoDB to send calendar invites, track invitations, and update event statistics.

**Notify Organizer of Successful Event Creation - Technical Overview**
Overview
The Notify Organizer of Successful Event Creation function in the Calendar Invite Server (CIS) informs event organizers when their event has been successfully created. This notification ensures transparency and confirms that the event has been properly registered in the system.
AWS Services Utilized
●	AWS Lambda: Handles event processing and notification dispatch.
●	Amazon SES (Simple Email Service): Sends confirmation emails to organizers.
●	Amazon SQS (Simple Queue Service): Manages event creation messages in the queue.
●	AWS CodeCommit: Stores the email notification template.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event notification.
2.	Extract Event Details:
○	Retrieves event details from the SQS message.
○	Loads the email template from AWS CodeCommit.
3.	Send Notification:
○	Sends a confirmation email via SES to notify the organizer.
4.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Iterates through each record in the SQS event.
●	Calls process_request_from to handle the event notification.
process_request_from(record)
●	Extracts event details from the message.
●	Loads the email template from CodeCommit.
●	Sends the email notification to the organizer.
●	Deletes the processed message from SQS.
get_event_notification_email_template(event)
●	Retrieves the email template from AWS CodeCommit.
●	Dynamically replaces placeholders ({mailto}, {summary}, {uid}, {events}) with event-specific details.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent retries.
3. Data Handling
Email Notification Format
●	Recipient: The email address of the event organizer (mailto field).
●	Template: Stored in AWS CodeCommit.
●	Subject: Defined by SUBJECT environmental variable.
●	Sender: Defined by SENDER environmental variable.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	SUBJECT: Subject line for the email notification.
●	SENDER: Sender email address for notifications.
●	NEW_EVENT_NOTIFICATION_EMAIL: Path to the email template in CodeCommit.
●	CODECOMMIT_REPO: Name of the CodeCommit repository storing the template.
●	SQS_URL: URL of the SQS queue for event notifications.
Error Handling
●	Failed Email Notifications: Logged for troubleshooting, but failures do not impact CIS functionality.
●	Missing Email Templates: If the template is unavailable, an error is logged and the message remains in SQS for retry.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Notify Organizer of Successful Event Creation function ensures that organizers receive confirmation when an event is successfully created. This improves system transparency and user confidence. AWS Lambda, SES, SQS, and CodeCommit work together to deliver these notifications efficiently.

**Notify Organizer of Successful Enrollment - Technical Overview**
Overview
The Notify Organizer of Successful Enrollment function was created to process a Shopify purchase/order. While not a core function of the Calendar Invite Server (CIS), it provides an automated notification mechanism when an organizer successfully enrolls via an external system such as Shopify.
AWS Services Utilized
●	AWS Lambda: Handles event processing and notification dispatch.
●	Amazon SES (Simple Email Service): Sends confirmation emails to organizers.
●	Amazon SQS (Simple Queue Service): Manages successful enrollment messages in the queue.
●	AWS CodeCommit: Stores the email notification template.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event notification.
2.	Extract Event Details:
○	Retrieves order details from the SQS message.
○	Loads the email template from AWS CodeCommit.
3.	Send Notification:
○	Send a confirmation email via SES to notify the organizer.
4.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Iterates through each record in the SQS event.
●	Calls process_request to handle the event notification.
process_request(record)
●	Extract order details from the message.
●	Loads the email template from CodeCommit.
●	Send the email notification to the organizer.
●	Deletes the processed message from SQS.
get_event_notification_template()
●	Retrieves the email template from AWS CodeCommit.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent retries.
3. Data Handling
Email Notification Format
●	Recipient: The email address of the organizer (mailto field).
●	Template: Stored in AWS CodeCommit.
●	Subject: Defined by SUBJECT environmental variable.
●	Sender: Defined by SENDER environmental variable.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	SUBJECT: Subject line for the email notification.
●	SENDER: Sender email address for notifications.
●	SUCCESSFUL_ENROLLMENT_NOTIFICATION_EMAIL: Path to the email template in CodeCommit.
●	CODECOMMIT_REPO: Name of the CodeCommit repository storing the template.
●	SQS_URL: URL of the SQS queue for successful enrollment notifications.
Error Handling
●	Failed Email Notifications: Logged for troubleshooting, but failures do not impact CIS functionality.
●	Missing Email Templates: If the template is unavailable, an error is logged and the message remains in SQS for retry.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Notify Organizer of Successful Enrollment function provides an automated notification mechanism for external integrations such as Shopify. While not a critical component of CIS, it enhances the user experience by confirming successful enrollments. AWS Lambda, SES, SQS, and CodeCommit are used to deliver these notifications efficiently.

**Notify Organizer of Failed Event Creation - Technical Overview**
Overview
The Notify Organizer of Failed Event Creation function in the Calendar Invite Server (CIS) informs event organizers when an event creation attempt fails. This notification allows organizers to take corrective action and ensures awareness of unsuccessful event submissions.
AWS Services Utilized
●	AWS Lambda: Handles event processing and notification dispatch.
●	Amazon SES (Simple Email Service): Sends failure notifications to organizers.
●	Amazon SQS (Simple Queue Service): Manages failed event creation messages in the queue.
●	AWS CodeCommit: Stores the email notification template.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event notification.
2.	Extract Event Details:
○	Retrieves event details from the SQS message.
○	Loads the email template from AWS CodeCommit.
3.	Send Notification:
○	Sends an email via SES to notify the organizer.
4.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Iterates through each record in the SQS event.
●	Calls process_request_from to handle the event notification.
process_request_from(record)
●	Extracts event details from the message.
●	Loads the email template from CodeCommit.
●	Sends the email notification to the organizer.
●	Deletes the processed message from SQS.
get_event_notification_template()
●	Retrieves the email template from AWS CodeCommit.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent retries.
3. Data Handling
Email Notification Format
●	Recipient: The email address of the event organizer (mailto field).
●	Template: Stored in AWS CodeCommit.
●	Subject: Defined by SUBJECT environmental variable.
●	Sender: Defined by SENDER environmental variable.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	SUBJECT: Subject line for the email notification.
●	SENDER: Sender email address for notifications.
●	FAILED_EVENT_NOTIFICATION_EMAIL: Path to the email template in CodeCommit.
●	CODECOMMIT_REPO: Name of the CodeCommit repository storing the template.
●	SQS_URL: URL of the SQS queue for failed event notifications.
Error Handling
●	Failed Email Notifications: Logged for troubleshooting, but failures do not impact CIS functionality.
●	Missing Email Templates: If the template is unavailable, an error is logged and the message remains in SQS for retry.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Notify Organizer of Failed Event Creation function ensures that organizers are promptly notified when an event creation fails. While not a critical component of CIS, this function enhances system transparency by providing actionable failure notifications. AWS Lambda, SES, SQS, and CodeCommit are leveraged to deliver these notifications efficiently.

**Notify Organizer of Event Limit Reached - Technical Overview**
Overview
The Notify Organizer of Event Limit Reached function in the Calendar Invite Server (CIS) informs event organizers when they have reached their allowed event creation limit. While this function is not currently core to CIS's operation, it serves as an optional feature for managing organizer event quotas.
AWS Services Utilized
●	AWS Lambda: Handles event processing and sends notifications.
●	Amazon SES (Simple Email Service): Sends event limit notification emails.
●	Amazon SQS (Simple Queue Service): Manages event notifications in the queue.
●	AWS CodeCommit: Stores the email notification template.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event notification.
2.	Extract Event Details:
○	Retrieves event details from the SQS message.
○	Loads the email template from AWS CodeCommit.
3.	Send Notification:
○	Sends an email via SES to notify the organizer.
4.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Iterates through each record in the SQS event.
●	Calls process_request_from to handle the event notification.
process_request_from(record)
●	Extracts event details from the message.
●	Loads the email template from CodeCommit.
●	Sends the email notification to the organizer.
●	Deletes the processed message from SQS.
get_event_notification_email_template()
●	Retrieves the email template from AWS CodeCommit.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent retries.
3. Data Handling
Email Notification Format
●	Recipient: The email address of the event organizer (mailto field).
●	Template: Stored in AWS CodeCommit.
●	Subject: Defined by SUBJECT environmental variable.
●	Sender: Defined by SENDER environmental variable.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	SUBJECT: Subject line for the email notification.
●	SENDER: Sender email address for notifications.
●	EVENT_LIMIT_REACHED_NOTIFICATION_EMAIL: Path to the email template in CodeCommit.
●	CODECOMMIT_REPO: Name of the CodeCommit repository storing the template.
●	SQS_URL: URL of the SQS queue for event notifications.
Error Handling
●	Failed Email Notifications: Logged for troubleshooting, but failures do not impact CIS functionality.
●	Missing Email Templates: If the template is unavailable, an error is logged and the message remains in SQS for retry.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Notify Organizer of Event Limit Reached function provides an optional notification mechanism for event organizers when they hit their event creation limit. While not a critical component of CIS, it enhances event management by ensuring organizers are aware of their usage constraints. This function relies on AWS Lambda, SES, SQS, and CodeCommit to deliver notifications efficiently.

**Get New Event Request from Email - Technical Overview**
Overview
The Get New Event Request from Email function in the Calendar Invite Server (CIS) processes event creation, updates, and cancellations received via email. It extracts event details, determines the request type, and triggers the appropriate event processing flow using AWS services.
AWS Services Utilized
●	AWS Lambda: Handles the processing of email-based event requests.
●	Amazon S3: Stores email content for processing.
●	Amazon SNS (Simple Notification Service): Sends notifications for event processing (creation, update, cancellation).
●	Amazon SQS (Simple Queue Service): Manages event request messages in the queue.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event linked to an S3 upload notification.
2.	Extract Email Data:
○	Retrieves the email file from S3.
○	Extracts event request details using the iCal format.
3.	Process Event Request:
○	Determines the request method (request, update, cancel).
○	Routes the request to the appropriate SNS topic for further processing.
4.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Iterates through each record in the SQS event.
●	Calls process_request_from to extract and process event requests.
process_request_from(record)
●	Extracts email content and event UID.
●	Determines the event request type and routes it accordingly.
●	Deletes successfully processed messages from SQS.
extract_event_request_from(email, event_request_uid)
●	Uses the iCal library to parse event request details from the email.
●	Standardizes event field formats.
process_event_request_by_method(event_request)
●	Routes the request based on its method (request, update, cancel).
event_request_is_valid(event_request)
●	Checks if the method is one of the approved request types.
cancel_event_for(event_request)
●	Publishes a SNS message for event cancellation processing.
create_event_record_for(event_request)
●	Publishes a SNS message for new event creation.
update_event_for(event_request)
●	Publishes a SNS message for updating an existing event.
notify_user_of_request_failure_for(event_request)
●	Sends an SNS notification if the request is invalid or fails.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent retries.
3. Data Handling
Event Request Data Format
●	Source: Extracted from an email stored in S3.
●	Request Methods Supported:
○	request: Creates a new event.
○	update: Updates an existing event.
○	cancel: Cancels an event.
DynamoDB Records (Referenced but Not Updated in This Function)
●	Event UID Lookup:
○	Used for identifying the corresponding event.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	NEW_EVENT_REQUEST: SNS topic ARN for new event requests.
●	EVENT_UPDATE: SNS topic ARN for event updates.
●	EVENT_CANCELLATION: SNS topic ARN for event cancellations.
●	FAILED_EVENT_CREATE: SNS topic ARN for failed event processing.
●	SQS_URL: URL of the SQS queue for processing event requests.
Error Handling
●	Invalid Event Requests: If the request does not contain a valid method, an SNS notification is sent to the requester.
●	Processing Failures: Errors are logged, and failed tasks remain in SQS for retry.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Get New Event Request from Email function ensures that event requests received via email are extracted, validated, and processed efficiently. By leveraging AWS Lambda, S3, SQS, and SNS, the function enables automated handling of event creation, updates, and cancellations.

**Get New Event Reply from Email - Technical Overview**
Overview
The Get New Event Reply from Email function in the Calendar Invite Server (CIS) processes RSVP responses received via email. It extracts the event details from the email, formats the response, and sends notifications to update the event's status.
AWS Services Utilized
●	AWS Lambda: Processes incoming email event replies.
●	Amazon S3: Stores email content for extraction.
●	Amazon SNS (Simple Notification Service): Sends notifications for event reply updates.
●	Amazon SQS (Simple Queue Service): Manages email event reply messages in the queue.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event linked to an S3 upload notification.
2.	Extract Email Data:
○	Retrieves the email file from S3.
○	Extracts event reply details using the iCal format.
3.	Process RSVP Response:
○	Checks if the reply contains a valid event method.
○	Formats the event reply data.
○	Publishes a notification to SNS to update the event status.
4.	Cleanup:
○	Deletes successfully processed messages from SQS.
2. Key Functions
lambda_handler(event, _)
●	Processes each record in the SQS event.
●	Calls process_request_from to handle event replies.
process_request_from(record)
●	Extracts email content and event UID.
●	Parses the RSVP response using the iCal library.
●	Sends the formatted response as a notification.
●	Removes successfully processed messages from SQS.
get_email_from(record)
●	Retrieves the S3 file location from the SQS event.
●	Extracts email content from the file stored in S3.
extract_event_request_from(email, event_request_uid)
●	Uses the iCal library to parse event reply details from the email.
send_notification_of_new(event_reply)
●	Publishes an SNS message containing the formatted event reply.
format_reply_request_from(event_reply)
●	Extracts and formats key RSVP details:
○	attendee: The email of the responding attendee.
○	dtstamp: Timestamp of the response in epoch format.
○	partstat: RSVP status (e.g., accepted, declined, tentative).
○	uid: Unique event identifier.
○	prodid: Product identifier from the iCal response.
get_epoch_from(timestamp)
●	Converts an iCal datetime string into an epoch timestamp.
delete_successfully_processed_sqs(message)
●	Removes successfully processed messages from SQS to prevent duplicate processing.
3. Data Handling
Event Reply Data Format
●	Source: Extracted from an email stored in S3.
●	Required Fields:
○	mailto_rsvp: Attendee’s email address.
○	dtstamp: Timestamp of response.
○	partstat: RSVP status (accepted, declined, tentative).
○	uid: Event identifier.
DynamoDB Records (Referenced but Not Updated in This Function)
●	Event UID Lookup:
○	Used for identifying the corresponding event.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	NEW_EVENT_REPLY: SNS topic ARN for event reply notifications.
●	SQS_URL: URL of the SQS queue for processing event replies.
Error Handling
●	Invalid RSVP Data: If the reply does not contain a valid method, it is ignored.
●	Processing Failures: Errors are logged, and failed tasks remain in SQS for retry.
●	Exception Logging: Logs errors to facilitate debugging.
Summary
The Get New Event Reply from Email function ensures that RSVP responses received via email are extracted, formatted, and notified efficiently. By utilizing AWS Lambda, S3, SQS, and SNS, the function enables seamless RSVP tracking for calendar invites.

**Get New Bulk Event Invites from Email - Technical Overview**
Overview
The Get New Bulk Event Invites from Email function in the Calendar Invite Server (CIS) processes event invite requests sent via email. The function extracts attendee details from an attached CSV file, validates the sender's authorization, and queues valid invites for processing while notifying the sender of any issues.
AWS Services Utilized
●	AWS Lambda: Processes incoming email requests.
●	Amazon S3: Stores and retrieves CSV files containing attendee information.
●	Amazon DynamoDB: Manages organizer authorization and event records.
●	Amazon SNS (Simple Notification Service): Queues valid invites for sending.
●	Amazon SES (Simple Email Service): Sends status reports to the request sender.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger: The function is triggered by an SQS event linked to an S3 upload notification.
2.	Extract Email Data:
○	Retrieves the CSV file from S3.
○	Extracts attendee details from the CSV file.
○	Parses the sender's email address.
3.	Validate Sender:
○	Checks if the sender is an authorized bulk sender.
○	Verifies that the sender is the organizer of the specified event.
4.	Process Attendees:
○	Validates each invitee's email.
○	Groups valid and invalid invites.
5.	Queue Valid Invites:
○	Sends valid invites to an SNS topic for processing.
6.	Notify Sender:
○	Sends an SES email to the sender with a summary of accepted and rejected invites.
2. Key Functions
lambda_handler(event, _)
●	Extracts the S3 file location from the event.
●	Calls process_bulk_event_invites to handle the request.
process_bulk_event_invites(event)
●	Retrieves email and CSV file from S3.
●	Validates sender authorization.
●	Extracts and validates attendee information.
●	Queues valid invites and logs invalid ones.
●	Sends a status report to the sender.
get_s3_file_location_from_sns_notification(sns_notification)
●	Extracts the S3 bucket and file key from the notification.
get_s3_file_content(bucket, key)
●	Retrieves the file content from S3.
●	Decodes the file from UTF-8-SIG format.
get_email_request(s3_file)
●	Parses the email headers to extract sender information.
sender_authorized_to_send_bulk_invites(sender)
●	Queries DynamoDB to verify if the sender is an authorized bulk sender.
get_event_invites(sender, email_request)
●	Parses the CSV file for attendee information.
●	Validates each attendee’s email.
●	Categorizes valid and invalid invites.
queue_event_invites_for_send(event, event_invites)
●	Publishes valid invites to SNS for processing.
send_sender_status_of_bulk_request(sender, event_invites, invalid_invites)
●	Sends a summary email to the sender via SES.
3. Data Handling
CSV File Format
The attached CSV file should contain:
●	uid: Unique identifier for the event.
●	email: Attendee’s email address.
●	name: Attendee’s name.
DynamoDB Records
●	Organizer Authorization:
○	pk: organizer#<email>
○	sk: bulk#<email>
●	Event Record Lookup:
○	pk: event#<uid>
○	sk: event#<uid>
Environmental Variables Used
●	REGION: AWS Region for execution.
●	THIRTYONE_TABLE: DynamoDB table storing organizer and event records.
●	NEW_EVENT_INVITE: SNS topic ARN for sending bulk invites.
●	SYSTEM_EMAIL: The default system email for notifications.
Error Handling
●	Invalid Invites: Attendees with missing or invalid details are grouped and reported back to the sender.
●	Unauthorized Senders: Requests from unauthorized senders are logged but not processed.
●	Exception Logging: Errors are logged to ensure traceability.
Summary
The Get New Bulk Event Invites from Email function streamlines handling bulk event invitations by leveraging AWS Lambda, S3, DynamoDB, SNS, and SES. The function ensures efficient and scalable event invite management by validating senders, processing CSV files, and notifying organizers.

**Event Management - Consolidated Technical Overview**
Overview
The Calendar Invite Server (CIS) Event Management module provides a comprehensive set of functions for handling events, invites, cancellations, updates, and attendee management. This module ensures the efficient creation, modification, and processing of events and related invitations while maintaining data integrity and enforcing validation rules.
AWS Services Utilized
●	AWS Lambda: Handles event management processing.
●	Amazon SQS (Simple Queue Service): Manages event-related requests in a queue.
●	Amazon DynamoDB: Stores and updates event and attendee records.
●	Amazon SNS (Simple Notification Service): Publishes notifications for event-related actions.
●	Amazon SES (Simple Email Service): Sends event invitations, cancellations, and updates.
Core Functionalities
1. Event Creation & Verification
●	Create New Event Record: Processes and stores new event details in DynamoDB.
●	Verify New Event Invite Request: Ensures event invitations are authorized before being sent.
2. Event Invite Management
●	Send Event Invite: Sends calendar invites via SES.
●	Send Bulk Event Invites: Processes and sends bulk invitations for events.
●	Get New Bulk Event Invites from Email: Extract event invite details from emails and process them.
●	Get New Event Request from Email: Handles new event requests received via email.
●	Notify Organizer of Successful Event Creation: Confirm to the organizer that an event has been successfully created.
3. Event Update & Cancellation
●	Update Event: Modifies event details and sends notifications of changes.
●	Send Event Update: Notifies attendees about event modifications.
●	Send Event Cancellation: Sends cancellation notices to attendees.
●	Stage Attendees for Updated Event: Identifies attendees who need to be notified of an event update.
●	Stage Attendees for Cancelled Event: Prepares cancellation notifications for all attendees of a canceled event.
4. RSVP & Attendee Management
●	Get New Event Reply from Email: Processes attendee responses (RSVPs) received via email.
●	Update Event Attendee Record: Updates attendee RSVP statuses in DynamoDB.
●	Notify Organizer of Failed Event Creation: Alert the organizer if an event fails to be created.
●	Notify Organizer of Event Limit Reached: Inform the organizer when they exceed their event creation limit.
●	Notify Organizer of Successful Enrollment: Confirms an organizer’s enrollment, typically for subscription-based services.
Workflow Overview
1.	Event Creation
○	A new event is created and verified.
○	Invitations are sent to attendees (either individually or in bulk).
○	Organizers receive confirmation of event creation.
2.	Event Updates & Cancellations
○	Updates to event details trigger notifications to attendees.
○	Cancellations result in staged notifications for all attendees.
3.	RSVP & Attendee Management
○	Attendee responses are processed and recorded in DynamoDB.
○	Organizer notifications are triggered based on event status (success, failure, or limits reached).
4.	Email-Based Processing
○	Event requests and RSVP responses can be handled via email integrations.
○	Bulk event invites can be processed from CSV attachments in emails.
Data Handling & Validation
●	DynamoDB Records
○	Event Record (event#<uid>) tracks event details, timestamps, and status.
○	Attendee Record (attendee#<email>) stores attendee details and RSVP history.
○	Organizer Statistics (organizer#<email>) maintains records of event creation limits and invite statistics.
●	Validation Steps
○	Ensures organizers are authorized before sending invites.
○	Prevents duplicate invites to the same attendee.
○	Verifies attendee RSVP responses before updating records.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	DYNAMODB_TABLE: Name of the DynamoDB table.
●	SQS_URL: URL of the SQS queue for processing event requests.
●	NEW_EVENT_INVITE: SNS topic ARN for approved invites.
●	EVENT_UPDATED: SNS topic ARN for event updates.
●	EVENT_CANCELLATION_REQUEST: SNS topic ARN for event cancellations.
●	SENDER: Email address used for sending event-related notifications.
●	RSVP_EMAIL: Email used for collecting RSVP responses.
Error Handling
●	Invalid Requests: Logs issues and skips processing if required fields are missing.
●	Authorization Failures: Prevents event invites from unauthorized organizers or suspended accounts.
●	Processing Failures: Logs errors and retains failed messages in SQS for retry.
●	Notification Failures: Logs issues in sending invites or updates but does not halt execution.
Summary
The Event Management module in CIS ensures robust handling of event lifecycle operations. By leveraging AWS Lambda, SQS, DynamoDB, SNS, and SES, this module provides automated and scalable event handling, covering the creation, invitations, updates, cancellations, and RSVP tracking. These processes enable seamless coordination between event organizers and attendees while enforcing validation and security measures.

**Create New Event Record - Technical Overview**
Overview
The Create New Event Record function in the Calendar Invite Server (CIS) processes event creation requests. It validates the organizer, stages the event, stores it in DynamoDB, and triggers notifications using SNS. The function also ensures that only authorized organizers can create events, manage event limits, and track event statistics.
AWS Services Utilized
●	AWS Lambda: Executes the function on event trigger.
●	Amazon DynamoDB: Stores event records, organizer details, and statistics.
●	Amazon SNS (Simple Notification Service): Sends notifications for new events and event updates.
●	Amazon SQS (Simple Queue Service): Manages incoming event creation requests.
Functionality Breakdown
1. Event Processing Flow
1.	An SQS event triggers the function.
2.	It retrieves and validates the organizer details.
3.	If the organizer is authorized, the event data is staged.
4.	The event record is stored in DynamoDB using transactional writes.
5.	Notifications for event creation or updates are published to SNS.
6.	Successfully processed messages are removed from SQS.
2. Key Functions
lambda_handler(event, _)
●	Handles incoming SQS messages.
●	Processes each record for event creation.
●	Logs errors and tracks failed tasks for retries.
process_request_from(record)
●	Extracts event data from the incoming request.
●	Retrieves organizer details and verifies authorization.
●	Stages the event and writes it to DynamoDB.
get_organizer_details(request)
●	Retrieves organizer email, subscription status, and event statistics.
organizer_can_send(organizer)
●	Validates the organizer's ability to create an event:
○	Check if the account is suspended.
○	Ensures the organizer has available event slots.
○	Bypass throttling for paid accounts.
create_new_event_record(event, staged_event, organizer)
●	Uses DynamoDB Transactions to:
○	Create a lookup entry for the event’s original UID.
○	Store the main event record.
○	Initialize event statistics.
○	Update the organizer’s event count.
●	Sends a notification for a new or updated event.
stage_request_from(event)
●	Formats event data before storage in DynamoDB.
●	Converts timestamps to epoch format.
●	Sets invite limits and assigns event metadata.
send_notification_of_new_event(event, organizer)
●	Publishes an SNS notification for event creation.
delete_successfully_processed_sqs(message)
●	Removes processed SQS messages to prevent retries.
3. DynamoDB Records
Event Record
●	pk: event#<uid>
●	sk: event#<uid>
●	Stores event metadata (timestamps, status, organizer details, etc.).
Organizer Statistics
●	pk: organizer#<email>
●	sk: organizer_statistics#<email>
●	Tracks the number of events sent, RSVP responses, and other key metrics.
Original Event Lookup
●	pk: original_event#<original_uid>
●	sk: original_event#<original_uid>
●	Maps the original UID to the system-generated UID.
Environmental Variables Used
●	REGION: AWS Region for execution.
●	DYNAMODB_TABLE: The table where event data is stored.
●	NEW_EVENT_CREATED: SNS topic ARN for event creation notifications.
●	EVENT_UPDATED: SNS topic ARN for event updates.
●	EVENT_LIMIT_REACHED: SNS topic ARN for event limit warnings.
●	EVENT_INVITE_LIMIT: Maximum invitees per event.
●	SQS_URL: URL of the SQS queue for event creation requests.
Error Handling
●	Failure Cases: If an error occurs, failed tasks remain in SQS for automatic retry.
●	Exception Logging: Errors are logged to track failures and debugging.
Summary
The Create New Event Record function provides a structured method for creating events in the Calendar Invite Server. By utilizing AWS Lambda, DynamoDB, SNS, and SQS, the function ensures scalable, reliable event processing with built-in authorization, event limits, and notifications. Proper validation, logging, and retry mechanisms contribute to a robust event management system.

**Cancel Event Function - Technical Overview**
Overview
The Cancel Event Function is a core Calendar Invite Server (CIS) component designed to handle event cancellations efficiently. It ensures that events are properly marked as canceled, updates the event records in DynamoDB, and notifies relevant systems via AWS SNS and SQS.
AWS Services Utilized
●	AWS Lambda: Executes the function when triggered.
●	Amazon DynamoDB: Stores and updates event records.
●	Amazon SNS (Simple Notification Service): Sends notifications when an event is canceled.
●	Amazon SQS (Simple Queue Service): Handles event cancellation messages in the processing queue.
Functionality Breakdown
1. Event Processing Flow
1.	An event from AWS SQS triggers the function.
2.	It retrieves the original event using the UID stored in DynamoDB.
3.	If the event is not canceled, its status will be updated to "canceled."
4.	The function stages the updated event data.
5.	A cancellation notification is sent via SNS.
6.	The successfully processed message is deleted from the SQS queue.
2. Key Functions
lambda_handler(event, _)
●	Handles incoming records from SQS.
●	Processes each event cancellation request.
●	Logs errors if any tasks fail.
●	Returns a completion status.
process_request_from(record)
●	Retrieves the event UID from the message payload.
●	Fetches the event details from DynamoDB.
●	Check if the event is already canceled.
●	If not, update the event record and send a cancellation notification.
get_event_uid_from(uid)
●	Queries DynamoDB to retrieve the event record based on its UID.
event_is_not_cancelled(event)
●	Returns True if the event is not already marked as canceled.
cancel_event(event)
●	Updates the event record in DynamoDB:
○	Sets status = "cancelled"
○	Updates dtstamp and last_modified timestamps.
○	Increments the event sequence number.
●	Uses DynamoDB update_item() API to persist changes.
send_notification_of_cancelled(event)
●	Publishes an SNS message to notify subscribed systems of the cancellation.
delete_successfully_processed_sqs(message)
●	Removes the processed SQS message to prevent reprocessing.
cleanup_all(tasks)
●	Ensures failed tasks trigger retries by throwing an exception.
Environmental Variables Used
●	REGION: AWS Region for service execution.
●	DYNAMODB_TABLE: The name of the DynamoDB table storing event records.
●	EVENT_CANCELLED: SNS topic ARN for cancellation notifications.
●	SQS_URL: URL of the SQS queue for event processing.
Error Handling
●	Failure Cases: If an error occurs while processing an event, the task counter tracks failures, and failed messages remain in SQS for retry.
●	Exception Logging: Errors are logged, ensuring debugging and issue resolution.
Summary
The Cancel Event Function provides a structured approach to handling event cancellations in CIS. The function ensures seamless and scalable event management by leveraging AWS Lambda, DynamoDB, SNS, and SQS. Proper logging, notification, and error-handling mechanisms ensure reliability and maintainability.

**Calendar Invite Server (CIS) - Pre-Built Notification Email Templates**
Overview
The Calendar Invite Server (CIS) includes a set of pre-built notification email templates designed to provide users with event-related updates. These templates are essential for communicating event creation, status updates, attendee reports, and other key notifications. The emails are structured in HTML and styled for optimal rendering in email clients.
Location in CICD File Structure
The email templates are located in the following directory within the CIS CICD structure:
Location: ../cis-cicd/data/email-templates
These templates can be modified to reflect each implementation's branding and language.
Email Template Breakdown
1. Event Limit Reached Notification
●	Purpose: Notifies users when they have exceeded their event creation limit.
●	Key Features:
○	Friendly message informing users of the limit.
○	Encourages users to upgrade or manage their event usage.
○	Provides links to support and documentation.
2. Failed Event Notification
●	Purpose: Alerts users that an event creation attempt has failed.
●	Key Features:
○	Clearly states the error and potential causes.
○	Recommends retrying event creation using supported email clients.
○	Provide support and troubleshooting resources.
3. New Event Notification
●	Purpose: Confirms successful creation of a new event in CIS.
●	Key Features:
○	Displays key event details (Event Name, Organizer, EventID).
○	Provides direct links to event management pages.
○	Encourages users to start inviting attendees.
4. Successful Enrollment Notification
●	Purpose: Confirms that a user has successfully subscribed or registered for CIS services.
●	Key Features:
○	Acknowledges enrollment and welcomes the user.
○	Includes links to helpful resources such as documentation and pricing.
○	Serves as a reference for users regarding their subscriptions.
5. Attendee Report Notification
●	Purpose: Delivers a report on attendee activity for a specific event.
●	Key Features:
○	Includes key event details (Event ID).
○	Provides an attached CSV report with attendee data.
○	Explains how to use the data for engagement analysis.
6. Development Version - New Event Notification
●	Purpose: A development/test version of the new event notification.
●	Key Features:
○	Similar to the New Event Notification but linked to a test environment.
○	Allows developers to test event creation workflows before production deployment.
Common Design Elements
●	HTML Structure: Each template is written in XHTML 1.0 Transitional for maximum compatibility.
●	Responsive Design: Uses inline styles for email client consistency.
●	Branding: Includes Calendar Snack (to be updated for CIS branding) logos and color schemes.
●	Dynamic Data: Some templates include placeholders (e.g., {uid}, {summary}, {mailto}) for dynamically inserting event-specific details.
●	Footer & Legal Notices: Each email includes a footer with a disclaimer, copyright notice, and support links.
Next Steps
●	Branding Update: Replace "Calendar Snack" references with "Calendar Invite Server".
●	Testing: Ensure all templates render properly in common email clients.
●	Customization Options: Provide guidelines for users to modify templates as needed.
This document is a technical reference for integrating and managing CIS email notifications.

**Process Shopify Order - Technical Overview**
Overview
The Process Shopify Order function in the Calendar Invite Server (CIS) Dashboard handles order processing for purchases made via Shopify. While this is not a core CIS function, it enables the automation of order validation, subscription registration, and user enrollment.
AWS Services Utilized
●	AWS Lambda: Handles API requests for processing Shopify orders.
●	Amazon DynamoDB: Stores subscription records for enrolled users.
●	Amazon SNS (Simple Notification Service): Sends order confirmation notifications.
Functionality Breakdown
1. Order Processing Flow
1.	Trigger:

○	The function is invoked via a Shopify webhook event.
○	Extracts order details from the request body.
2.	Validate and Parse Order:

○	Extracts the order details, including email, order_number, and line_items.
○	Ensures all required data is present.
3.	Register Subscription:

○	Iterates over each line item in the order.
○	Calls register_subscription() to store user subscription records in DynamoDB.
4.	Send Confirmation Notification:

○	Uses AWS SNS to send a confirmation notification for successful enrollment.
2. Key Functions
lambda_handler(event, _)
●	Extracts order details from the Shopify webhook event.
●	Calls process_order(event) to process the order.
●	Returns HTTP 200 if the order is successfully processed.
process_order(event)
●	Retrieves the order details using get_order(event).
●	Iterates over each line item and calls register_subscription().
●	Returns HTTP 200 if successful, otherwise HTTP 404 for invalid requests.
get_order(event)
●	Extracts the order payload from the request body.
●	Decodes Base64-encoded orders when necessary.
●	Returns a parsed JSON order object.
register_subscription(order_number, order_item, purchase_email)
●	Validates that the order item contains all required details.
●	Extracts relevant subscription data.
●	Calls complete_enrollment(subscription) to store the subscription record.
valid_subscription(order_item, purchase_email)
●	Ensures required fields (product_id, name) are present.
●	Ensures the purchase email is valid.
●	Returns True if valid, False otherwise.
get_organizer_email(order, purchase_email)
●	Extracts the organizer email from custom order properties.
●	Defaults to the purchase email if a custom email is not specified.
complete_enrollment(request)
●	Stores the subscription data in DynamoDB.
●	Uses a conditional check to prevent duplicate entries.
●	Calls send_notification_of_successful_enrollment().
send_notification_of_successful_enrollment(event)
●	Publishes a success notification using AWS SNS.
●	Sends the confirmation to the organizer’s email.
3. Data Handling
DynamoDB Records
●	Subscription Record:
○	pk: organizer#<organizer_email>
○	sk: subscription#<organizer_email>
○	Stores purchase details including product name, product ID, order number, and organizer email.
4. Environmental Variables Used
Variable	Description
REGION	AWS Region for execution.
DYNAMODB_TABLE	Name of the DynamoDB table storing subscription records.
SUCCESSFUL_ENROLLMENT	SNS topic ARN for sending order success notifications.
LOG_LEVEL	Logging verbosity level.
Security Considerations
●	This function should only be accessible via Shopify webhooks.
●	IAM policies should restrict access to SNS and DynamoDB to prevent unauthorized modifications.
●	Subscription records should be write-protected to avoid duplicate entries.
Error Handling
●	Invalid Order Data:
○	Returns HTTP 404 if order data is incomplete.
○	Log missing fields for debugging.
●	DynamoDB Insert Failures:
○	Logs errors but does not disrupt processing.
●	Unhandled Exceptions:
○	Captures unexpected errors and returns an error response.
Summary
The Process Shopify Order function automates subscription handling for purchases made via Shopify. Leveraging AWS Lambda, DynamoDB, and SNS ensures smooth order processing and user enrollment while maintaining security and efficiency.

**Get System Events - Technical Overview**
Overview
The Get System Events function in the Calendar Invite Server (CIS) Dashboard retrieves all system events. Because it can access all event data, this function is restricted to system administrators and code committers.
AWS Services Utilized
●	AWS Lambda: Handles API requests for retrieving system-wide events.
●	Amazon DynamoDB: Stores and retrieves event data.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger:
○	The function is invoked via an API Gateway request.
○	No specific organizer is required; it retrieves all system events.
2.	Retrieve Event List:
○	Queries DynamoDB to fetch all system events.
○	Filters results based on event timestamps.
3.	Format Event Data:
○	Converts raw DynamoDB data into a structured JSON format.
4.	Return Response:
○	Returns the formatted event list in a JSON HTTP 200 response.
2. Key Functions
lambda_handler(event, __)
●	Extracts request details.
●	Calls get_system_event_list() to retrieve all system-wide events.
●	Returns the HTTP response with event data.
get_system_event_list()
●	Calls get_event_records() to retrieve event details from DynamoDB.
●	Formats the retrieved events using format_events_from(event_list).
get_event_records()
●	Queries DynamoDB for all system-wide events.
●	Uses a GSI (Global Secondary Index) named system_events to fetch records.
●	Retrieves only events with a last_modified timestamp greater than 0.
●	Limits query results to the smaller value of EVENT_VIEW_LENGTH or MAX_EVENT_VIEW_LENGTH.
format_events_from(event_list)
●	Converts DynamoDB results into a structured JSON format.
●	Includes fields such as:
○	uid, mailto, organizer, status, created, dtstart, dtend, summary_html, description_html, location_html.
3. Data Handling
DynamoDB Records
●	Event Record:
○	pk: event#<uid>
○	sk: event#<uid>
○	Stores event details including timestamps, organizer email, location, and status.
4. Environmental Variables Used
Variable	Description
REGION	AWS Region for execution.
DYNAMODB_TABLE	Name of the DynamoDB table storing event records.
EVENT_VIEW_LENGTH	Default max number of events returned per request.
MAX_EVENT_VIEW_LENGTH	Upper limit for the number of events that can be retrieved.
LOG_LEVEL	Logging verbosity level.
Security Considerations
●	This function should only be accessible to system administrators and authorized users.
●	IAM policies should restrict access to this function to prevent unauthorized data exposure.
●	Queries use tenant-based filtering to ensure isolation of event data.
Error Handling
●	Unauthorized Access:
○	Access should be restricted via IAM roles and API Gateway authentication.
●	DynamoDB Query Failures:
○	Logs errors when fetching events but does not disrupt processing.
●	Unhandled Exceptions:
○	Captures unexpected errors and returns an error response.
Summary
The Get System Events function provides administrators with full visibility into all system-wide events within the CIS Dashboard. It leverages AWS Lambda and DynamoDB for efficient and controlled retrieval of event data, ensuring secure and structured event management.

Get Organizer Events (Legacy) - Technical Overview
Overview
The Get Organizer Events (Legacy) function in the Calendar Invite Server (CIS) Dashboard is an older version of the get_organizer_events function. It retrieves all events associated with an organizer but relies on a legacy method of extracting the organizer's email via path parameters instead of the request body. This function is maintained for historical reference and backward compatibility but is deprecated in favor of the newer implementation.
AWS Services Utilized
●	AWS Lambda: Handles API requests for retrieving organizer events.
●	Amazon DynamoDB: Stores and retrieves event data.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger:
○	The function is invoked via an API Gateway request.
○	Extracts the organizer email from the request’s path parameters.
2.	Retrieve Event List:
○	Queries DynamoDB to fetch events linked to the organizer.
○	Filters results based on event timestamps.
3.	Format Event Data:
○	Converts raw DynamoDB data into a structured JSON format.
4.	Return Response:
○	Returns the formatted event list in a JSON HTTP 200 response.
○	If an error occurs, returns HTTP 404.
2. Key Functions
lambda_handler(event, _)
●	Extracts request details.
●	Calls get_organizer_events(request) to process the request.
●	Returns the HTTP response with event data.
get_organizer_events(request)
●	Extracts the organizer’s email from the request path parameters.
●	Calls get_event_list(organizer) to retrieve event details.
●	Formats the retrieved events using format_events(events).
●	Returns the formatted list as a JSON response.
get_event_list(organizer)
●	Queries DynamoDB for events associated with the given organizer email.
●	Uses a GSI (Global Secondary Index) named organizer_events to fetch records.
●	Retrieves only events with a last_modified timestamp greater than 0.
●	Limits query results based on the EVENT_VIEW_LENGTH environment variable.
get_organizer(request)
●	Extracts the organizer email from the request’s path parameters instead of the request body.
●	This differs from the modern function, which uses JSON body parsing.
format_events(events)
●	Converts DynamoDB results into a structured JSON format.
●	Includes fields such as:
○	uid, mailto, organizer, status, created, dtstart, dtend, summary_html, description_html, location_html.
invalid_request()
●	Returns a HTTP 404 response for invalid requests.
●	Logs errors for debugging.
3. Data Handling
DynamoDB Records
●	Event Record:
○	pk: event#<uid>
○	sk: event#<uid>
○	Stores event details including timestamps, organizer email, location, and status.
4. Environmental Variables Used
Variable	Description
REGION	AWS Region for execution.
DYNAMODB_TABLE	Name of the DynamoDB table storing event records.
EVENT_VIEW_LENGTH	Max number of events returned per request.
LOG_LEVEL	Logging verbosity level.
Deprecation Notes
●	This function is deprecated in favor of get_organizer_events, which retrieves the organizer email from the request body instead of path parameters.
●	Legacy Compatibility:
○	Older systems relying on GET requests with path parameters may still use this function.
○	Future updates should transition to the new API structure.
Error Handling
●	Missing Organizer Email:
○	Returns HTTP 404 if the request path does not contain a valid organizer email.
○	Logs the issue for debugging.
●	DynamoDB Query Failures:
○	Logs errors when fetching events but does not disrupt processing.
●	Unhandled Exceptions:
○	Captures unexpected errors and returns an error response.
Summary
The Get Organizer Events (Legacy) function is an older method for retrieving an organizer’s events using path parameters instead of the request body. While still operational, it has been replaced by a more robust implementation. The function remains in place for backward compatibility but is no longer recommended for future development.

**Get Organizer Events - Technical Overview**
Overview
The Get Organizer Events function in the Calendar Invite Server (CIS) Dashboard retrieves all events associated with a given organizer. It processes an API request, extracts the organizer’s email, fetches event data from DynamoDB, and returns the results as a JSON response.
AWS Services Utilized
●	AWS Lambda: Handles API requests for retrieving organizer events.
●	Amazon DynamoDB: Stores and retrieves event data.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger:
○	The function is invoked via an API Gateway request.
○	Extracts the organizer email from the request body.
2.	Retrieve Event List:
○	Queries DynamoDB to fetch events linked to the organizer.
○	Filters results based on event timestamps.
3.	Format Event Data:
○	Converts raw DynamoDB data into a structured JSON format.
4.	Return Response:
○	Returns the formatted event list in a JSON HTTP 200 response.
○	If an error occurs, returns HTTP 404 with an error log.
2. Key Functions
lambda_handler(event, _)
●	Extracts request details.
●	Calls get_organizer_events(request) to process the request.
●	Returns the HTTP response with event data or an error message.
get_organizer_events(request)
●	Extracts the organizer’s email from the request body.
●	Calls get_event_list(organizer) to retrieve event details.
●	Formats the retrieved events using format_events(events).
●	Returns the formatted list as a JSON response.
get_event_list(organizer)
●	Queries DynamoDB for events associated with the given organizer email.
●	Uses a GSI (Global Secondary Index) named organizer_events to fetch records.
●	Retrieves only events with a last_modified timestamp greater than 0.
●	Limits query results based on the EVENT_VIEW_LENGTH environment variable.
get_organizer(request)
●	Extracts the organizer email from the request body.
●	Decodes Base64-encoded requests before processing.
format_events(events)
●	Converts DynamoDB results into a structured JSON format.
●	Includes fields such as:
○	uid, mailto, organizer, status, created, dtstart, dtend, summary_html, description_html, location_html.
invalid_request()
●	Returns a HTTP 404 response for invalid requests.
●	Logs errors and warning messages for debugging.
3. Data Handling
DynamoDB Records
●	Event Record:
○	pk: event#<uid>
○	sk: event#<uid>
○	Stores event details including timestamps, organizer email, location, and status.
4. Environmental Variables Used
Variable	Description
REGION	AWS Region for execution.
DYNAMODB_TABLE	Name of the DynamoDB table storing event records.
EVENT_VIEW_LENGTH	Max number of events returned per request.
LOG_LEVEL	Logging verbosity level.
Error Handling
●	Missing Organizer Email:
○	Returns HTTP 404 if the request body does not contain a valid organizer email.
○	Logs the issue for debugging.
●	DynamoDB Query Failures:
○	Logs errors when fetching events but does not disrupt processing.
●	Unhandled Exceptions:
○	Captures unexpected errors and returns an error response.
Summary
The Get Organizer Events function provides a streamlined approach for retrieving an organizer’s event history. By leveraging AWS Lambda and DynamoDB, it ensures efficient data retrieval and structured event management within the CIS Dashboard.


**Get Event Attendee Report - Technical Overview**
Overview
The Get Event Attendee Report function in the Calendar Invite Server (CIS) Dashboard retrieves the list of attendees for a given event, compiles the data into a CSV report, and sends it to the event organizer via email. This function automates attendee reporting and enhances event tracking.
AWS Services Utilized
●	AWS Lambda: Handles the report generation request.
●	Amazon DynamoDB: Stores attendee and event data.
●	Amazon SES (Simple Email Service): Sends the generated report to the event organizer.
●	Amazon CodeCommit: Stores email templates for report notifications.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger:
○	The function is invoked through an API Gateway request.
○	Extracts the event UID from event["pathParameters"]["uid"].
2.	Retrieve Attendee Data:
○	Queries DynamoDB to fetch all attendees associated with the given event.
3.	Determine Organizer Email:
○	If attendees exist, the first attendee’s mailto field is used as the organizer’s email.
○	If no attendees are found, the organizer email is fetched from the event record in DynamoDB.
4.	Generate CSV Report:
○	A CSV file is created containing attendee details (email, name, status, origin, prodid).
○	The file is stored in a temporary local directory.
5.	Send Report via Email:
○	The email template is retrieved from AWS CodeCommit.
○	The CSV file is attached and emailed to the organizer using AWS SES.
2. Key Functions
lambda_handler(event, _)
●	Extracts the event UID from the request.
●	Calls generate_attendee_report_for(uid) to initiate report generation.
●	Returns an HTTP 200 response indicating that the request was acknowledged.
generate_attendee_report_for(uid)
●	Fetches the attendee list for the event.
●	Retrieves the organizer’s email.
●	Calls generate_csv_report_for(event) to create the report.
●	Calls send_report_to_organizer_for(event) to email the report.
get_attendee_list_for(uid)
●	Queries DynamoDB for attendees associated with the given event.
●	Uses the Partition Key (pk) as event#<uid> and filters on Sort Key (sk) prefixed with attendee#.
get_organizer_email_for(event)
●	Determines the organizer’s email address based on the event data.
●	If attendees exist, uses the first attendee’s mailto field.
●	Otherwise, retrieves the organizer email from the event record.
generate_csv_report_for(event)
●	Creates a CSV file with attendee details.
●	Stores the file in a temporary directory for email attachment.
●	The CSV includes fields:
○	email, name, status, origin, prodid.
send_report_to_organizer_for(event)
●	Fetches the email template from AWS CodeCommit.
●	Attaches the CSV report to the email.
●	Sends the email via AWS SES.
get_attendee_report_email_template_for(event)
●	Retrieves the email body template from AWS CodeCommit.
●	Replaces placeholders with event-specific values.
3. Data Handling
DynamoDB Records
●	Attendee Record:
○	pk: event#<uid>
○	sk: attendee#<email>
○	Stores attendee RSVP status, email, and event metadata.
●	Event Record:
○	pk: event#<uid>
○	sk: event#<uid>
○	Stores organizer email and event metadata.
4. Environmental Variables Used
Variable	Description
REGION	AWS Region for DynamoDB and SES.
DYNAMODB_TABLE	Name of the DynamoDB table storing event and attendee records.
LOCAL_CSV_FILE	Path template for storing CSV reports locally.
SUBJECT	Email subject template for attendee reports.
SENDER	Email address used for sending reports.
ATTENDEE_REPORT_EMAIL	AWS CodeCommit file path for the email template.
CODECOMMIT_REPO	AWS CodeCommit repository storing email templates.
Error Handling
●	Missing Event UID: If the API request does not contain a valid uid, logs an error and returns a 400 Bad Request.
●	DynamoDB Query Failures: If the attendee list retrieval fails, logs the error but does not interrupt execution.
●	SES Email Sending Failures: Logs the failure but does not retry.
●	File Handling Errors: Ensures proper cleanup of temporary files to prevent storage issues.
Summary
The Get Event Attendee Report function automates the retrieval, generation, and distribution of attendee reports. By leveraging AWS Lambda, DynamoDB, SES, and CodeCommit, this function enables efficient event tracking and reporting for Calendar Invite Server (CIS) Dashboard.

**Get New Event Invite from API - Technical Overview**
Overview
The Get New Event Invite from API function in the Calendar Invite Server (CIS) Dashboard processes incoming event invite requests from external sources. It validates the request, extracts the required data, and queues the invite for processing via AWS SNS.
AWS Services Utilized
●	AWS Lambda: Handles API requests for processing new event invites.
●	Amazon SNS (Simple Notification Service): Queues event invite requests for processing.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger:

○	The function is invoked via an API Gateway request.
○	Extracts relevant parameters from the request.
2.	Validate Request Data:

○	Checks for a valid email and UID.
○	Ensures the request origin is from a recognized source.
3.	Extract Invite Data:

○	Retrieves the UID, email, attendee name, landing page, and origin.
○	Assigns default values where necessary.
4.	Queue Invite for Processing:

○	Publishes the request to AWS SNS for further handling.
○	Redirects the requestor to the defined landing page.
2. Key Functions
lambda_handler(event, _)
●	Extracts the event request from the API call.
●	Calls get_event_invite_from_api(request) to process the invite.
●	Returns an appropriate HTTP response based on processing results.
get_event_invite_from_api(request)
●	Validates the request format.
●	Calls queue_invite_for_processing(request_data) if valid.
●	Returns HTTP 400 if validation fails.
request_is_valid(request)
●	Validates request parameters:
○	Ensures the email format is valid.
○	Confirms UID structure using regex.
○	Checks if the request origin is in the valid origins list.
valid_origins()
●	Returns a tuple of accepted request origins including:
○	api, bulk, convertkit, emailcta, hubspot, klaviyo, landingpage, mailchimpcta, sendgrid, webform, wix, zoho, and others.
get_request_values_from(request)
●	Extracts required request parameters (uid, email, name, origin).
●	Retrieves the landing page URL, defaulting to a pre-defined value if missing.
●	Assigns default values for missing attributes.
get_landing_page_from(request)
●	Extracts the landing page URL from the request.
●	Ensures URLs start with http:// or https://, defaulting to http:// if missing.
queue_invite_for_processing(request)
●	Prepares an SNS message containing the event invite request.
●	Publishes the message to AWS SNS (NEW_EVENT_INVITE_REQUEST).
●	Redirects the requestor to the landing page.
3. Data Handling
Request Parameters
●	Required:
○	pathParameters["uid"]: Unique event identifier.
○	queryStringParameters["email"]: Organizer’s email address.
●	Optional:
○	queryStringParameters["name"]: Attendee name (defaults to customer).
○	queryStringParameters["origin"]: Origin of the request (defaults to api).
○	queryStringParameters["landing"]: Redirect URL (defaults to predefined value).
4. Environmental Variables Used
Variable	Description
REGION	AWS Region for execution.
NEW_EVENT_INVITE_REQUEST	SNS topic ARN for queuing new event invites.
LOG_LEVEL	Logging verbosity level.
Error Handling
●	Invalid Request Parameters:
○	Returns HTTP 400 if the email or UID is invalid.
○	Logs invalid requests for debugging.
●	SNS Publishing Failures:
○	Logs failures but allows request processing to continue.
●	Unhandled Exceptions:
○	Logs errors for investigation without exposing sensitive details.
Summary
The Get New Event Invite from API function ensures a secure and efficient way to queue event invites for processing. By leveraging AWS Lambda and SNS, it supports scalable and automated event invite handling within the CIS Dashboard.

**Get Event Attendee Sanitized List - Technical Overview**
Overview
The Get Event Attendee Sanitized List function in the Calendar Invite Server (CIS) Dashboard retrieves the list of attendees for a specified event, sanitizes personally identifiable information (PII), and returns a cleaned version of the list. This function ensures privacy while allowing event organizers to analyze attendee data.
AWS Services Utilized
●	AWS Lambda: Handles API requests for attendee data retrieval.
●	Amazon DynamoDB: Stores attendee and event data.
Functionality Breakdown
1. Event Processing Flow
1.	Trigger:
○	The function is invoked via an API Gateway request.
○	Extracts the event UID from event["pathParameters"]["uid"].
2.	Retrieve Attendee Data:
○	Queries DynamoDB to fetch all attendees for the event.
3.	Sanitize Attendee Data:
○	Redacts personally identifiable information (PII) from attendee names and email addresses.
4.	Return Sanitized Data:
○	Returns the cleaned attendee list as a JSON response.
2. Key Functions
lambda_handler(event, _)
●	Extracts the event UID from the request.
●	Calls get_sanitized_attendee_list_for(uid) to fetch and sanitize the attendee data.
●	Returns an HTTP 200 response containing the sanitized attendee list.
get_sanitized_attendee_list_for(uid)
●	Retrieves the attendee list by calling get_attendee_list_for(uid).
●	Calls sanitize(attendee_list) to remove sensitive information.
●	Returns the sanitized attendee data.
get_attendee_list_for(uid)
●	Queries DynamoDB for attendees associated with the given event.
●	Uses the Partition Key (pk) as event#<uid> and filters on Sort Key (sk) prefixed with attendee#.
●	Limits results to 100 attendees per request for efficiency.
sanitize(attendee_list)
●	Iterates through the attendee list and applies sanitization functions:
○	Calls sanitize_sender_from(email) to mask email addresses.
○	Calls sanitize_attendee(name) to obfuscate attendee names.
sanitize_sender_from(email)
●	Redacts the email’s local part, replacing all but the first character with asterisks.
○	Example: johndoe@example.com → j*******@example.com.
sanitize_attendee(name)
●	Redacts the attendee’s name, keeping only the first character visible.
○	Example: John Doe → J*******.
3. Data Handling
DynamoDB Records
●	Attendee Record:
○	pk: event#<uid>
○	sk: attendee#<email>
○	Stores attendee name, email, RSVP status, origin, and event metadata.
4. Environmental Variables Used
Variable	Description
REGION	AWS Region for DynamoDB.
DYNAMODB_TABLE	Name of the DynamoDB table storing event and attendee records.
LOG_LEVEL	Logging verbosity level.
Error Handling
●	Missing Event UID: If the API request does not contain a valid uid, logs an error and returns a 400 Bad Request.
●	DynamoDB Query Failures: If the attendee list retrieval fails, logs the error but does not interrupt execution.
●	Sanitization Issues: Ensures fallback mechanisms in case of unexpected data structures.
Summary
The Get Event Attendee Sanitized List function provides event organizers with a privacy-friendly way to access attendee data. By leveraging AWS Lambda and DynamoDB, it ensures efficient data retrieval while maintaining attendee confidentiality.

**Calendar Invite Server (CIS) Dashboard - Overview**
Introduction
The Calendar Invite Server (CIS) Dashboard provides administrators and authorized users with the tools to manage, track, and process calendar invite events. This dashboard acts as a central hub for event data, offering capabilities for retrieving event details, managing attendees, handling system-wide event processing, and integrating external services such as Shopify for subscription management.
Core Functional Areas
The CIS Dashboard consists of several key functional areas:
1. Event Management
Manages the retrieval and organization of event data at multiple levels:
●	Get Event Attendee Report – Generates a CSV report of all attendees for a given event and emails it to the event organizer.
●	Get Event Attendee Sanitized List – Returns a privacy-compliant list of attendees with personally identifiable information (PII) masked.
●	Get New Event Invite from API – Processes new event invites received via API, validating and queuing them for handling.
●	Get Organizer Events – Retrieves all events associated with a specific organizer.
●	Get Organizer Events (Legacy) – A deprecated function that follows an older method of retrieving organizer events via path parameters instead of request bodies.
●	Get System Events – Provides system administrators with the ability to retrieve all events stored within the CIS database.
2. Attendee and Invitation Handling
Manages attendee information and invites across different methods:
●	Get Event Attendee Report – Extracts attendee data, compiles it into a report, and emails it to the event organizer.
●	Get Event Attendee Sanitized List – Returns a filtered attendee list with minimal data exposure to maintain privacy compliance.
●	Get New Event Invite from API – Facilitates the integration of external invite sources by accepting and processing new invites.
3. System Administration & Security
Provides administrative functions that ensure event data integrity and access control:
●	Get System Events – Allows system administrators to access all events within the CIS infrastructure.
●	Get Organizer Events (Legacy) – Maintained for compatibility with older implementations but replaced with a more secure version.
●	Logging and Monitoring – Integrated with AWS logging mechanisms for auditing and security purposes.
4. E-Commerce Integration
Allows for external integrations with online platforms to facilitate automated subscription handling:
●	Process Shopify Order – Handles purchases made via Shopify, processes the subscription details, and enrolls the user into CIS automatically.
Security Considerations
●	Restricted Access – Some functions, such as get_system_events, are restricted to system administrators or authorized users.
●	Data Privacy – Functions like get_event_attendee_sanitized_list ensure that sensitive attendee information is properly masked.
●	Secure API Processing – Functions that accept external API requests validate inputs and log activities for security monitoring.
Scalability and Future Enhancements
●	Optimized DynamoDB Queries – Ensuring efficient data retrieval while maintaining performance.
●	Expanded API Integrations – Future integrations could include additional event marketing tools and CRM platforms.
●	Enhanced UI/UX – The dashboard could be extended with a web-based interface for improved user interaction.
Conclusion
The CIS Dashboard is a critical component of the Calendar Invite Server, providing structured access to event data, attendee management, and external integrations. It supports a wide range of use cases, from event tracking and reporting to subscription-based enrollments via Shopify. With a focus on security, scalability, and administrative control, the CIS Dashboard serves as a powerful backend for managing calendar invite interactions at scale.

**Calendar Invite Server (CIS) API Overview**
Introduction
The Calendar Invite Server (CIS) provides a set of APIs designed to facilitate the creation, management, and tracking of calendar invites. These APIs are structured into functional categories based on their purpose, including Event, Organizer, System, and Statistics. Each API follows RESTful principles, utilizing AWS API Gateway and Lambda for processing requests.
Authentication & Security
●	Some APIs require an API key for access, ensuring secure usage.
●	AWS IAM roles manage access permissions, restricting data exposure.
●	Authentication is handled using AWS API Gateway security policies.
API Categories
1. Event API
The Event API handles event-related operations, such as retrieving event summaries, managing invites, and generating event-based reports.
Endpoints:
●	GET /event/{uid} - Retrieves event summary details.
●	GET /event/{uid}/invite - Fetches new event invite information.
●	GET /event/{uid}/report - Generates an event attendee report.
●	GET /event/{uid}/statistics - Returns event-level statistical data.
●	GET /event/{uid}/attendees - Provides a sanitized attendee report.
Integration Type: AWS Proxy (some direct AWS integrations for DynamoDB queries)
2. Organizer API
The Organizer API focuses on managing event organizers and their related data.
Endpoints:
●	GET /organizer/{organizer}/events - Fetches a list of events managed by a specific organizer.
●	GET /shadow/organizer/{organizer}/events - Retrieves legacy event data.
●	GET /shadow/organizer/{organizer}/statistics - Provides analytics on organizer-level events.
Integration Type: AWS Proxy and direct AWS integrations
3. System API
The System API provides administrative access to CIS-wide event data.
Endpoints:
●	GET /system/events - Retrieves all events across the system.
●	GET /system/statistics - Fetches system-wide statistics and insights.
Integration Type: AWS Proxy and direct AWS integrations
4. Statistics API
This API provides statistical insights across various levels, including event, organizer, and system-wide reports.
Endpoints:
●	GET /event/{uid}/statistics - Event-specific statistics.
●	GET /shadow/organizer/{organizer}/statistics - Organizer-level analytics.
●	GET /system/statistics - System-wide statistical data.
Integration Type: AWS integrations for DynamoDB queries
5. Order API (Shopify Integration)
●	POST /order - Processes Shopify orders for CIS usage.
Integration Type: AWS Proxy

API Request & Response Formats
All API requests use JSON for request bodies (where applicable) and JSON responses.
Example Request:
GET /event/{uid}/statistics HTTP/1.1
Host: api.calendarinviteserver.com
x-api-key: YOUR_API_KEY

Example Response:
{
  "event_id": "12345",
  "attendee_count": 150,
  "rsvp_confirmed": 120,
  "rsvp_declined": 10,
  "rsvp_pending": 20
}

Error Handling
CIS APIs return standard HTTP status codes:
●	200 OK – Successful request
●	400 Bad Request – Invalid input parameters
●	401 Unauthorized – Missing or invalid API key
●	404 Not Found – Requested resource does not exist
●	500 Internal Server Error – Unexpected server issue
Conclusion
The CIS API suite provides powerful tools for managing event invitations, tracking attendance, and analyzing event data. With a structured RESTful approach and AWS integration, these APIs ensure efficiency, security, and scalability for enterprise users.
 
**Appendix: Additional Enhancements for API Documentation**
To further improve understanding and usability of the CIS APIs, consider incorporating the following enhancements:
1. Use Case Examples
Adding practical scenarios showing how different users (e.g., a marketing team, event organizer, or admin) would use these APIs could help contextualize their purpose. For example:
●	How an event organizer uses the Event API to track RSVP confirmations.
●	How a system administrator uses the System API to generate system-wide reports.
●	How an agency integrates CIS APIs into their existing marketing automation workflows.
2. Authentication & Security Enhancements
Right now, the overview mentions API keys and AWS IAM roles, but it might be beneficial to elaborate on:
●	How API keys are generated and managed. (Is there a dashboard for key management, or is it done manually?)
●	OAuth 2.0 or JWT Support (if applicable).
●	Rate limiting and throttling to prevent abuse.
3. API Versioning & Deprecation
If CIS plans to introduce updates over time, consider a versioning strategy section:
●	Will there be v1, v2, etc.?
●	How will deprecated endpoints be communicated?
●	Are there backward compatibility guarantees?
4. Webhooks / Real-time Data
Does the system support webhooks for real-time event updates? If so, documenting webhook events (e.g., "new RSVP received," "event update sent") could be beneficial.
5. Pagination & Query Parameters
For endpoints that return lists (e.g., /events, /attendees), adding pagination details (e.g., limit, offset, sorting options) would ensure developers can efficiently retrieve data.
6. Sample API Integration Flow
A step-by-step integration guide for a common workflow (e.g., "Sending an Event Invite and Tracking RSVPs") would make the overview more actionable.

**END - GHANCHIN - Audit - FEB 17 - 2025**

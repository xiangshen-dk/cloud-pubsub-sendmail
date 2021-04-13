# pubsub_sendmail - Send emails from Google Cloud Pub/Sub events
## Overview

pubsub_sendmail is a Google [Cloud Function](https://cloud.google.com/functions) that can be triggered by a Google Cloud [Pub/Sub](https://cloud.google.com/pubsub) which then sends an email using Python [smtplib](https://docs.python.org/3/library/smtplib.html) to the desired recipient. You can also have the email come from a static IP address by using a [VPC Access Connector](https://cloud.google.com/vpc/docs/configure-serverless-vpc-access) along with [Cloud NAT](https://cloud.google.com/nat/docs/overview).

pubsub_sendmail is configured using environment variables in the deployment shell script.  These environment variables are described in a later section of this document.  Revisions or bug fixes are welcome.  You can also of course adapt the code to any use cases that are not covered.

SMTP is very flexible but that flexibility can be challenging to deal with from a configuration perspective. The best way to deal with problems in the configuration is to do incremental testing and look at the [Cloud Functions Logs](https://cloud.google.com/functions/docs/monitoring/logging).

## Concepts

### Encryption - Forced, Opportunistic, or None

   It is important to understand the various modes email encryption modes.  With *forced encryption* (also called *implicit*), the entire session with the mail server is encrypted from start to end.  Nothing is in plaintext.  With *opportunistic encryption* (also called *explicit*) the session begins in plaintext but is subsequently elevated to encrypted by sending a STARTTLS command.  Said differently, if encryption is *allowed* (but not *required*) by the server it is *opportunistic*.  If it is required by the server it is *forced*. When no encryption is supported, the entire session is in plaintext (highly unlikely).

## Installation

Here are the high level steps that are needed to deploy pubsub_sendmail.  This will be expanded over time.

1. Identify the following information regarding the mail server.  The mail server provider may not have all of this information up front but gather what you can.

   * Server name

   * Server port?

     >Note: Google Cloud does not allow egress on port 25.  If you are told to use this port, there is often an alternative, typically 587.

   * Encryption mode (Forced, Opportunistic, none)?

   * Do you need to register an IP address with the mail server or the network firewall protecting it?

   * Does the server need to do a DNS lookup on the hostname provided during initialization (EHLO, HELO)? 

2. Identify the additional Google Cloud information.

   * In which [region](https://cloud.google.com/functions/docs/locations) do you want to deploy the function?

3. Gather additional information regarding the transmission of e-mail.  If you want to determine this information dynamically you will need to modify the code to gather this information.

   * What email address should be used for the sender?

   * What email address should be used for the recipient?

   * What subject should be used for the message?

   * What fully qualified domain name ("FQDN") do you want to pass to the mail server as the hostname of pubsub_sendmail?  This is referred to as the *local host*. This is for the EHLO/HELO dialog. You can try leaving it blank and see what your mail server does with the default generated by smtplib.  Keep in mind that the Cloud Function will have no knowledge of the sender's domain unless you choose to assign this value.


4. If you want the egress traffic from pubsub_sendmail to come from a static IP:

   * Select an existing VPC or create a new VPC.

   * Create a /28 subnet in the VPC in the desired region for the VPC Access Connector.

   * Create a VPC Access Conncetor using the /28 you created.

   * Create a Cloud NAT for the VPC if one does not already exist.   The Cloud NAT must be configured to receive traffic sent through the VPC Access Connnector subnet.  Assign a static IP address to the Cloud NAT.

   * If the mail server does a DNS lookup of the *local host* value, you may need to assign a DNS A record to it.  Try the deployment without creating an A record first.

5. Create a service account for the Cloud Function.  The service account itself needs no specific permissions but will be an alternative to the default service account whose permissions are not needed.   When you deploy pubsub_sendmail, you must have the Service Account User permission on this service account so you can "act as" the service account during the deploment.

6. Clone or download this repository.

7. Edit the deploy-pubsub-sendmail file and make the changes below to the environment variables.

   * MAIL_FROM - Set this to the email address of the sender (e.g. user@example.com).

   * MAIL_TO - Set this to the email address of the recipient (e.g. user@example.com).

   * MAIL_SERVER - Set this to the host and TCP port of email server (e.g. mail.example.com:587). If the port number is not specified and MAIL_FORCE_TLS is set to TRUE then the port defaults to 465.  Otherwise the port defaults to 25.  Note that Google Cloud blocks egress to port 25.  Port 587 is often used as an alternative for opportunistic or no encryption.

   * MAIL_SUBJECT - Set this to the email subject (e.g. "Pub/Sub Email").

   * MAIL_FORCE_TLS - Set this to TRUE for forced (also known as implicit) encryption.  If unset or set to any other value then encryption is opportunistic (also known as explicit).

   * MAIL_LOCAL_HOST - Set this to the fully qualified domain to use for the EHLO/HELO SMTP command.  A Cloud Function does not have a host name that is associated with your mail ddomain.  The mail server may be fine without that.  The smtplib functions will generate a default hostname.  You can try this default if you wish but it may not behave as expected.

   * MAIL_DEBUG - Set this to TRUE to generate additional debugging info in the Cloud Functions log.  If unset or set to anything other than TRUE then no additional debugging info is generated.

   * FN_PUBSUB_TOPIC - Set this to the Cloud Function Pub/Sub from which pubsub_sendmail will receive events.

   * FN_REGION - Set this to the Google Cloud region in which to deploy the function.  Note that Cloud Functions must be supported in this region.

   * FN_SOURCE_DIR - Set this to the source directory for the function code, the directory that contains the main.py file.

   * FN_SA - Set this to the service account to use for the pubsub_sendmail Cloud Function.  Use the complete email address of the service account.

   * FN_VPC_CONN - Set this to the VPC Connector to use.  If not set, then egress traffic from pubsub_sendmail will go out over the internet directly.

8. Deploy the pubsub_sendmail function.

   ```
   chmod 755 deploy-pubsub-sendmail
   ./deploy-pubsub-sendmail
   ```

## Common connection problems and ways to address them

1. When you are deploying pubsub_sendmail to a new server, set MAIL_DEBUG to TRUE to put additional information into the logs.

   > Note: The error messages in the logs are not always accurate.

2. Many connection problems arise from a mismatch between the client (pubsub_senndmail in this case) and the mail server/port which is specified in the MAIL_SERVER environment variable.  Make sure you know the correct server name, port, and encryption mode (forced, opportunistic, or none at all).

3. Remember that Google Cloud blocks outbound connections to port 25 - the default port for opportunistic connections.  For opportunistic encryption connections, you will likely use port 587.  You must specifically include this port number in MAIL_SERVER (e.g. mail.example.com:587).

4. If a message in the Cloud Function logs says something like "Wrong Encryption Mode,"  it usually means that there is a conflict between the mail server and pubsub_sendmail.   If the server is supposed to be enforcing encryption, try connecting to it with telnet on the desitred port (assuming your IP is not blocked) and see if any plaintext message appears.  If plaintext appears (e.g. "Hello...."), then encryption s likely opportunistic on that port.  The server may provide forced encryption at a different server or port.  If encryption is required, make sure you set MAIL_FORCE_TLS to TRUE.  If you need to use a port other than 465, you must specify it in MAIL_SERVER.

5. If you want the emails to come from a static IP address, you must set the "FN_VPC_CONN" to the name of the VPC connector to use and also assign a Cloud NAT to the VPC and attach a static IP address to the Cloud NAT.

6. If a message in the Cloud Function logs says something like "IP not registered," it often (but not always) means that pubsub_sendmail is not connecting on a registered IP address.  If a static IP is required, make sure a static IP address is configured on the Cloud NAT and that it has been registered with the mail server.   This error message may also mean that the encryption mode is incorrect.

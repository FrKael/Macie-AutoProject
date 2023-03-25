## Using Macie to scan S3 for Personally Identifiable Information

Make sure your personal information is protected with Macie and S3!

## Features overview

In this project I will document:

* How to create some PII (personally identifiable information) in an Amazon S3 bucket

* Configure Amazon Macie to detect and report that information

* Receive alerts via EventBridge and SNS

* Deploy this solution to the us-east-1 (Virginia) region

* Verify the sending of data to the email

## Architecture

![architecture](https://github.com/FrKael/Macie-AutoProject/blob/main/images/architecture.png)


For this project I will create the example files:

**Credit cards - cc.txt**

```
American Express
5135725008183484 09/26
CVE: 550
American Express
347965534580275 05/24
CCV: 4758
Mastercard
5105105105105100
Exp: 01/27
Security code: 912
```

**Employee information - employees.txt**

```
74323 Julie Field
Lake Joshuamouth, OR 30055-3905
1-196-191-4438x974
53001 Paul Union
New John, HI 94740
Amanda Wells
354-70-6172
242 George Plaza
East Lawrencefurt, VA 37287-7620
GB73WAUS0628038988364
587 Silva Village
Pearsonburgh, NM 11616-7231
LDNM1948227117807
Brett Garza
```

**Access credentials - keys.txt**
```
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_SESSION_TOKEN=AQoDYXdzEPT//////////wEXAMPLEtc764bNrC9SAPBSM22wDOk4x4HIZ8j4FZTwdQWLWsKWHGBuFqwAeMicRXmxfpSPfIeoIYRqTflfKD8YUuwthAx7mSEI/qkPpKPi/kMcGdQrmGdeehM4IC1NtBmUpp2wUE8phUZampKsburEDy0KPkyQDYwT7WZ0wq5VSXDvp75YU9HFvlRd8Tx6q6fE8YQcHNVXAkiY9q6d+xo0rKwT38xVqr7ZD0u0iPPkUL64lIZbqBAz+scqKmlzm8FDrypNC9Yjc8fPOLn9FX9KSYvKTr4rvx3iSIlTJabIQwj2ICCR/oLxBA==
github_key: c8a2f31d8daeb219f623f484f1d0fa73ae6b4b5a
github_api_key: c8a2f31d8daeb219f623f484f1d0fa73ae6b4b5a
github_secret: c8a2f31d8daeb219f623f484f1d0fa73ae6b4b5a
```

**Custom data (Australian licence plates) - plates.txt**
```python
# Victoria
1BE8BE
ABC123
DEF-456
# New South Wales
AO31BE
AO-15-EB
BU-60-UB
# Queensland
123ABC
000ZZZ
987-YXW
```

### Stage 1 - Sync your aws acount, create a then add the example data to S3

[Amazon S3](https://s3.console.aws.amazon.com/s3/home?region=us-east-1)

Sync the account editing the credential file: <kbd>~/.aws/credentials</kbd> or with 

```
aws configure
```

Create a new s3 bucket from CLI using
```
aws s3 mb s3://bucketpii --region us-east-1
```
Upload the files to the s3 bucket
```
$ aws s3 cp ~/simple_macie s3://bucketpii/ --recursive
upload: ./cc.txt to s3://bucketpii/cc.txt
upload: ./plates.txt to s3://bucketpii/plates.txt
upload: ./keys.txt to s3://bucketpii/keys.txt
upload: ./employees.txt to s3://bucketpii/employees.txt
```

verify your files in the S3 bucket with

```
$ aws s3 ls bucketpii
2023-03-11 13:48:30        158 cc.txt
2023-03-11 13:48:30        279 employees.txt
2023-03-11 13:48:30        623 keys.txt
2023-03-11 13:48:30        113 plates.txt
```
Or check the console

![SS1](https://github.com/FrKael/Macie-AutoProject-/blob/main/images/ss1.png)

### Stage 2 - Enable Amazon Macie

[Amazon macie](https://us-east-1.console.aws.amazon.com/macie/)

Follow the directions of Macie -> <kbd>Enable Macie</kbd>

Once enabled, wait a couple of minutes, and refresh a few times until Macie is ready and this screen has to be enabled

![SS2](https://github.com/FrKael/Macie-AutoProject-/blob/main/images/ss2.png)

From the “S3 Buckets” section, then click Create job

![SS3](https://github.com/FrKael/Macie-AutoProject-/blob/main/images/ss3.png)

This first job has:
* named: maciejobpii
* selected S3 bucket: maciejobpii
* Refine the scope: One-time job
* Managed data identifier: all
* Custom data identifiers: None

The entire job might take up to 20 minutes to complete

When all the job is done, the status must be: Complete

![SS4](https://github.com/FrKael/Macie-AutoProject-/blob/main/images/ss4.png)

To see the results of the <kbd>findings</kbd>, choose view in <kbd>show findings</kbd>

![SS5](https://github.com/FrKael/Macie-AutoProject-/blob/main/images/ss5.png)

The result is displayed in this console panel:

![SS7](https://github.com/FrKael/Macie-AutoProject-/blob/main/images/ss7.png)

It can be seen in JSON format too:

![SS8](https://github.com/FrKael/Macie-AutoProject-/blob/main/images/ss8.png)

### Stage 3 - Setting up SNS

[Amazon Simple Notification Service](https://us-east-1.console.aws.amazon.com/sns/home?region=us-east-1)

I have a <kbd>SNS topic</kbd> with:
* Standard type 
* Named: pii-alerts
* Access policy method: Basic
    * Publishers: Only the specified AWS Accounts: <kbd>MyAccountNumber</kbd>
    * Subscribers: Only the specified AWS Accounts <kbd>MyAccountNumber</kbd>
_#this method can also be applied to resources_

![SS9](https://github.com/FrKael/Macie-AutoProject-/blob/main/images/ss9.png)

Add a subscription with a mail (provided by: temp-mail.org)

![SS10](https://github.com/FrKael/Macie-AutoProject-/blob/main/images/ss10.png)

### Stage 4 - Setting up EventBridge

[Amazon EventBridge](https://us-east-1.console.aws.amazon.com/events/)

Create rule: "EventBridge Rule"
I have a <kbd>eventBridge rule</kbd>  with:
* Named pii-events
* Event pattern:
    * Event source: AWS Services
    * AWS service: Macie
    * Event type: Macie finding
* target 1:
    * AWS Service: SNS topic
    * Topic: pii-alerts

![SS11](https://github.com/FrKael/Macie-AutoProject-/blob/main/images/ss11.png)

### Stage 5 - Adding a custom Macie data identifier

In Amazon Macie, I have a <kbd>Custom data identifiers</kbd> with:
* Named LicencePlates
* Regular expression:
```
([0-9][a-zA-Z][a-zA-Z]-?[0-9][a-zA-Z][a-zA-Z])|([a-zA-Z][a-zA-Z][a-zA-Z]-?[0-9][0-9][0-9])|([a-zA-Z][a-zA-Z]-?[0-9][0-9]-?[a-zA-Z][a-zA-Z])|([0-9][0-9][0-9]-?[a-zA-Z][a-zA-Z][a-zA-Z])|([0-9][0-9][0-9]-?[0-9][a-zA-Z][a-zA-Z])
```
_(basic regular expression that will pick up the licence plate formats in the example file.)_

### Stage 6 - Starting a new job

From the “S3 Buckets” section, then click Create job

This second job has:

* named: jobpii2
* selected: bucketpii
* Refine the scope: One-time job
* Managed data identifier: all
* Custom data identifiers: LicencePlates

![SS12](https://github.com/FrKael/Macie-AutoProject-/blob/main/images/ss12.png)

After a few minutes, some non-working emails will be received.

![SS13](https://github.com/FrKael/Macie-AutoProject-/blob/main/images/ss13.png)

These are JSON outputs of Macie's findings, including the custom data identifier.

![SS14](https://github.com/FrKael/Macie-AutoProject-/blob/main/images/ss14.png)

## Result: 

Exploring the custom identifier finding shows that it found 9 License Plates in our plates.txt file

![SS15](https://github.com/FrKael/Macie-AutoProject-/blob/main/images/ss15.png)


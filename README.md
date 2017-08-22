# UiPath Alexa Python

### Overview

This Github contains the code for an Alexa skill. This skill can provision a project to become available to use, and also launch a project on any robot you have enabled on Orchestrator. This is done by:

* If you ask Alexa to **provision** a robot, it launches 'Initial_Robot' on an ADMIN machine which you will have previously set up.
* This admin machine will then run the Python script 'Download_S3_File.py', which downloads your chosen project from a folder within your S3 bucket.
* (The admin machine knows which project to download using an Orchestrator asset, which you must manually assign.)
* The admin machine will then open the downloaded project and publish the file to Orchestrator.
* The admin machine will then open Orchestrator, create a process using the newly published file, and then create a schedule with this process.
* (For the Orchestrator API to run a robot, it requires it to be in the schedule. Therefore, this new process is scheduled to run in 2099, but can be manually invoked at any time.)
* The admin machine then signs out of Orchestrator, and the project is ready to use.
* Then, if you ask Alexa to **launch** a robot, it will check if it has already been provisioned, and then also check if the robot you choose exists.
* If both return true, the project runs on your chosen robot. This can be an AWS server, or a local machine, as long as the device is configured to work with Orchestrator.

Note:

* Project = A .xaml and project.json file.
* Robot = A machine to run a project.

### Known Issues

* If the same UiPath project is provisioned twice, it will fail, as the previous project has not been deleted. I can't seem to get the API to work, but it is possible, and is commented out in the main code (lambda_function.py).

On line ~379 in lambda_function.py:

```
###Delete existing deployed robot:########################################## - tested - API not functioning as expected.
###The API seems to respond with 200 OK with this request, but does not delete the process. For now, the process must be manually deleted.
#response = requests.delete("https://demo.uipath.com/odata/Processes('Lambda_Function')", headers=header_auth)

#if response.status_code != 200:
#    print('Status:', response.status_code, 'Problem with the request. Exiting.')
#    print(response.json())
#    return statement("An error has occured.")
#    exit()
############################################################################
```

* When you ask Alexa to provision a project, it will fail unless you put the project name as an asset in Orchestrator. Again, cannot seem to get the API to perform, and is commented out in the main code.

On line ~305 in lambda_function.py:

```
###Publish project name to Orchestrator ASSET:##############################
##This feature does not work, and you must manually add the asset in orchestrator for now!
#data_initial = {
#  "Name": "ProjectName",
#  "CanBeDeleted": False,
#  "ValueScope": "Global",
#  "ValueType": "Text",
#  "Value": str(ROBOTprocessname),
#  "StringValue": str(ROBOTprocessname),
#  "BoolValue": False
#}
#
#data=json.dumps(data_initial)
#
#response = requests.post("https://demo.uipath.com/odata/Assets", json=data, headers=header_auth)
#
#if response.status_code != 200:
#    print('Status:', response.status_code, 'Problem with the request. Exiting.')
#    print(response.json())
#    return statement("An error has occured. test")
#    exit()
############################################################################
```

* The provisioning robot (Initial_Robot) is delicate, and will need to be drastically improved! It has been 'thrown' together in a short amount of time.
* Sometimes the Download_S3_File.py does not create the required directory on the ADMIN machine, sometimes it does, it is temperamental.
* If the Alexa skill behaves abnormally, ask the skill to "Reset conversation", as a first step to diagnosing the issue.

### Prerequisites

Before getting started with this project, you will require:

* An Amazon Echo or Echo Dot (or be willing to use the Amazon simulator.)
* An installation of Python 2.7 on your device, found [here](https://www.python.org/downloads/).
* An Amazon Web Services account.
* An Amazon Developer account.
* Some pre-obtained knowledge on the functionality of AWS [Lambda](https://aws.amazon.com/lambda/) and [Alexa Skills Kit](https://developer.amazon.com/edw/home.html#/skills).
* UiPath installed on an ADMIN machine of your choice.
* A UiPath Orchestrator account/server.

It is also advised to install the necessary module [virtualenv](https://virtualenv.pypa.io/en/stable/), as this will be required during the deployment process. This can be achieved by:

```
setx PATH "%PATH%;C:\Python27\Scripts" (Windows only - Optional)

pip install virtualenv
```

Note: The first line enables 'pip' to be used to install Python modules, and you can skip this line if you have used 'pip' on your machine previously. Additionally, if an error occurs, you may need to close and reopen your CMD terminal window after entering the first line before installing virtualenv.

## Getting Started

First of all, you will need to create an S3 bucket on Amazon Web Services ([AWS](https://aws.amazon.com/)), with permissions available only to yourself, and not public. This bucket will contain Alexa-related data.. You will then need to create a directory on your local device, such as "Alexa_Skill", and download the [lambda_function.py file](https://github.com/jlindsey1/UiPath_API/blob/master/Alexa_Skill/lambda_function.py) to that location. It is important that the filename is not altered throughout this process.

Furthermore, ensure that the region you are in (found at in the top bar of the AWS console) is the same as where you intend the Alexa skill to be used. In this readme, Ireland (eu-west-1) is used, but this can be changed as needed.

### UiPath Orchestrator

Before proceeding, ensure you have an [Orchestrator](https://www.uipath.com/orchestrator) server ready to use, and you have linked one or more robots to this server. You will need to dedicate one of these machines to be the 'ADMIN' robot, which becomes the robot that provisions future projects on Orchestrator.

This ADMIN robot can be a local device, or a server on AWS. A t2.micro is sufficient, and linking this robot to Orchestrator is identical to using a local machine.

Within Orchestrator, you will need to create three assets.

* Password1: This is the password of your Orchestrator account, which the ADMIN robot will use.
* ProjectName: This is the project which you wish for Alexa to provision and run. For example: 'Project1' (Eventually, Alexa should be able to update this automatically, but for now, this must be changed manually.)
* User: This is the username of your ADMIN robot. For example: 'ADMINISTRATOR', or 'JLINDSEY'.

Note: The 'ProjectName' must be identical (case-sensitive) to the name of your UiPath Projects stored in your S3 bucket.

### AWS Credentials

Zappa requires [AWS CLI](http://docs.aws.amazon.com/cli/latest/userguide/installing.html), the Amazon Command Line Interface. To install this, the following Python module is needed:

```
pip install awscli
```

To obtain the credentials to configure awscli, navigate to [AWS IAM Roles](https://console.aws.amazon.com/iam/):

* Navigate to 'users'.
* Add a user.
* Choose a username, such as 'Zappa-Deployment-User'.
* Enable: Programmatic Access.
* Navigate to the next page.
* Choose 'Attach existing policies directly'.

At this stage, you must decide which permissions you with to grant to Zappa program. The easiest solution is to provide 'AdministratorAccess' (**Highly Recommended - Much easier**), but if you wish to be more restrictive, the following are the minimum essentials:

* APIGatewayAdministrator
* AWSLambdaFullAccess
* IAMReadOnlyAccess

(Depending on whether you choose to use AdministratorAccess or minimal permissions, sections of this readme will vary, so please ensure you follow the correct instructions.)

Create and confirm the creation of the user, then note the Access key ID & secret access key, as you will need them momentarily.

(It is possible to be increasingly further restrictive, and is a [topic of open discussion.](https://github.com/Miserlou/Zappa/issues/244))

To configure the AWS CLI, type:

```
aws configure
```

* AWS Access Key ID: (Use the key obtained previously.)
* AWS Secret Access Key: (Use the key obtained previously.)
* Default region name: (Select the region of your S3 bucket location.)
* Default output format: (Select a format you prefer, as this does not impact on this installation. Default = None.)

At this point, do not discard the secret access key, as it is still needed later.

### Zappa

Due to the dependencies of the Python modules used within lambda_function.py, there is only one method of deploying the function to AWS Lambda. The automated and simplified deployment service [Zappa](https://github.com/Miserlou/Zappa) can be used, and a detailed guide can be found for using Zappa on the corresponding [Github](https://github.com/Miserlou/Zappa). Alternatively, this readme will demonstrate the required method.

First, navigate to the directory where the lambda_function.py file is stored. Open the file, and then modify the constants at the top of the file with the name and region of your S3 bucket. (The region you choose **MUST** match the true location of the S3 bucket, or the Alexa skill will fail.) **You also need to add your Orchestrator log-in details**. For example:

```
userbucketname='s3-bucket-name-example'
userbucketregion='eu-west-1'
orchestrator_username="admin"
orchestrator_password="************"
orchestrator_tenancy="capgeminitest"
ADMINprocessname="Initial_Robot"
ADMINmachine=????
```
If you don't know the ADMINmachine number, leave it as '0', continue this installation, and then ask Alexa to "List all robots". Then, when Alexa tells you the ID number, type that into the variable *ADMINmachine* above and upload the Python skill again. If unsure, there is at the end of this readme on how to add code once your skill is already deployed.

Save and exit the file. Then, create a Python Virtual Environment (in Terminal/CMD) in the directory you made (eg Alexa_Skill) by
```
virtualenv virtual-env
```
where 'virtual-env' is the name of the environment. The name of this can be user-specified, and can vary however needed.

Then, activate the virtual environment by (for Windows):
```
virtual-env\Scripts\activate.bat
```
or (for macOS):
```
source  virtual-env/bin/activate
```

If this is successful, the environment name (virtual-env) will appear on the left in the terminal window, to inform you that you are based within the virtual environment. Next, install the following modules:

```
pip install zappa
pip install boto3
pip install botocore
pip install flask
pip install flask_ask
pip install requests
```

(Note, installing boto3 is optional and should already function on AWS Lambda. However, it has been stated here for completeness.)

Now, to initialise Zappa, type:

```
zappa init
```

Zappa will ask you what production stage this program is in. You can call this anything, but for this readme, it will be called 'dev' or 'development'.

```
dev
```

Then you have to decide the name of your bucket. You can use the same bucket as the one created previously at the start of this readme. Enter that here.

```
s3-bucket-name-example
```

You will then be asked the name of your app's function. The default option given is required here.

```
lambda_function.app
```

Then you must decide if you wish to deploy the function globally. This choice is yours, but it is unlikely to be the case. For this readme, 'n' (no) will be chosen.

Then confirm the settings with 'y'.

Now navigate to the folder where 'lambda_function.py' is stored, and open 'zappa_settings.json'. It should look like the code below. Add in the lines with (ADD) attached.

```
{
    "dev": {
        "app_function": "lambda_function.app",
        "aws_region": "eu-west-1",
        "profile_name": "default",
        "s3_bucket": "s3-bucket-name-example",
        "keep_warm": false, (ADD THIS LINE)
        "timeout_seconds": 30, (ADD THIS LINE)
        "memory_size": 256, (ADD THIS LINE)
        "manage_roles": false, (ADD THIS LINE - BUT SET TO 'TRUE' IF USING 'AdministratorAccess' WHEN YOU CREATED THE USER ROLE.)
        "role_name":"alexa-skill-lambda-role-zappa-dev" (ADD THIS LINE)
    }
}
```

The additional lines perform the following:

* "keep_warm": Calls the Lambda function every 4 minutes. For development/testing, this isn't necessary, but can be enabled if needed.
* "timeout_seconds": How long the Lambda function is allowed to run before timing out. Usually this requires no more than 30 seconds, but the maximum limit is 5 minutes.
* "memory_size": Amount of RAM that is allocated to the function. 256MB is usually plenty, but can be increased if needed.
* "manage_roles": This prevents Zappa from trying to manage the IAM roles. **SET THIS TO 'TRUE' IF USING 'AdministratorAccess'.**
* "role_name": This can be whichever name you prefer, just make sure it is memorable.

## Deployment

Before deploying the Lambda function, you must first create the role which the function will use. There are two ways of doing this, depending on whether you are using AdministratorAccess on your user role.

### AWS Lambda - Without Administrator Access (Not Recommended)

If you are **NOT** using Administrator access on the user role you created previously, do the following:

* You will need to create the IAM role for Zappa. To start this, navigate to [AWS IAM Roles](https://console.aws.amazon.com/iam).
* Then navigate to 'roles'.
* Create a custom role.
* Choose AWS Service Role > AWS Lambda.
* Enable S3FullAccess, CloudWatchFullAccess.
* Choose next, and give the role a name, and create the role.

* Click on  'Add inline policy' or 'Create role policy', at the bottom of the page.
* Choose 'Custom policy'.
* Paste in the contents of [this file.](https://github.com/jlindsey1/UiPath_API/blob/master/Alexa_Skill/ZappaPermissions1.json) These are the default permissions assigned by Zappa, but can be further restricted if needed.

Then to create the Lambda function:

* In terminal, still within the virtual environment described earlier, write:

```
zappa deploy dev
```
where 'dev' is the development stage you named when setting up Zappa. This will .zip your code and automatically upload and create your Lambda function for you. **This step will take a few minutes to complete.**

### AWS Lambda - With Administrator Access (Recommended)

Alternatively, if you **ARE** using Administrator access on the user role you created previously, do the following:

* Create the Lambda function using terminal, still within the virtual environment described earlier, write:

```
zappa deploy dev
```
where 'dev' is the development stage you named when setting up Zappa. This will .zip your code and automatically upload and create your Lambda function for you. **This step will take a few minutes to complete.**

* Then click on the [AWS IAM role](https://console.aws.amazon.com/iam) named 'alexa-skill-lambda-role-zappa-dev', which was created automatically by Zappa, but named by you in an earlier step.
* This role should already permitted to access S3, but to be sure, additional policies will need to be manually granted. To do this, click on 'Attach Policy'.
* Add the following: S3FullAccess, CloudWatchFullAccess. Then attach the policies.

***Regardless of whether or not you used AdministratorAccess, the process is now identical from this point on:***

* When the process has completed, Zappa will provide you with a URL, and this will be used later when setting up the Alexa skill. It will be in the form:

```
https://XXXXXXXXXX.execute-api.XXXXXXXXXX.amazonaws.com/dev
```

Now navigate to the [Lambda](https://aws.amazon.com/lambda/) function.

* Ensure that your Lambda function is present. It will be in the form 'XXXXXXXXXX-dev', where 'dev' is the development stage set when setting up Zappa previously.
* If the function is not present, within Zappa, try redeploying the function with:

```
zappa update dev
```

This command will automatically update the Lambda function with any changes you make to the code. **You won't have to reconfigure the IAM's roles again.**

The Lambda function is now created and ready to use.

### Amazon Alexa Skill Setup

Within your Amazon Developer Portal, navigate to the [Alexa Skills Kit](https://developer.amazon.com/edw/home.html#/skills). Then:

* Add a new skill.
* Change the language to the relevant location. (Note: This skill was tested with English UK.) (This language CANNOT be changed at a later date.)
* Choose a name for the skill, and an invocation name. This is what users will say to initiate your skill. (Example: "Orchestrator")
* For the 'Intent Schema', copy and paste the text in the file /Alexa_Skill/intentschema.json on [Github](https://github.com/jlindsey1/UiPath_API/blob/master/Alexa_Skill/intentschema.json).
* For the 'Sample Utterances', copy and paste the text in the file /Alexa_Skill/sampleutterances.txt on [Github](https://github.com/jlindsey1/UiPath_API/blob/master/Alexa_Skill/sampleutterances.txt).
* In 'Custom Slot Types', create a slot called 'response', and enter two responses, stating 'yes' and 'no'.
* In configuration, choose HTTPS, and then select the geographical region of your Lambda function. In the text box, paste in the URL Zappa provided to you previously, in the form:

```
https://XXXXXXXXXX.execute-api.XXXXXXXXXX.amazonaws.com/dev
```

* For 'Certificate for EU Endpoint', choose 'My development endpoint is a sub-domain of a domain that has a wildcard certificate from a certificate authority'.

### Configuring the ADMIN Machine

Hard method:

* Create this directory on the device: "C:/Users/Documents/Github/UiPath_API/Lambda_Function".
* Within the /Lambda_Function folder, copy the file [Download_S3_File.py](https://github.com/jlindsey1/UiPath_API/blob/master/Lambda_Function/Download_S3_File.py) into that location.
* Within the .py file, modify the AWS Credentials to match those you obtained earlier in this setup.
* Copy the folder [Initial_Robot](https://github.com/jlindsey1/UiPath_API/tree/master/Lambda_Function/Initial_Robot) (the whole folder, not just the contents) to within /Lambda_Function.

Easy method: Log into Github on the Admin machine, clone this repository. Then:

* Within the Download_S3_File.py file, modify the AWS Credentials to match those you obtained earlier in this setup.

At this point, discard the secret access key.

### Testing

In the testing panel, type:

"What is the date."

and the skill should respond with:

"date/month"

if the skill is functioning. If you receive an error, investigate the CloudWatch logs and diagnose accordingly.

## Additional Requirements

You will require at least one project folder in your S3 bucket, which contains the **.xaml** and **project.json** files for a UiPath robot.. An example of this can be found in /Lambda_Function/project1 on [Github](https://github.com/jlindsey1/UiPath_API/tree/master/Lambda_Function). This will open Notepad and type 'This worked!'.

## Features and Functionality

The following assumes the invocation name is "Orchestrator".

### Initiating/Resetting the Skill

*	“Alexa, ask Orchestrator to…” for a specific question, or: “Alexa, launch Orchestrator” if you do not have a specific question.
(You can invoke this at any time if the skill pauses, and it will remember where you left off.)
*	To reset the conversation, say “Alexa, ask Orchestrator to reset conversation.”

### Launching a Robot

(Alexa responses are in *italics*.)

*	“(Alexa, ask Cloud to) Launch robot.”
*	*“Which project would you like to launch?”*
*	“One”
*	*“Which robot would you like to use?”*
*	"Three seven nine seven." (**If you are unsure of the robot name, ask Alexa to "List all robots", and the numbers will be announced to you.**)
*	*“Success.”*

### Provisioning a Robot

(Alexa responses are in *italics*.)

(**Make sure you have deleted the previous process (if it exists), and that the project name asset has been assigned in Orchestrator. This is due to the API not peforming as expected for now...**)

*	“(Alexa, ask Cloud to) Provision robot.”
*	*“Which project would you like to provision?”*
*	“One”
*	*“Success.”* (**This will launch the provisioning robot on the ADMIN machine. The number you specify corresponds to the number in your S3 bucket, sorted by alphabetical order.**)

### Listing all Robots

(Alexa responses are in *italics*.)

To list robots ready to use:

*	“Alexa, ask Cloud to list all robots.”
*	*“Name: Local Machine. Number: 3797.”*

### Listing all Projects

(Alexa responses are in *italics*.)

To list projects in your S3 bucket ready to use:

*Function not implemented.*

## Adding Modifications to the Python Code

If you make any changes to lambda_function.py after the initial deployment with Zappa, updating the code is incredibly simple - and this is the key benefit to Zappa.

First, activate the virtual environment by (for Windows):
```
virtual-env\Scripts\activate.bat
```
or (for macOS):
```
source  virtual-env/bin/activate
```

Then, type
```
zappa update dev
```
and that's it. After Zappa has deployed your code, it should be ready to use immediately.

## Debugging

If you find the skill is not performing as expected, navigate to the Alexa Skill simulator within your [Alexa Skills Kit](https://developer.amazon.com/edw/home.html#/skills). This way, you will be able to see the response returned by the Lambda function. If the response returned is

```
There was an error calling the remote endpoint, which returned (error here).
```
or
```
The response is invalid.
```

then viewing the CloudWatch log for your Lambda function should help diagnose why the skill failed.

If any help is required, please contact the developer for this skill.

## Built With

* [Flask](http://flask.pocoo.org/) - A microframework for Python.
* [Flask-Ask](https://github.com/johnwheeler/flask-ask) - Flask extension used to simplify the Python code when building the Alexa skill.
* [Zappa](https://github.com/Miserlou/Zappa) - A Python 2.7 server-less deployment package.

## Authors

* **Jordan Lindsey** - [Github](https://github.com/jlindsey1)

## License

Flask is Copyright (c) 2015 by Armin Ronacher and contributors. Some rights reserved.

Flask-Ask is licensed under the Apache License 2.0.

Zappa is provided under the MIT License.

This project is Copyright (c) 2017 by Capgemini UK.

## Acknowledgments

* .

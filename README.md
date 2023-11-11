# Wild Rydes X
<img src="wildrydes.png" alt="Frontend Sync" width="700"> 
A simple serverless web application that enables users to request unicorn rides from the Wild Rydes fleet.  

  
The application will present users with an HTML-based user interface for indicating the location where they would like to be picked up and will interact with a RESTful web service on the backend to submit the request and dispatch a nearby unicorn. The application will also provide facilities for users to register with the service and log in before requesting rides.


  
  
## Prerequisites
You will need an AWS account, An account with ArcGIS to add mapping to your app, a text editor, and a web browser.

## Application architecture
The application architecture uses ```AWS S3, Cloudshell, CodeCommit, Lambda, Amazon API Gateway, Amazon DynamoDB, Amazon Cognito, and AWS Amplify Console.```  
  
  
Amplify Console provides continuous deployment and hosting of the static web resources including HTML, CSS, JavaScript, and image files which are loaded in the user's browser. JavaScript executed in the browser sends and receives data from a public backend API built using Lambda and API Gateway. Amazon Cognito provides user management and authentication functions to secure the backend API. Finally, DynamoDB provides a persistence layer where data can be stored by the API's Lambda function.

<img src="Serverless_Architecture.png" alt="Frontend Sync" width="700">   


## 1. Static Web Hosting

AWS Amplify hosts static web resources including HTML, CSS, JavaScript, and image files which are loaded in the user's browser.

## 2. User Management

Amazon Cognito provides user management and authentication functions to secure the backend API.


## 3. Serverless Backend

Amazon DynamoDB provides a persistence layer where data can be stored by the API's Lambda function.


## 4. RESTful API

JavaScript executed in the browser sends and receives data from a public backend API built using Lambda and API Gateway.  


## Module 1: Static Web Hosting with Continuous Deployment
1. Create an empty repository in CodeCommit
2. Add a policy to your IAM user so you can access Codecommit (AWSCodeCommitPowerUser)
3. Create Git credentials for your IAM user to allow HTTPS connections to CodeCommit


Sample: 
```
User name: iamuser-at-211572490796
Password: dD/qSSsbm9LewrwFWEfWEFWEfWEsdsDsfSdefewARa=
```
4. Clone the repository. Use AWS Cloudshell(Create an empty folder for future code)
"git clone https://git-codecommit.ca-central-1.amazonaws.com/v1/repos/wildrydes-siteX"  
Input the username and password from Step 3  
Output should be: warning: You appear to have cloned an empty repository.  

5. Copy the project code from the S3 bucket and commit it to the new repo.  

Paste in Cloudshell: ```aws s3 cp s3://wildrydes-us-west-2/WebApplication/1_StaticWebHosting/website ./ --recursive```


After a successful copy, do "git add ." to push it to your repo from your Cloudshell  
Then "git commit -m "Initial Commit"  

```git config --global user.email "sampleemail23@xmail.com"```  
```git config --global user.name "iamuser"```  
```git commit -m "Initial Commit"```  
```git push```

Go to AWS Amplify  
Choose "Host a Web App"  
Choose AWS CodeCommit  
Select a repository  
Hit next, review and deploy  
Wait for it to finish  
Then click on the link to see if it works  

Go back to CodeCommit  
Choose index.html  
Edit the title to your own satisfaction. Then Save.  
After that, watch Amplify build again.  

## Module 2: Manage Users  
We will create an Amazon Cognito user pool to manage your users' accounts.  
  
Go to Cognito,
Create a user pool
After that copy the following:  
```User pool ID: ca-west-4_tfhCwerfb```
```Client ID: 5dfeklflkwelhlklwnblkrrlkwkl```

Back to CodeCommit  
Go to wildrydes-siteXjsconfig.js  
Input the UserPoolID and ClientID and Region  
Hit Save.  
  
Then register on the site,  
Copy the invoke URL  


```eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJGYXJkYW5nZGFyZ2luZyIsImVtYWlsIjoiZmFyZGFuZ0BleGFtcGxlLmNvbSIsImlhdCI6MTY5OTU4MTA0MywiZXhwIjoxNjk5NTg0NjQzLCJpc3MiOiJHYW1lcyIsImF1ZCI6IkdhbWVzIn0.SevNTT5GHPLgGOc8R5ISJQfMOk_A0V3uMZ8amRqNc7w```  

## Module 3: Serverless Service Backend  
We will use AWS Lambda and Amazon DynamoDB to build a backend process for handling requests for your web application  

Go to DynamoDb  
Create a table.  
name a Table ```RydeX``` or whatever you like, it should be "String"  
The rest can be defaults.  
Wait for it to finish creating.  
Then click on the name, and find the ARN. Copy the ARN  

```arn:aws:dynamodb:ca-west-6:52546575453689:table/RydesX```

Go to IAM  
Choose Role  
Name the role  
Look for ```AWSLambdaBasicExecutionRole```  
Create role  

Search for the newly-created role.  
Add permission  
Create Inline Policy  
Add Write:PutItem  
On Resources, choose Specific and choose Text. Then paste the DynamoDb table ARN we copied earlier  

Go to Lambda  
Create Function  
```Author From Scratch```  
Give a function name  
Choose Node.js 16.x for Runtime  

On Change default execution role,  
"Use an existing role"  
Then search for "WildRydesLambda"  

Create Function  
Paste the code to replace the index.js in Lambda  

```const randomBytes = require('crypto').randomBytes;
const AWS = require('aws-sdk');
const ddb = new AWS.DynamoDB.DocumentClient();

const fleet = [
    {
        Name: 'Angel',
        Color: 'White',
        Gender: 'Female',
    },
    {
        Name: 'Gil',
        Color: 'White',
        Gender: 'Male',
    },
    {
        Name: 'Rocinante',
        Color: 'Yellow',
        Gender: 'Female',
    },
];

exports.handler = (event, context, callback) => {
    if (!event.requestContext.authorizer) {
      errorResponse('Authorization not configured', context.awsRequestId, callback);
      return;
    }

    const rideId = toUrlString(randomBytes(16));
    console.log('Received event (', rideId, '): ', event);

    // Because we're using a Cognito User Pools authorizer, all of the claims
    // included in the authentication token are provided in the request context.
    // This includes the username as well as other attributes.
    const username = event.requestContext.authorizer.claims['cognito:username'];

    // The body field of the event in a proxy integration is a raw string.
    // In order to extract meaningful values, we need to first parse this string
    // into an object. A more robust implementation might inspect the Content-Type
    // header first and use a different parsing strategy based on that value.
    const requestBody = JSON.parse(event.body);

    const pickupLocation = requestBody.PickupLocation;

    const unicorn = findUnicorn(pickupLocation);

    recordRide(rideId, username, unicorn).then(() => {
        // You can use the callback function to provide a return value from your Node.js
        // Lambda functions. The first parameter is used for failed invocations. The
        // second parameter specifies the result data of the invocation.

        // Because this Lambda function is called by an API Gateway proxy integration
        // the result object must use the following structure.
        callback(null, {
            statusCode: 201,
            body: JSON.stringify({
                RideId: rideId,
                Unicorn: unicorn,
                Eta: '30 seconds',
                Rider: username,
            }),
            headers: {
                'Access-Control-Allow-Origin': '*',
            },
        });
    }).catch((err) => {
        console.error(err);

        // If there is an error during processing, catch it and return
        // from the Lambda function successfully. Specify a 500 HTTP status
        // code and provide an error message in the body. This will provide a
        // more meaningful error response to the end client.
        errorResponse(err.message, context.awsRequestId, callback)
    });
};

// This is where you would implement logic to find the optimal unicorn for
// this ride (possibly invoking another Lambda function as a microservice.)
// For simplicity, we'll just pick a unicorn at random.
function findUnicorn(pickupLocation) {
    console.log('Finding unicorn for ', pickupLocation.Latitude, ', ', pickupLocation.Longitude);
    return fleet[Math.floor(Math.random() * fleet.length)];
}

function recordRide(rideId, username, unicorn) {
    return ddb.put({
        TableName: 'Rides',
        Item: {
            RideId: rideId,
            User: username,
            Unicorn: unicorn,
            RequestTime: new Date().toISOString(),
        },
    }).promise();
}

function toUrlString(buffer) {
    return buffer.toString('base64')
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=/g, '');
}

function errorResponse(errorMessage, awsRequestId, callback) {
  callback(null, {
    statusCode: 500,
    body: JSON.stringify({
      Error: errorMessage,
      Reference: awsRequestId,
    }),
    headers: {
      'Access-Control-Allow-Origin': '*',
    },
  });
}
```


Then deploy and execute Test. You should get status 200  

## Module 4: Deploy a RESTful API
We will use Amazon API Gateway to expose the Lambda function you built in the previous module as a RESTful API.  
  
Now go to API Gateway  
Create REST API  
New API  
API Name: WildRydesX  
API endpoint type: Edge-optimized  
Create API  

Go to API Gateway Authorizer  

Create authorizer  
Name the authorizer  
Authorizer type: Cognito  
choose User Pool  
Token source: Authorization  

Resources  
Create resource  
Resource name: ride  
Enable CORS  
Click yellow button => Create resource  

Create Method:  
Method type: POST  
Integration type: Lambda Function  
Toggle On: Lambda proxy integration  

Click on Method Request  
Edit  
Authorization: Cognito UserPool  
Save  
Deploy API  
Stage: New Stage:  
Stage Name: Dev  

Copy Invoke URL  

```https://wf2kun903m.execute-api.ca-east-4.amazonaws.com/dev```

Back to CodeCommit  
Go to wildrydes-siteXjsconfig.js  
Add the Invoke URL  
Save  

Note: It is possible that you will see a delay between updating the config.js file in your S3 bucket and when the updated content is visible in your browser. You should also ensure that you clear your browser cache before executing the following steps.  
  
Update the ArcGIS JS version from 4.3 to 4.6 in the ride.html file as:  

```<script src="https://js.arcgis.com/4.6/"></script>```  
```<link rel="stylesheet" href="https://js.arcgis.com/4.6/esri/css/main.css">```

2. Save the modified file. Add, commit, and git push it to your Git repository to have it automatically deploy to AWS Amplify console.  

3. Visit /ride.html under your website domain.  

4. If you are redirected to the ArcGIS sign-in page, sign in with the user credentials you created previously in the Introduction section as a prerequisite of this tutorial.  

5. After the map has loaded, click anywhere on the map to set a pickup location.  

6. Choose Request Unicorn. You should see a notification in the right sidebar that a unicorn is on its way and then see a unicorn icon fly to your pickup location.  

## Module 5: Resource Cleanup  
Please delete all resources to AVOID UNWANTED CHARGES  

Delete your Amplify app.

In the AWS Amplify console, select the wildrydes-site app you created in Module 1.  
On the app homepage, choose Actions and select Delete app. Enter delete when prompted to confirm, then choose Delete.  


If you used the provided AWS CloudFormation template to complete Module 2, simply delete the stack using the AWS CloudFormation console. Otherwise, delete the Amazon Cognito user pool you created in Module 2.  
  
In the Amazon Cognito console, select your WildRydes User pool name.  
Choose Delete user pool.  
Select the checkbox next to Deactivate deletion protection.  
Enter WildRydes to confirm deletion, and choose Delete.  
  
Delete the AWS Lambda function, IAM role and Amazon DynamoDB table you created in Module 3.  
  
AWS Lambda function  
  
In the AWS Lambda console on the Functions page, select the RequestUnicorn function you created in Module 3.  
From the Actions drop-down, choose Delete function.  
  
IAM role  
  
In the IAM console, select Roles from the left navigation pane.  
Enter WildRydesLambda into the filter box.  
Select the checkbox next to the role you created in Module 3, WildRydesLambda and choose Delete.  
To confirm deletion, enter WildRydesLambda into the text input field. Choose Delete.  
  
Amazon DynamoDB table  
  
In the Amazon DynamoDB console, select Tables in the left navigation pane.  
Select the checkbox next to the Rides table you created in Module 3.  
Choose Delete.  
Select the checkbox next to Delete all CloudWatch alarms for Rides, enter confirm in the text input field, and choose Delete.  
The Status field on the Tables page will change to Deleting, and the table will disappear from the tables list when it has been successfully deleted.  
  
Delete the REST API created in Module 4.  
  
In the Amazon API Gateway console, select the WildRydes API you created in Module 4.  
In the Actions dropdown, choose Delete.  
Choose Delete on the Delete API confirmation screen.  
  
AWS Lambda automatically creates a new log group per function in Amazon CloudWatch Logs and writes logs to it when your function is invoked. You should delete the log group for the RequestUnicorn function. Also, if you launched any CloudFormation stacks, there may be log groups associated with custom resources in those stacks that you should delete.  
  
From the Amazon CloudWatch console, expand Logs in the left navigation pane and select Log groups.  
Select the checkbox next to the /aws/lambda/RequestUnicorn log group. If you have several log groups in your account, you can enter /aws/lambda/RequestUnicorn into the Filter text box to locate the log group.  
Select Delete log group(s) from the Actions dropdown.  
Choose Delete when prompted to confirm.  
If you launched any CloudFormation templates to complete a module, repeat steps 2-4 for any log groups that begin with /aws/lambda/wildrydes-webapp.  



Credits to AWS

[Build a Serverless Web Application with AWS](https://aws.amazon.com/getting-started/hands-on/build-serverless-web-app-lambda-apigateway-s3-dynamodb-cognito/)
















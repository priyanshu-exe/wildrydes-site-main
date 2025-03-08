# AWS End-to-End Web Application with 7 Services

We're creating a web application for a unicorn ride-sharing service called Wild Rydes. The app uses IAM, Amplify, Cognito, Lambda, API Gateway and DynamoDB, with code stored in GitHub and incorporated into a CI/CD pipeline with Amplify.

The app will let you create an account and log in, then request a ride by clicking on a map (powered by ArcGIS). The code can also be extended to build out more functionality.

Hosting the Frontend with AWS Amplify 
Hosting the frontend of a serverless application with AWS Amplify is straightforward. AWS Amplify is a development platform that helps to host and deploy full-stack web applications with integrated CI/CD capabilities. 
1.	Access the Amplify Console: In the AWS Management Console, navigate to the Amplify service. Click "Get started" and select "Host web app." 
2.	Link GitHub Repository: Choose GitHub as the source provider, authenticate with your account, and select your "wildrides-site-main" repository. This connection enables Amplify to monitor the repository for changes, automatically triggering new deployments. 
3.	Configure Build Settings: Review the build settings. By default, Amplify will build and deploy every change made to the main branch, ensuring the live application is always up-to-date. 
4.	Specify Application Name: Name your Amplify app, such as "wild-rides-2023." After clicking "Create app," Amplify will set up the infrastructure and initiate the first deployment. 
 
 
 
 
 
Once deployment is complete, Amplify provides a URL for accessing the Wild Rides application. Using Amplify offers scalability, as it automatically manages the infrastructure needed to handle incoming traffic and adjusts resources to meet demand. 
Implementing User Authentication with Amazon Cognito 
To enable user authentication, Amazon Cognito provides a secure and scalable user directory that can handle registration, sign-in, and authentication for your web and mobile applications. 
1.	Create a Cognito User Pool: Navigate to Amazon Cognito in the AWS Management Console and select "Manage User Pools" to create a new pool. 
2.	Configure Pool Settings: During the setup, define how users sign in, such as by username or email. You can configure password policies, multi-factor authentication, and account recovery options to enhance security. 
3.	Integrate with Application: Copy the user pool ID and app client ID, then open the application’s configuration file (e.g., config.js). Update these values to connect the app with Cognito’s authentication system. 
4.	Deploy Changes: Commit and push the updated config.js file to GitHub, which triggers Amplify to redeploy the app with the new authentication settings. 
  
 
 
Amazon Cognito simplifies the management of user data, offering secure authentication without requiring additional backend infrastructure. It also integrates seamlessly with other AWS services, enabling role-based access control. 
Implementing the Serverless Backend 
A core component of the Wild Rides application is its backend, which will handle serverless operations like storing and retrieving ride requests. By using serverless services, you only pay for resources when they’re used, reducing overall operational costs. 
Creating the DynamoDB Table 
Amazon DynamoDB is a fully managed NoSQL database that provides high availability and scalability for storing ride request data. 
1.	Create the Table: In the DynamoDB console, create a new table named "rides" with a partition key called "rideId" (String). This setup enables the database to uniquely identify each ride request. 
2.	Configure Provisioning: DynamoDB offers both on-demand and provisioned capacity modes, allowing you to adjust based on anticipated usage. For production environments, consider enabling auto-scaling to handle unpredictable workloads effectively. 
 
Implementing Serverless Logic with AWS Lambda 
AWS Lambda lets you run backend logic without the need for server provisioning. In the Wild Rides app, Lambda will handle ride dispatch requests and store them in DynamoDB. 
1.	Set Up Lambda Function: Go to AWS Lambda and create a function named "request-unicorn2023" using the Node.js 16.x runtime. Grant the Lambda function permissions to access the DynamoDB table. 
2.	Write Lambda Code: Replace the default function code with logic that inserts ride requests into the DynamoDB table. For instance, the function could accept user input like ride location and unicorn preference. 
3.	Test Lambda Function: To ensure proper functionality, create a test event in Lambda. Verify that the function correctly inserts data into DynamoDB. Testing the function provides valuable insights into any adjustments needed before integrating it with the frontend. 
   
  
Integrating the Frontend and Backend with API Gateway 
API Gateway enables the creation of RESTful APIs that connect AWS services to the application’s frontend. 
1.	Create API: In the API Gateway console, create a REST API named "wild-rides-2023". This API will expose Lambda functions as endpoints for the frontend to interact with. 
2.	Define Routes and Methods: Add a resource path, such as "/ride," and configure it with a POST method that invokes the "request-unicorn-2023" Lambda function. 
3.	Secure API with Cognito: Set up the API Gateway to use the Cognito user pool for authorization, ensuring only authenticated users can request rides. 
4.	Deploy API: Deploy the API to a new stage (e.g., "dev"). Copy the Invoke URL, as it will be added to the config.js file in the application to connect the frontend with the backend. 
  
Testing the Application 
After setting up the components, you can test the full functionality of the Wild Rides application. 
1.	Access the Application: Open the app in your browser, register a new user, or log in to your account. 
2.	Request a Ride: Click on the map to request a unicorn ride. The frontend should invoke the Lambda function, which then saves the request to DynamoDB. 
3.	Verify Ride Requests: Check the DynamoDB table to confirm the data was saved successfully. 
Testing each functionality is crucial for identifying any potential issues before going live. Conducting various tests, such as unit tests on Lambda code and integration tests on API Gateway, ensures that the system is reliable and user-ready. 
  
Cleaning Up Resources 
Once testing is complete, it’s important to delete resources to prevent additional costs. 
1.	Delete Amplify and Cognito Resources: Remove the Amplify app and Cognito user pool from the AWS console. 
2.	Remove Lambda Function and DynamoDB Table: Navigate to AWS Lambda and DynamoDB to delete the function and table. 
3.	Delete API Gateway: Remove the API from API Gateway. 
4.	Optional GitHub Cleanup: If the repository is no longer needed, you can delete it from GitHub to prevent unintended access to the code.  
 
 


## The Application Code
The application code is here in this repository.

## The Lambda Function Code
Here is the code for the Lambda function, originally taken from the [AWS workshop](https://aws.amazon.com/getting-started/hands-on/build-serverless-web-app-lambda-apigateway-s3-dynamodb-cognito/module-3/ ), and updated for Node 20.x:

```node
import { randomBytes } from 'crypto';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';

const client = new DynamoDBClient({});
const ddb = DynamoDBDocumentClient.from(client);

const fleet = [
    { Name: 'Angel', Color: 'White', Gender: 'Female' },
    { Name: 'Gil', Color: 'White', Gender: 'Male' },
    { Name: 'Rocinante', Color: 'Yellow', Gender: 'Female' },
];

export const handler = async (event, context) => {
    if (!event.requestContext.authorizer) {
        return errorResponse('Authorization not configured', context.awsRequestId);
    }

    const rideId = toUrlString(randomBytes(16));
    console.log('Received event (', rideId, '): ', event);

    const username = event.requestContext.authorizer.claims['cognito:username'];
    const requestBody = JSON.parse(event.body);
    const pickupLocation = requestBody.PickupLocation;

    const unicorn = findUnicorn(pickupLocation);

    try {
        await recordRide(rideId, username, unicorn);
        return {
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
        };
    } catch (err) {
        console.error(err);
        return errorResponse(err.message, context.awsRequestId);
    }
};

function findUnicorn(pickupLocation) {
    console.log('Finding unicorn for ', pickupLocation.Latitude, ', ', pickupLocation.Longitude);
    return fleet[Math.floor(Math.random() * fleet.length)];
}

async function recordRide(rideId, username, unicorn) {
    const params = {
        TableName: 'Rides',
        Item: {
            RideId: rideId,
            User: username,
            Unicorn: unicorn,
            RequestTime: new Date().toISOString(),
        },
    };
    await ddb.send(new PutCommand(params));
}

function toUrlString(buffer) {
    return buffer.toString('base64')
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=/g, '');
}

function errorResponse(errorMessage, awsRequestId) {
    return {
        statusCode: 500,
        body: JSON.stringify({
            Error: errorMessage,
            Reference: awsRequestId,
        }),
        headers: {
            'Access-Control-Allow-Origin': '*',
        },
    };
}
```

## The Lambda Function Test Function
Here is the code used to test the Lambda function:

```json
{
    "path": "/ride",
    "httpMethod": "POST",
    "headers": {
        "Accept": "*/*",
        "Authorization": "eyJraWQiOiJLTzRVMWZs",
        "content-type": "application/json; charset=UTF-8"
    },
    "queryStringParameters": null,
    "pathParameters": null,
    "requestContext": {
        "authorizer": {
            "claims": {
                "cognito:username": "the_username"
            }
        }
    },
    "body": "{\"PickupLocation\":{\"Latitude\":47.6174755835663,\"Longitude\":-122.28837066650185}}"
}
```


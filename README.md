## Step 1: Create DynamoDB Table
- DynamoDB > Table > Create table
- Enter table name and partition key (table's primary key)
- Create an item in the table and fill attributes like id, name, price, availability, etc. for testing.

## Step 2: Create IAM Role for Lambda Function
- Create an IAM role - ServerlessCRUDLamdaRole
- Add policies - AWSLambdaBasicExecutionRole for basic Cloudwatch logging
- Create an inline policy with following permissions
    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "VisualEditor0",
                "Effect": "Allow",
                "Action": [
                    "dynamodb:PutItem",
                    "dynamodb:DeleteItem",
                    "dynamodb:GetItem",
                    "dynamodb:UpdateItem",
                    "dynamodb:Scan"
                ],
                "Resource": "arn:aws:dynamodb:ap-south-1:437384699476:table/CoffeeShop"
            }
        ]
    }
    ```

## Step 3: Create Lambda Function
- Create a lambda function 
- Enter function name
- Runtime - Node 22.x
- Architecture : arm64 (for cost saving)
- Execution role: Choose the role created above
- Now, In local, run 
    ```
    mkdir get 
    cd get
    npm init
    npm i @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
    touch index.mjs 
    ```
- Write the logic for getting the dynamodb items
- Zip the code
    ```
    cd get 
    zip -r get.zip . # zips everything in the current directory (.) to get.zip file
    ```
- OR Manually go inside the get/ folder select all and compress to get.zip file
- Upload the zip to lambda
- Change the handler if needed 
- Deploy and test the code
    ```
    lambda.zip
    ├── index.js           <- Your Lambda entry point (e.g., handler)
    ├── utils.js           <- Any other local modules
    ├── package.json
    └── node_modules/      <- Installed dependencies
    ```

### Likewise do it for the rest of the functions
- get/
- post/
- update/
- delete/

Notice, how all these lambda have common packages but we still have to upload this heavy node modules each time
In future, if there's a need to install some other npm package then node modules folder has to be updated for all the lambda functions
which is redundant and time consuming
There's a way to simplify this. We can do the change once and this can be imported in all the lambda functions with the help of something called Lambda layer.

## Step 4: Create Lambda Layer

### Initialize the nodejs folder
```
mkdir nodejs
cd nodejs
npm init
npm i @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb

<!-- To keep the common utilities -->
touch utils.mjs
```

### Write utils.mjs
This contains DynamoDB imports, client initialization, exports and createResponse function which are common for all the lambda functions
```
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import {
    DynamoDBDocumentClient,
    ScanCommand,
    GetCommand,
    PutCommand,
    UpdateCommand,
    DeleteCommand
} from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

const createResponse = (statusCode, body) => {
    return {
        statusCode,
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(body),
    };
};

export {
    docClient,
    createResponse,
    ScanCommand,
    GetCommand,
    PutCommand,
    UpdateCommand,
    DeleteCommand
};
```

### Zip the folder

```
cd .. 
zip -r layer.zip nodejs
```

Zip file structure must be
```
layer.zip
└── nodejs/
    ├── node_modules/
    ├── package.json
```
### Upload the lambda layer on AWS
- Go to AWS Lambda > Layers.
- Click Create layer.
- Upload your layer.zip file.
- Select Node.js runtime - 22.x

### Use the Layer in Your Lambda Function
- Go to your Lambda function.
- In the Layers section, click Add a layer.
- Choose your custom layer and version.
- Save the function.


## Step 5: Create API Gateway to expose the lambda functions via APIs

1. API Gateway > HTTP API > Build 
2. Enter API name - CoffeeShop
3. Keep defaults and build
4. Create routes GET /coffee and GET /coffee/{id}
5. Create an integration 
   - Integration > Create integration > 
   - Target - Lambda > Choose the Lambda function. 
6. Attach integration to the routes
7. Test the /get and /get/c001 routes 
8. Add rest of the routes (method and paths) with lambda integration
9. Routes
   ```
    GET /coffee  -> getCoffee lambda function
    GET /coffee/{id}  -> getCoffee lambda function
    POST /coffee  -> createCoffee lambda function
    PUT /coffee/{id}  -> updateCoffee lambda function
    DELETE /coffee/{id}  -> deleteCoffee lambda function
   ```
Note: Routes consist of two parts: an HTTP method and a resource path, for example GET /pets.

## Step 6: Create Cognito user pool

1. Cognito > Create user pool
2. App type: Single-page application
3. Name your app: CoffeeShopClient
4. Options for sign-in identifiers: Email
5. Required attributes for sign-up: Email
6. Add a return URL: http://localhost:5173
7. Click on Create user directory

## Step 7: Create authorizer in API Gateway

1. Authorizer type: JWT
2. Name: CoffeeShopClient
3. Issuer URL: 
4. Audience: Client Id (from cognito)

## Step 8: Integrate cognito authorizer with all routes
Now when you access the API directly using browser. The following message is seen
```
GET /coffee 
{
  "message": "Unauthorized"
}
```

## Step 9: Add Cognito authentication code to react app
## Authentication Flow (AWS Cognito)

This app uses [AWS Cognito](https://aws.amazon.com/cognito/) for authentication, integrated via [`react-oidc-context`](https://github.com/authts/react-oidc-context).

### How It Works

- The app is wrapped in an `AuthProvider` (see `main.jsx`), which provides authentication state and methods to all components.
- In `App.jsx`, authentication state is accessed using the `useAuth()` hook.

#### User Experience

1. **Sign In**
    - If the user is not authenticated, a **Sign in** button is shown.
    - Clicking **Sign in** triggers a redirect to the Cognito login page.
    - After successful login, Cognito redirects back to the app and the user is authenticated.

2. **Authenticated State**
    - When authenticated, the main app (`<Home />`) is displayed.
    - A **Sign out** button is shown, which removes the local session.

3. **Sign Out**
    - Clicking **Sign out** (when not authenticated) logs the user out from Cognito and redirects back to the app.

#### Code Reference

```jsx
// App.jsx (excerpt)
if (auth.isAuthenticated) {
  return (
    <div>
      <Home />
      <button onClick={() => auth.removeUser()}>Sign out</button>
    </div>
  );
}

return (
  <div>
    <button onClick={() => auth.signinRedirect()}>Sign in</button>
    <button onClick={() => signOutRedirect()}>Sign out</button>
  </div>
);
```

### Environment Variables

Make sure to set the following in your `.env` file:

```
VITE_COGNITO_AUTHORITY=...
VITE_COGNITO_CLIENT_ID=...
VITE_COGNITO_REDIRECT_URI=...
VITE_COGNITO_DOMAIN=...
VITE_COGNITO_LOGOUT_URI=...
```

---

**Summary:**  
Authentication is handled automatically by `react-oidc-context` and AWS Cognito. The app shows sign-in/sign-out buttons as appropriate, and protects routes based




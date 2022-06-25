-   Feature flagging is the future of continuous releases; we control what the users see through a flag management interface and completely de-couple continuous deployment from continuous delivery. Companies embracing flag management technology have a competitive advantage, being able to "test in production" up front and control the rollout of their features though flag management solutions such as LaunchDarkly.
    
    We previously covered LaunchDarkly (LD) feature flag (FF) setup and test strategies for front-end applications in [Effective Test Strategies for Front-end Applications using LaunchDarkly Feature Flags and Cypress](https://dev.to/muratkeremozcan/effective-test-strategies-for-testing-front-end-applications-using-launchdarkly-feature-flags-and-cypress-part1-the-setup-jfp). In contrast, this two-part series focuses on a deployed NodeJS service, a serverless app on AWS featured in the book [Serverless Applications with Node.js](https://www.manning.com/books/serverless-applications-with-node-js?query=serverless%20aciton) by Slobodan Stojanović and Aleksandar Simović. You might recognize the API from the blog post [CRUD API testing a deployed service with Cypress using cy-api, cy-spok, cypress-data-session & cypress-each](https://dev.to/muratkeremozcan/crud-api-testing-a-deployed-service-with-cypress-using-cy-api-spok-cypress-data-session-cypress-each-4mlg).
    
    In part1 we setup LaunchDarkly feature flags in our lambda handlers, deploy the lambda using ClaudiaJs, verify our service's behavior via [VsCode REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) and [Amazon CloudWatch](https://aws.amazon.com/cloudwatch/). In part 2 we bring in Cypress, api e2e test the feature flags, and showcase effective feature flag test strategies that can work for any deployed service. The PR for part1 of the blog series can be found at [feature flag setup and test](https://github.com/muratkeremozcan/pizza-api/pull/4). The PR for part2 of the blog series can be found at [e2e testing LD feature flags with Cypress](https://github.com/muratkeremozcan/pizza-api/pull/7). The branch saga through the blog series looks like the below. Once can check out these and follow along the blog, granted they have a LaunchDarkly account (2 week trial), an AWS account and familiarity with deploying lambdas.
    
    1.  `before-feature-flags`
        
    2.  `ld-ff-setup-test` : Part 1 where we fully setup the node SDK for our lambda and showed it working via rest client.
        
    3.  `before-cypress-setup`
        
    4.  `cypress-setup`
        
    5.  `after-cypress-setup`
        
    6.  `ld-ff-ld-e2e`: Part 2 : testing the deployed service and feature flags
        
    
    Let's start by setting up a project and a simple feature flag at the [LaunchDarkly web app](https://app.launchdarkly.com/). Here is the high level overview of the blog post.
    
    -   [Setup at LaunchDarkly web app](#setup-at-launchdarkly-web-app)
        
        -   [Create a project](#create-a-project)
            
        -   [Create a Boolean FF for later use](#create-a-boolean-ff-for-later-use)
            
    -   [Setup the LD client instance at our service](#setup-the-ld-client-instance-at-our-service)
        
        -   [LD & lambda function basic sanity test](#ld--lambda-function-basic-sanity-test)
            
        -   [Add a FF to `update-order` handler](#add-a-ff-to-update-order-handler)
            
        -   [Reusable module to get flag values](#reusable-module-to-get-flag-values)
            
    -   [Setup environment variables](#setup-environment-variables)
        
        -   [Gather the values from the LD web app](#gather-the-values-from-the-ld-web-app)
            
        -   [Local env vars and `process.env`](#local-env-vars-and-processenv)
            
        -   [Lambda env vars](#lambda-env-vars)
            
    -   [Summary](#summary)
        
    -   [References](#references)
        
    
    ## Setup at [LaunchDarkly web app](https://app.launchdarkly.com/)
    
    ### Create a project
    
    Nav to _[https://app.launchdarkly.com/settings/projects](https://app.launchdarkly.com/settings/projects) > Create project_. Give it any name like `pizza-api-example`, and the rest as default.
    
    ![Create project](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5igpl5cpf4knugljjftm.png)
    
    Two default environments get created for us. We can leave them as they are, or delete one of them for our example. The critical item to note here is the **SDK key**, since we are not using a client-side ID. In contrast to our Node API here, the UI app with React was using the [clientSideID](https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress/blob/main/src/index.js#L8). In the beginning code samples, we will keep the SDK key as a string. Later we will use `dotenv` to read them from a local `.env` file, and configure the lambda environment variable.
    
    ![Sdk key](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bylyld4eg4840181fx1o.png)
    
    ### Create a Boolean FF for later use
    
    Nav to [https://app.launchdarkly.com/pizza-api-example/test/features/new](https://app.launchdarkly.com/pizza-api-example/test/features/new) and create a boolean feature flag named `update-order`. You can leave the settings as default, and enter optional descriptions. We will use the flag to toggle the endpoint `PUT {{baseUrl}}/orders/{{orderId}}`.
    
    ![Create a feature flag](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8uzw38r75pf8l0550n83.png)
    
    ## Setup the LD client instance at our service
    
    Install the LD SDK as a dependency; `npm install launchdarkly-node-server-sdk`.
    
    ### LD & lambda function basic sanity test
    
    Let's start with a simple example, console logging whether the LD client instance initialized successfully. In the handler file `./handlers/get-orders.js` import the LD client, initialize it, add a simple function to log out the initialization, then invoke it anywhere in the `getOrders()` function.
    
    // ./handlers/get-orders.js  
    ​  
    // ...other imports...  
    ​  
    const ld = require('launchdarkly-node-server-sdk');  
    ​  
    // initialize the LD client  
    ​  
    const ldClient = ld.init("sdk-**your-SDK-KEY**");  
    ​  
    // add a simple function to log out LD client status  
    const ldClientStatus = async (event) => {  
      let response = {  
        statusCode: 200,  
      };  
      try {  
        await client.waitForInitialization();  
        response.body = JSON.stringify("Initialization successful");  
      } catch (err) {  
        response.body = JSON.stringify("Initialization failed");  
      }  
      return response;  
    };  
    ​  
    // the handler function we had in place   
    function getOrders(orderId) {  
      console.log("Get order(s)", orderId);  
    ​  
      console.log("INVOKING LAUNCHDARKLY TEST");  
      ldClientStatus().then(console.log);  
    ​  
      // ... the rest of the function ...
    
    Upload the lambda. We are assuming you are familiar with deploying lambdas, and for our example all it takes is `npm run update` or `npm run create` for the initial lambda creation. ClaudiaJs is used under the hood to handle all the complexities. What we want to see at the end is LD giving information about the stream connection. ![Upload get-orders sanity for ldClient](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/atyusdvpjunnnry76g8g.png)
    
    We use the [VsCode REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) - or any API test utility - to send a request for `GET {{base}}/orders`.
    
    ![Sanity test ldClient](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fkpzbl6slaz14jf07ovt.png)
    
    Once we can confirm the LD info and the log `Initialization Successful` at CloudWatch logs , then we have proof that the setup is working.
    
    ![CloudWatch GET sanity](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2yqdr2knaf7djk2w9snd.png)
    
    ### Add a FF to `update-order` handler
    
    Regarding how to setup the Node SDK and use feature flags, there are a few approaches in LD docs. We like the recipe at the [LD with TS blog post](https://launchdarkly.com/blog/using-launchdarkly-with-typescript/) the best.
    
    // handlers/update-order.js  
    ​  
    // ... other imports ...  
    ​  
    // require launchdarkly-node-server-sdk  
    const ld = require("launchdarkly-node-server-sdk");  
    ​  
    // ldClient holds a copy of the LaunchDarkly client   
    // that will be returned once the SDK is initialized  
    let ldClient;  
    ​  
    /** Handles the initialization using the SDK key,  
     * which is available on the account settings in the LaunchDarkly dashboard.  
     * Once the client is initialized, getClient() returns it. */  
    async function getClient() {  
      const client = ld.init("sdk-****");  
      await client.waitForInitialization();  
      return client;  
    }  
    ​  
    /** A generic wrapper around the client's variation() method   
     used to get a flag's current value  
     * Initializes the client if it doesn't exist, else reuses the existing client.  
     * Populates an anonymous user key if one is not provided for user targeting. */  
    async function getLDFlagValue(key, user, defaultValue = false) {  
      if (!ldClient) ldClient = await getClient();  
    ​  
      if (!user) {  
        user = {  
          key: "anonymous",  
        };  
      }  
    ​  
      return ldClient.variation(key, user, defaultValue);  
    }  
    ​  
    function updateOrder(orderId, options) {  
      console.log("Update an order", orderId);  
    ​  
      getLDFlagValue("update-order").then((flagValue) => {  
        console.log("FEATURE FLAG VALUE IS:", flagValue);  
      });  
    ​  
      // ... the rest of the handler code ...
    
    Proceed to turn on the flag at the LD interface.
    
    ![update flag on](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8g6f0m4rqypczspk9euk.png)
    
    Deploy the lambda with `npm run update`. Use the rest client to update an order. We should be getting a 200 response, and seeing the value of the flag at Amazon CloudWatch, whether the flag value is true or false.
    
    ![CloudWatch: functions at handler](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qmy1pxs7frlk9caj7eob.png)
    
    ### Reusable module to get flag values
    
    There are two challenges with our code. First, we would have to duplicate it in any other handler that is using feature flags. Second, the `ldClient` variable being in the global scope is not optimal.
    
    What if we could put it all in a module, from which we could import the utility function `getLDFlagValue` to any handler? What if the handler calling our utility function had exclusive access to the LaunchDarkly client instance without any other part of the application knowing about it? Let's see how that can work. Create a new file `get-ld-flag-value.js`.
    
    We use an IIFE and wrap the module so that `ldClient` cannot be observed by any other part of the application. This way, the handler has exclusive access to the LaunchDarkly client instance.
    
    // ./handlers/get-ld-flag-value.js  
    ​  
    const ld = require("launchdarkly-node-server-sdk");  
    ​  
    const getLDFlagValue = (function () {  
      let ldClient;  
    ​  
      async function getClient() {  
        const client = ld.init("sdk-***");  
        await client.waitForInitialization();  
        return client;  
      }  
    ​  
      async function flagValue(key, user, defaultValue = false) {  
        if (!ldClient) ldClient = await getClient();  
    ​  
        if (!user) {  
          user = {  
            key: "anonymous",  
          };  
        }  
    ​  
        return ldClient.variation(key, user, defaultValue);  
      }  
    ​  
      return flagValue;  
    })();  
    ​  
    module.exports = getLDFlagValue;
    
    Import our utility function at our handler, and use the constant with any kind of logic. For our example, if the flag is true, we update the order as usual. If the flag is off, we return information about the request letting the requester know that we received it, and we let them know that the feature is not available. The final version of our handler should look like the below.
    
    const AWSXRay = require("aws-xray-sdk-core");  
    const AWS = AWSXRay.captureAWS(require("aws-sdk"));  
    const docClient = new AWS.DynamoDB.DocumentClient();  
    const getLDFlagValue = require("./get-ld-flag-value");  
    ​  
    async function updateOrder(orderId, options) {  
      // we acquire the flag value  
      const FF_UPDATE_ORDER = await getLDFlagValue("update-order");  
    ​  
      console.log("You tried to Update the order: ", orderId);  
      console.log("The flag value is: ", FF_UPDATE_ORDER);  
    ​  
      if (!options || !options.pizza || !options.address) {  
        throw new Error("Both pizza and address are required to update an order");  
      }  
    ​  
      // once we have the flag value, any logic is possible  
      if (FF_UPDATE_ORDER) {  
        return docClient  
          .update({  
            TableName: "pizza-orders",  
            Key: {  
              orderId: orderId,  
            },  
            // Describe how the update will modify attributes of an order  
            UpdateExpression: "set pizza = :p, address = :a",   
            ExpressionAttributeValues: {  
              // Provide the values to the UpdateExpression expression  
              ":p": options.pizza,  
              ":a": options.address,  
            },  
            // Tell DynamoDB that you want a whole new item to be returned  
            ReturnValues: "ALL_NEW",   
          })  
          .promise()  
          .then((result) => {  
            console.log("Order is updated!", result);  
            return result.Attributes;  
          })  
          .catch((updateError) => {  
            console.log(`Oops, order is not updated :(`, updateError);  
            throw updateError;  
          });  
      } else {  
        console.log("Update order feature is disabled");  
        return {  
          orderId: orderId,  
          pizza: options.pizza,  
          address: options.address,  
        };  
      }  
    }  
    ​  
    module.exports = updateOrder;
    
    Update the lambda with `npm run update`. Set the flag to true and send a request using rest client. The feedback should look like the below
    
    ![Flag true](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hmmhjdrezhsg42139zv9.png)
    
    Toggle the flag value to false at the LD interface. Send another PUT request using rest client. We should be getting the below feedback.
    
    ![Flag false](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/r6j8vn783arhm24zrwd5.png)
    
    Notice that when we toggled the flag, we did not have to deploy our lambda again. **This is why feature flagging is the future of continuous delivery; we control what the users see through LaunchDarkly interface, completely de-coupling deployment from feature delivery**.
    
    ## Setup environment variables
    
    ### Gather the values from the LD web app
    
    In preparation for the test section of this guide, we gather all the environment variables we need from the LD interface.
    
    We get the project key (`pizza-api-example`) and the SDK key from the Projects tab.
    
    ![Projects tab](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6tzpfp4tq6xu3nklreqv.png)
    
    We create an Auth token for our api at Authorization tab. It needs to be an Admin token. We can name it the same as the project; `pizza-api-example`.
    
    ![Project token](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qfoxgxfzsi56w25txpxg.png)
    
    ### Local env vars and `process.env`
    
    We can use [dotenv](https://www.npmjs.com/package/dotenv) to have access to `process.env` in our Node code. `npm i dotenv` and create a gitignored `.env` file in the root of your project. Note that `dotenv` has to be a project dependency because we need it in the source code.
    
    Per convention, we can create a `.env.example` file in the root, and that should communicate to repo users that they need an `.env` file with real values in place of wildcards.
    
    LAUNCHDARKLY_SDK_KEY=sdk-***  
    LAUNCH_DARKLY_PROJECT_KEY=pizza-api-example  
    LAUNCH_DARKLY_AUTH_TOKEN=api-***
    
    > In this example, we are testing on only one environment; `Test`. In the real world we have many environments. Each environment has its unique `LAUNCHDARKLY_SDK_KEY`. When such is the case, so that we can interrogate the flag state in any deployment, we can either [Set All Cypress Env Values Using A Single GitHub Actions Secret](https://glebbahmutov.com/blog/secrets-to-env/), or we can configure our config files per deployment to contain a variable per environment. This will be shown in part 2 of the series.
    
    ### Lambda env vars
    
    Navigate to our lambda function in AWS > Configuration > Environment variables and add `LAUNCHDARKLY_SDK_KEY`. This is the only environment variable that gets used in the code. The trio of environment variables get used in the tests and will be needed later in the `.env` file, Github settings and yml file for the pipeline.
    
    ![Lambda env vars](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/suchvk1lncv2053wjyk3.png)
    
    Now we can update our two handler files that are using the SDK key. In order to use `dotenv` and gain access to `process.env`, all we need is to require it.
    
    > After the guide was completed, we made some additional enhancements in favor or stateless testing with services.
    
    // ./handlers/get-ld-flag-value.js  
    ​  
    const ld = require("launchdarkly-node-server-sdk");  
    require("dotenv").config();  
    ​  
    /**  
     * 1. Initializes the LD client & waits for the initialization to complete.  
     * 2. Gets the flag value using the LD client.  
     * 3. If a user is not provided while getting the flag value, populates an anonymous user generic users.  
     * 4. The code calling the LD client cannot be observed by any other part of the application.  
     */  
    export const getLDFlagValue = (function () {  
      /** Handles the initialization using the SDK key,  
       * which is available on the account settings in the LaunchDarkly dashboard.  
       * Once the client is initialized, getClient() returns it. */  
      async function getClient() {  
        const client = ld.init(process.env.LAUNCHDARKLY_SDK_KEY);  
        await client.waitForInitialization();  
        return client;  
      }  
    ​  
      /** A generic wrapper around the client's variation() method used get a flag's current value  
       * Initializes the client  
       * Populates an anonymous user key if one is not provided, to handle generic users. */  
      async function flagValue(key, user, defaultValue = false) {  
        // We want a unique LD client instance with every call to ensure stateless assertions  
        // otherwise our back to back flag assertions would result in a cached value vs the current  
        const ldClient = await getClient();  
    ​  
        if (!user) {  
          user = {  
            key: "anonymous",  
          };  
        }  
    ​  
        const flagValue = await ldClient.variation(key, user, defaultValue);  
          
        // we add some logging to make testing easier later  
        console.log(  
          `**LDclient** flag: ${key} user.key: ${user.key} value: ${flagValue}`  
        );  
        return flagValue;  
      }  
    ​  
      return flagValue;  
    })();  
    ​  
    module.exports = getLDFlagValue;
    
    In case you still want to keep the sanity test in `get-orders` handler, update that too.
    
    // ./handlers/get-orders.js  
    ​  
    // ... other imports ...  
    const ld = require("launchdarkly-node-server-sdk");  
    require("dotenv").config();  
    ​  
    const ldClient = ld.init(process.env.LAUNCHDARKLY_SDK_KEY);
    
    As usual, we deploy our code with `npm run update`, set the flag value at LD interface, send a request with rest client and observe the results at CloudWatch. Toggle the flag and repeat the test to ensure basic sanity.
    
    ## Summary
    
    In this guide we covered LaunchDarkly Feature Flag setup for Node lambda functions. We created a project and a boolean feature flag at the LD interface. We showcased preferred best practices setting up and using `launchdarkly-node-server-sdk` in a lambda. Finally we demoed a fully working example for a mid sized service and provided reproducible source code.
    
    In the next section we will explore how to test our service while it is being controlled by feature flags.
    
    ## References
    
    -   [https://docs.launchdarkly.com/sdk/server-side/node-js](https://docs.launchdarkly.com/sdk/server-side/node-js)
        
    -   [https://docs.launchdarkly.com/guides/platform-specific/aws-lambda/?q=lambda](https://docs.launchdarkly.com/guides/platform-specific/aws-lambda/?q=lambda)
        
    -   [https://launchdarkly.com/blog/using-launchdarkly-with-typescript/](https://launchdarkly.com/blog/using-launchdarkly-with-typescript/)
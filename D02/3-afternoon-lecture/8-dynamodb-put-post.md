# POST & PUT Data To DynamoDB

<!-- <img src="https://i.imgur.com/Wmp6bps.png" width=700/>  -->



This lecture will focus on posting new items and update existing items in DynamoDB via the **POST/PUT** routes.  We will focus on the following:

- updating the **project-create** Lambda function to create new items in DynamoDB
- updating the **project-show** Lambda function to update existing items in DynamoDB

### POST Route

Let's open the **projects-create** Lambda function and import the SDK and connect to DynamoDB. 

```js
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB({region: 'us-east-1', apiVersion: '2012-08-10'});
```


#### The putItem Method

The method we will use to create new items is **putItem**.

HTTP | Resource  | CRUD Operation | Lambda | DynamoDB | Has Data
-----------|------------------|------------------|:---:|:---:|:---:
PUT     | /projects/:id      | Update specified _project_  | projects-update | updateItem | Yes


 If we do a quick search for **putItem** in the [AWS Docs](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html) we should see the following:

<img src="https://i.imgur.com/nWqd2by.png">

Clicking on it should take us to the details section on how to use the method.

We can see that the **putItem** method takes in a **params** object and a **callback** function. This is identical to the **scan** and **getItem** methods. 

<img src="https://i.imgur.com/dygqJ9T.png">

Let's first start with creating the **params** object.  Since DynamoDB doesn't take on the responsibility for creating unique values for the primary keys we will need to make sure every entry is unique.  

The data structure is very similar to what was returned via **scan** and **getItem** and this will be the same pattern needed to create an item. 

For this we will use a combo of  `project_${Math.random()}`.  

```js
const params = {
    Item: {
        "ProjectId": {
            S: `project_${Math.random()}`
        },
        "Title": {
            S: event.title
        },
        "Image": {
            S: event.image
        },
        "Description": {
            S: event.description
        }
    },
    TableName: "projects"
};  
```

Let's pass the param to **dynamodb.putItem** and set it up for **await** and **.promise()**.

```js
 try {
    const data = await dynamodb.putItem(params).promise()
    console.log('data', data)

    response.body = data

} catch(err) {
    response.body = err
}
```
<details><summary>Solution Code</summary>

```js
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB({region: 'us-east-1', apiVersion: '2012-08-10'});

exports.handler = async (event) => {
    console.log('event', event)
    
    const response = {
        statusCode: 200,
    }
    
    const params = {
        Item: {
            "ProjectId": {
                S: event.id
            },
            "Title": {
                S: event.title
            },
            "Image": {
                S: event.image
            },
            "Description": {
                S: event.description
            }
        },
        TableName: "projects"
    };
    
    try {
        const data = await dynamodb.putItem(params).promise()
        console.log('data', data)
        
        response.body = params
        
    } catch(err) {
        response.body = err
    }

    return response
};
```

</details>

If we test this we will find that **putItem** doesn't return the newly created item so we opt to pass back the **params** object since it contains all the values that now exist in the DB. 

```js
response.body = params
```

Let's create a new Lambda test called **createItem** and use the following object: 

```js
  {
    "title": "Newly Added Project",
    "image": "https://i.imgur.com/L9K6hli.png",
    "description": "Add project description here..."
}
```

Once we run the test we should receive the following error message. 

<img src="https://i.imgur.com/wia5svN.png">

#### Assigning Lambda Permissions To DynamoDB

This is the same error we received with **scan** and **getItem** which means we need to assign the Lambda role permissions to DynamoDB via **IAM**.  


Take a moment to open **IAM** and find the **projects-create-xxx** role. It should look something like the below: 

<img src="https://i.imgur.com/5xN96Fp.png">

Click on **Attach Policies** and do a search for **dynamo**.  The policy we are going to attach is **AmazonDynamoDBFullAccess** as this is the role we will need to create items in the DB. 

<img src="https://i.imgur.com/Wpdigql.png">

Attach the policy and let's run the Lambda test again. We should see the following:


<img src="https://i.imgur.com/BCSLnfx.png">

Also confirm that the item has been created in our DynamoDB project table. 

**REPLACE THIS IMAGE**
<img src="https://i.imgur.com/tzky8Ne.png">



#### Testing The POST Route

Let's also take a moment to test via the **POST** route in the API Gateway.  Pass it the same object as before and confirm that the same data has been returned, with only the projectId being different. 

**UPDATE THIS IMAGE**
<img src="https://i.imgur.com/H963ZLv.png">


### Formating Response Data

This isn't the best format to return via the API so let's reformat it in the **Integration Response** using a **Mapping Template**.  

Create a new **Mapping Template** and assign it a **Content Type** of **application/json**. 

<img src="https://i.imgur.com/p5i3suH.png">

#### AWS Documentation

It's as this point we need to take a look at the [AWS Documentation](https://docs.amazonaws.cn/en_us/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html#input-variable-reference) regarding what this involves.  

If we take a look specifically at **input variables** we can see that **$input.json(x)** returns the results as a JSON string.  


<img src="https://i.imgur.com/3iqLYvD.png">

Here is how we will use **$input.json()** to format the data. 

```js
{
 "statusCode": $input.json('$.statusCode'),
 "body": {
    "projectId": $input.json('$.body.Item.ProjectId.S')
    "title": $input.json('$.body.Item.Title.S')
    "image": $input.json('$.body.Item.Image.S')
    "description": $input.json('$.body.Item.Description.S')
 }
}
```

Copy and paste the above code into the template body. 

<img src="https://i.imgur.com/s1qr6yD.png">

### Testing via API Gateway and Postman

Let's make sure to give it one final test via the **POST** route and confirm the data is formatted as expected. 

<img src="https://i.imgur.com/VWPHBgL.png">

#### Redeploy And Test The POST Route Using Postman

Once that has been confirmed re-deploy the API and test it via Postman.  If successful we should see the following:

<img src="https://i.imgur.com/Lp6gOrt.png">





### PUT Route

With our **POST** route working let's now configure the **PUT** route. Let's recap the steps we performed for the **POST** route as this requires that we do much of the same:

- import the **AWS SDK** and connect to DynamoDB
- Find the right method to use for updating an item (hint: updateItem)
- Update the **Integration Response** to format the response data
- Update the **Integration Request** to include the data model


Let's open the **projects-update** Lambda function and import the SDK and connect to DynamoDB. 

```js
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB({region: 'us-east-1', apiVersion: '2012-08-10'});
```


#### The updateItem Method

The method we will use to update existing items is **updateItem**. If we do a quick search for **updateItem** in the [AWS Docs](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html) we should see the following:

<img src="https://i.imgur.com/LGFxdSr.png">

Clicking on it should take us to the details section on how to use the method.

We can see that the **updateItem** method takes in a **params** object and a **callback** function although the **params** object contains quite a few more keys. 

<img src="https://i.imgur.com/JIc8OYa.png">

Based on how it is being used this is how we should structure it to work with our requirements. 

```js
 const params = {
    // the keys to be updated
    ExpressionAttributeNames: {
        "#T": "Title", 
        "#I": "Image",
        "#D": "Description"
    }, 
    // the values to assign to the above keys
    ExpressionAttributeValues: {
        ":t": { S: event.title }, 
        ":i": { S: event.image },
        ":d": { S: event.description}
    }, 
    // this is the primary key to target
    Key: { "ProjectId": { S: event.id } }, 
    // return all new values
    ReturnValues: "ALL_NEW", 
    // table name to target
    TableName: "projects", 
    // matches the keys to values
    UpdateExpression: "SET #T = :t, #I = :i, #D = :d"
 };
    
```

Now we can configure the method. Since this method will return the new value we can set **reponse.body** to the returned item. 

```js
try {
    const data = await dynamodb.updateItem(params).promise()
    console.log('data', data)
    
    response.body = data
    
} catch(err) {
    response.body = err
}

return response
```

Make sure to tests this via the Lambda funcion. 

<details><summary>Solution Code</summary>



```js

const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB({region: 'us-east-1', apiVersion: '2012-08-10'});

exports.handler = async (event) => {
    
 const response = {
        statusCode: 200,
    }

 const params = {
  ExpressionAttributeNames: {
    "#T": "Title", 
    "#I": "Image",
    "#D": "Description"
  }, 
  ExpressionAttributeValues: {
    ":t": { S: event.title }, 
    ":i": { S: event.image },
    ":d": { S: event.description}
  }, 
    Key: { "ProjectId": { S: event.id } }, 
    ReturnValues: "ALL_NEW", 
    TableName: "projects", 
    UpdateExpression: "SET #T = :t, #I = :i, #D = :d"
 };
    
  try {
    const data = await dynamodb.updateItem(params).promise()
    console.log('data', data)
    
    response.body = data
        
    } catch(err) {
        response.body = err
  }

  return response
};
```
</details>

#### Integration Response 

If we take a look at the **AWS Gateway** logs for this function we should see the following:

<img src="https://i.imgur.com/ckJOpds.png">

This provides us the content we ned to setup the **Integration Response** for the **PUT** route. 

```js
{
 "statusCode": $input.json('$.statusCode'),
 "body": {
    "projectId": $input.json('$.body.Attributes.ProjectId.S')
    "title": $input.json('$.body.Attributes.Title.S')
    "image": $input.json('$.body.Attributes.Image.S')
    "description": $input.json('$.body.Attributes.Description.S')
 }
}
```

Copy and paste the above code into the template body. 

<img src="https://i.imgur.com/LPan9aG.png">

### Testing via API Gateway and Postman

Let's make sure to give it one final test via the **PUT** route and confirm the data is formatted as expected. 

We will need to grab a **projectId** from one of the existing items along with passing it the updated data via the **Request Body**

```js
  {
    "title": "Test",
    "image": "https://i.imgur.com/L9K6hli.png",
    "description": "Add project description here..."
}
```

<img src="https://i.imgur.com/nf8SCRK.png">

#### Redeploy And Test The POST Route Using Postman

Once that has been confirmed re-deploy the API now test it via Postman.  If successful we should see the following:

<img src="https://i.imgur.com/Lp6gOrt.png">



<!-- <img src="https://i.imgur.com/s1qr6yD.png"> -->

  ### Resources

  - [DynamoDB Services Page](https://aws.amazon.com/dynamodb/)
  - [AWS SDK](https://aws.amazon.com/sdk-for-javascript/)
  - [Async/Await Lambda](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/using-async-await.html)

```js
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB({region: 'us-east-1', apiVersion: '2012-08-10'});
const projects = require('projectData')

exports.handler = async (event) => {
    // TODO implement
    const response = {
        statusCode: 200,
    };
    
    let seedData = projects.map( project => {
        return {
            PutRequest: {
                Item: {
                   "ProjectId": { S: `project_${Math.random()}` },
                    "Title": { S: project.title }, 
                    "Image": { S: project.image }, 
                    "Description": { S: project.description }
                }
            }
        }
    })
    
    let params = { 
        RequestItems: { "projects": seedData }
    };
    
    try{
      await dynamodb.batchWriteItem(params).promise()
    } catch(err) {
      console.log('err', err)
    }
    
    try {
      const data = await dynamodb.scan({"TableName": "projects"}).promise()  
      const items = data.Items.map( (data,index) => {
        return {
            projectId: data.ProjectId.S,
            title: data.Title.S,
            image: data.Image.S,
            description: data.Description.S
        }
      })
        response.body = items
    } catch(err) {
        response.body = err
    }
 
    return response;
};

```
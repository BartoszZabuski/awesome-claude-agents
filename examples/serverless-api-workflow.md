# Serverless API Workflow Example

This example demonstrates how the Node.js Lambda Architect and AWS CDK Architect work together to build a complete serverless application.

## Scenario: Building a Serverless Order Processing System

The user wants to create a serverless API for processing e-commerce orders with the following requirements:
- RESTful API endpoints for order CRUD operations
- Event-driven order processing with SQS
- DynamoDB for data storage
- Email notifications via SNS
- Complete CDK infrastructure

## Workflow Execution

### Step 1: Tech Lead Orchestrator Analysis

```yaml
User: "Build a serverless order processing API with Lambda and CDK"

Tech Lead Analysis:
- Detected: Serverless architecture needed
- Runtime: Node.js/TypeScript preferred
- Infrastructure: AWS CDK
- Services: Lambda, API Gateway, DynamoDB, SQS, SNS

Team Assembly:
1. nodejs-lambda-architect - For Lambda functions
2. aws-cdk-architect - For infrastructure
```

### Step 2: Lambda Architect Creates Functions

The **nodejs-lambda-architect** creates the Lambda functions:

```typescript
// src/handlers/orders/create-order.ts
import { APIGatewayProxyHandler } from 'aws-lambda';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';
import { SQSClient, SendMessageCommand } from '@aws-sdk/client-sqs';
import { randomUUID } from 'crypto';

const ddbClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const sqsClient = new SQSClient({});

export const handler: APIGatewayProxyHandler = async (event) => {
  const order = JSON.parse(event.body || '{}');
  const orderId = randomUUID();
  
  // Store order in DynamoDB
  await ddbClient.send(new PutCommand({
    TableName: process.env.ORDERS_TABLE!,
    Item: {
      orderId,
      customerId: order.customerId,
      items: order.items,
      total: order.total,
      status: 'PENDING',
      createdAt: new Date().toISOString(),
    },
  }));
  
  // Send to processing queue
  await sqsClient.send(new SendMessageCommand({
    QueueUrl: process.env.PROCESSING_QUEUE_URL!,
    MessageBody: JSON.stringify({ orderId, action: 'PROCESS_ORDER' }),
  }));
  
  return {
    statusCode: 201,
    body: JSON.stringify({ orderId, status: 'PENDING' }),
  };
};
```

```typescript
// src/handlers/orders/process-order.ts
import { SQSHandler } from 'aws-lambda';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, UpdateCommand } from '@aws-sdk/lib-dynamodb';
import { SNSClient, PublishCommand } from '@aws-sdk/client-sns';

const ddbClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const snsClient = new SNSClient({});

export const handler: SQSHandler = async (event) => {
  for (const record of event.Records) {
    const { orderId } = JSON.parse(record.body);
    
    try {
      // Process payment (simplified)
      await processPayment(orderId);
      
      // Update order status
      await ddbClient.send(new UpdateCommand({
        TableName: process.env.ORDERS_TABLE!,
        Key: { orderId },
        UpdateExpression: 'SET #status = :status, processedAt = :timestamp',
        ExpressionAttributeNames: { '#status': 'status' },
        ExpressionAttributeValues: {
          ':status': 'COMPLETED',
          ':timestamp': new Date().toISOString(),
        },
      }));
      
      // Send notification
      await snsClient.send(new PublishCommand({
        TopicArn: process.env.NOTIFICATION_TOPIC_ARN!,
        Subject: 'Order Processed',
        Message: `Order ${orderId} has been successfully processed.`,
      }));
    } catch (error) {
      // Update status to failed
      await ddbClient.send(new UpdateCommand({
        TableName: process.env.ORDERS_TABLE!,
        Key: { orderId },
        UpdateExpression: 'SET #status = :status',
        ExpressionAttributeNames: { '#status': 'status' },
        ExpressionAttributeValues: { ':status': 'FAILED' },
      }));
      
      throw error; // Let SQS retry
    }
  }
};
```

### Step 3: CDK Architect Creates Infrastructure

The **aws-cdk-architect** creates the complete infrastructure:

```typescript
// lib/order-processing-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subscriptions from 'aws-cdk-lib/aws-sns-subscriptions';
import { SqsEventSource } from 'aws-cdk-lib/aws-lambda-event-sources';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import { Construct } from 'constructs';

export class OrderProcessingStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // DynamoDB table for orders
    const ordersTable = new dynamodb.Table(this, 'OrdersTable', {
      partitionKey: { name: 'orderId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
    });

    // SQS queue for order processing
    const processingQueue = new sqs.Queue(this, 'ProcessingQueue', {
      visibilityTimeout: cdk.Duration.minutes(5),
      deadLetterQueue: {
        maxReceiveCount: 3,
        queue: new sqs.Queue(this, 'ProcessingDLQ'),
      },
    });

    // SNS topic for notifications
    const notificationTopic = new sns.Topic(this, 'NotificationTopic');
    notificationTopic.addSubscription(
      new subscriptions.EmailSubscription('orders@example.com')
    );

    // Lambda function for creating orders
    const createOrderFn = new NodejsFunction(this, 'CreateOrderFunction', {
      entry: 'src/handlers/orders/create-order.ts',
      runtime: lambda.Runtime.NODEJS_18_X,
      environment: {
        ORDERS_TABLE: ordersTable.tableName,
        PROCESSING_QUEUE_URL: processingQueue.queueUrl,
      },
      timeout: cdk.Duration.seconds(30),
    });

    // Lambda function for processing orders
    const processOrderFn = new NodejsFunction(this, 'ProcessOrderFunction', {
      entry: 'src/handlers/orders/process-order.ts',
      runtime: lambda.Runtime.NODEJS_18_X,
      environment: {
        ORDERS_TABLE: ordersTable.tableName,
        NOTIFICATION_TOPIC_ARN: notificationTopic.topicArn,
      },
      timeout: cdk.Duration.minutes(5),
      reservedConcurrentExecutions: 10,
    });

    // Add SQS event source
    processOrderFn.addEventSource(new SqsEventSource(processingQueue, {
      batchSize: 10,
      maxBatchingWindowInMs: 1000,
    }));

    // Grant permissions
    ordersTable.grantReadWriteData(createOrderFn);
    ordersTable.grantReadWriteData(processOrderFn);
    processingQueue.grantSendMessages(createOrderFn);
    processingQueue.grantConsumeMessages(processOrderFn);
    notificationTopic.grantPublish(processOrderFn);

    // API Gateway
    const api = new apigateway.RestApi(this, 'OrdersApi', {
      restApiName: 'Order Processing API',
      deployOptions: {
        tracingEnabled: true,
        metricsEnabled: true,
      },
    });

    // Add endpoints
    const orders = api.root.addResource('orders');
    orders.addMethod('POST', new apigateway.LambdaIntegration(createOrderFn));

    // Lambda for GET endpoint
    const getOrderFn = new NodejsFunction(this, 'GetOrderFunction', {
      entry: 'src/handlers/orders/get-order.ts',
      runtime: lambda.Runtime.NODEJS_18_X,
      environment: {
        ORDERS_TABLE: ordersTable.tableName,
      },
    });
    
    ordersTable.grantReadData(getOrderFn);
    
    const orderIdResource = orders.addResource('{orderId}');
    orderIdResource.addMethod('GET', new apigateway.LambdaIntegration(getOrderFn));

    // Outputs
    new cdk.CfnOutput(this, 'ApiUrl', {
      value: api.url,
      description: 'Order Processing API URL',
    });
  }
}

// bin/app.ts
#!/usr/bin/env node
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { OrderProcessingStack } from '../lib/order-processing-stack';

const app = new cdk.App();

new OrderProcessingStack(app, 'OrderProcessingStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
});
```

### Step 4: CDK Architect Adds Monitoring

```typescript
// lib/constructs/monitoring-construct.ts
import * as cdk from 'aws-cdk-lib';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as sns from 'aws-cdk-lib/aws-sns';
import { Construct } from 'constructs';

export interface MonitoringProps {
  functions: lambda.IFunction[];
  alarmTopic: sns.ITopic;
}

export class MonitoringConstruct extends Construct {
  constructor(scope: Construct, id: string, props: MonitoringProps) {
    super(scope, id);

    // Create dashboard
    const dashboard = new cloudwatch.Dashboard(this, 'OrdersDashboard', {
      dashboardName: 'order-processing-dashboard',
    });

    // Add widgets for each function
    props.functions.forEach((fn) => {
      // Error rate alarm
      new cloudwatch.Alarm(this, `${fn.node.id}ErrorAlarm`, {
        metric: fn.metricErrors(),
        threshold: 10,
        evaluationPeriods: 2,
        alarmDescription: `High error rate for ${fn.functionName}`,
      }).addAlarmAction(new cw_actions.SnsAction(props.alarmTopic));

      // Add to dashboard
      dashboard.addWidgets(
        new cloudwatch.GraphWidget({
          title: `${fn.functionName} Metrics`,
          left: [fn.metricInvocations()],
          right: [fn.metricErrors(), fn.metricThrottles()],
        })
      );
    });
  }
}
```

## Collaboration Benefits

### 1. **Separation of Concerns**
- Lambda Architect focuses on business logic and Lambda best practices
- CDK Architect handles infrastructure, security, and AWS service integration

### 2. **Optimized Code**
- Lambda functions are optimized for cold starts and performance
- Infrastructure is designed for scalability and cost efficiency

### 3. **Best Practices Applied**
- TypeScript for type safety
- Proper error handling and retries
- Monitoring and alerting built-in
- Security with least-privilege IAM

### 4. **Production Ready**
- Complete solution with all necessary components
- Follows AWS Well-Architected Framework
- Ready for deployment with `cdk deploy`

## Testing the Solution

```bash
# Deploy the stack
npm install
npm run build
cdk deploy

# Test the API
curl -X POST https://[api-id].execute-api.[region].amazonaws.com/prod/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "123",
    "items": [{"productId": "abc", "quantity": 2}],
    "total": 99.99
  }'
```

## Summary

This workflow demonstrates how specialized agents collaborate:
1. Tech Lead Orchestrator identifies the need for serverless expertise
2. Node.js Lambda Architect creates optimized Lambda functions
3. AWS CDK Architect builds complete infrastructure
4. Both agents ensure their components integrate seamlessly
5. Result is production-ready serverless application

The collaboration ensures both business logic and infrastructure follow best practices for their respective domains.
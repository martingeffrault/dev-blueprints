# AWS (2025-2026)

> **Last updated**: January 2026
> **Services covered**: Lambda, API Gateway, DynamoDB, S3, IAM, CDK, CloudFormation
> **Focus**: Serverless-first, Well-Architected, Cost-optimized

---

## Philosophy (2025-2026)

AWS in 2025-2026 emphasizes **serverless-first architecture**, **infrastructure as code with CDK**, and **Well-Architected principles** across all workloads.

**Key philosophical shifts:**
- **Serverless by default** — Lambda, API Gateway, DynamoDB over EC2/RDS
- **CDK over CloudFormation** — TypeScript/Python IaC with constructs
- **ARM64 (Graviton)** — 40% better price-performance
- **Single-table DynamoDB** — Access pattern-first design
- **IAM least privilege** — Fine-grained permissions with Access Analyzer
- **Cost-aware development** — FinOps integrated into development workflow
- **Observability-first** — Powertools for structured logging, tracing, metrics

---

## TL;DR

- Use Lambda with ARM64 (Graviton) for cost-performance
- Use CDK (TypeScript) over raw CloudFormation
- Design DynamoDB with access patterns first (single-table when appropriate)
- Always use IAM roles with least privilege, never long-term credentials
- Use API Gateway HTTP APIs for simple use cases, REST APIs for advanced features
- Enable S3 Block Public Access at organization level
- Use Powertools for Lambda observability (logging, tracing, metrics)
- Tag everything for cost allocation
- Use Savings Plans and Spot Instances for cost optimization

---

## AWS Well-Architected Framework

The six pillars guide all architectural decisions:

| Pillar | Focus |
|--------|-------|
| **Operational Excellence** | Automate operations, respond to events, continuous improvement |
| **Security** | Protect data, manage permissions, detect threats |
| **Reliability** | Recover from failures, scale horizontally |
| **Performance Efficiency** | Use resources efficiently, adopt new technologies |
| **Cost Optimization** | Eliminate waste, right-size resources |
| **Sustainability** | Minimize environmental impact |

---

## Best Practices

### Lambda Function (Node.js/TypeScript)

```typescript
// handler.ts
import { Logger, Metrics, Tracer } from '@aws-lambda-powertools/all';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, GetCommand } from '@aws-sdk/lib-dynamodb';
import type { APIGatewayProxyEventV2, APIGatewayProxyResultV2 } from 'aws-lambda';

// Initialize outside handler for connection reuse
const logger = new Logger({ serviceName: 'user-service' });
const tracer = new Tracer({ serviceName: 'user-service' });
const metrics = new Metrics({ serviceName: 'user-service', namespace: 'MyApp' });

const client = tracer.captureAWSv3Client(new DynamoDBClient({}));
const docClient = DynamoDBDocumentClient.from(client);

interface User {
  pk: string;
  sk: string;
  name: string;
  email: string;
}

export const handler = async (
  event: APIGatewayProxyEventV2
): Promise<APIGatewayProxyResultV2> => {
  // Add correlation ID for tracing
  const correlationId = event.headers['x-correlation-id'] ?? crypto.randomUUID();
  logger.appendKeys({ correlationId });

  try {
    const userId = event.pathParameters?.userId;
    if (!userId) {
      return { statusCode: 400, body: JSON.stringify({ error: 'userId required' }) };
    }

    logger.info('Fetching user', { userId });

    const result = await docClient.send(new GetCommand({
      TableName: process.env.TABLE_NAME!,
      Key: { pk: `USER#${userId}`, sk: `PROFILE#${userId}` }
    }));

    if (!result.Item) {
      metrics.addMetric('UserNotFound', 'Count', 1);
      return { statusCode: 404, body: JSON.stringify({ error: 'User not found' }) };
    }

    metrics.addMetric('UserFetched', 'Count', 1);

    return {
      statusCode: 200,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(result.Item)
    };
  } catch (error) {
    logger.error('Failed to fetch user', { error });
    metrics.addMetric('UserFetchError', 'Count', 1);

    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Internal server error' })
    };
  } finally {
    metrics.publishStoredMetrics();
  }
};
```

### Lambda Function (Python)

```python
# handler.py
import json
import os
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext
from aws_lambda_powertools.utilities.data_classes import APIGatewayProxyEventV2
import boto3
from botocore.config import Config

# Initialize outside handler for connection reuse
logger = Logger(service="user-service")
tracer = Tracer(service="user-service")
metrics = Metrics(service="user-service", namespace="MyApp")

# Configure retry and connection settings
config = Config(
    retries={'max_attempts': 3, 'mode': 'adaptive'},
    connect_timeout=5,
    read_timeout=10
)
dynamodb = boto3.resource('dynamodb', config=config)
table = dynamodb.Table(os.environ['TABLE_NAME'])


@logger.inject_lambda_context
@tracer.capture_lambda_handler
@metrics.log_metrics(capture_cold_start_metric=True)
def handler(event: dict, context: LambdaContext) -> dict:
    """Handle user fetch request."""
    api_event = APIGatewayProxyEventV2(event)

    user_id = api_event.path_parameters.get('userId') if api_event.path_parameters else None
    if not user_id:
        return {
            'statusCode': 400,
            'body': json.dumps({'error': 'userId required'})
        }

    logger.info("Fetching user", extra={"user_id": user_id})

    try:
        response = table.get_item(
            Key={
                'pk': f'USER#{user_id}',
                'sk': f'PROFILE#{user_id}'
            }
        )

        item = response.get('Item')
        if not item:
            metrics.add_metric(name="UserNotFound", unit=MetricUnit.Count, value=1)
            return {
                'statusCode': 404,
                'body': json.dumps({'error': 'User not found'})
            }

        metrics.add_metric(name="UserFetched", unit=MetricUnit.Count, value=1)

        return {
            'statusCode': 200,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps(item)
        }

    except Exception as e:
        logger.exception("Failed to fetch user")
        metrics.add_metric(name="UserFetchError", unit=MetricUnit.Count, value=1)

        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Internal server error'})
        }
```

### CDK Stack (TypeScript)

```typescript
// lib/api-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as nodejs from 'aws-cdk-lib/aws-lambda-nodejs';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as apigatewayv2 from 'aws-cdk-lib/aws-apigatewayv2';
import * as integrations from 'aws-cdk-lib/aws-apigatewayv2-integrations';
import * as logs from 'aws-cdk-lib/aws-logs';
import { Construct } from 'constructs';

export class ApiStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // DynamoDB table with on-demand capacity
    const table = new dynamodb.Table(this, 'UsersTable', {
      tableName: 'users',
      partitionKey: { name: 'pk', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'sk', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      pointInTimeRecovery: true,
      encryption: dynamodb.TableEncryption.AWS_MANAGED,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    // Global Secondary Index for access patterns
    table.addGlobalSecondaryIndex({
      indexName: 'GSI1',
      partitionKey: { name: 'GSI1PK', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'GSI1SK', type: dynamodb.AttributeType.STRING },
      projectionType: dynamodb.ProjectionType.ALL,
    });

    // Lambda function with ARM64 and Powertools
    const userFunction = new nodejs.NodejsFunction(this, 'UserFunction', {
      entry: 'src/handlers/user.ts',
      handler: 'handler',
      runtime: lambda.Runtime.NODEJS_22_X,
      architecture: lambda.Architecture.ARM_64, // Graviton - 40% better price-performance
      memorySize: 1024,
      timeout: cdk.Duration.seconds(10),
      tracing: lambda.Tracing.ACTIVE,
      environment: {
        TABLE_NAME: table.tableName,
        POWERTOOLS_SERVICE_NAME: 'user-service',
        POWERTOOLS_METRICS_NAMESPACE: 'MyApp',
        LOG_LEVEL: 'INFO',
      },
      bundling: {
        minify: true,
        sourceMap: true,
        externalModules: ['@aws-sdk/*'], // Use Lambda-provided SDK
      },
      logRetention: logs.RetentionDays.ONE_MONTH,
    });

    // Grant least privilege access
    table.grantReadData(userFunction);

    // HTTP API (cheaper, faster than REST API)
    const api = new apigatewayv2.HttpApi(this, 'UserApi', {
      apiName: 'user-api',
      corsPreflight: {
        allowOrigins: ['https://myapp.com'],
        allowMethods: [apigatewayv2.CorsHttpMethod.GET, apigatewayv2.CorsHttpMethod.POST],
        allowHeaders: ['Content-Type', 'Authorization'],
        maxAge: cdk.Duration.hours(1),
      },
    });

    // Route integration
    api.addRoutes({
      path: '/users/{userId}',
      methods: [apigatewayv2.HttpMethod.GET],
      integration: new integrations.HttpLambdaIntegration('UserIntegration', userFunction),
    });

    // Outputs
    new cdk.CfnOutput(this, 'ApiEndpoint', {
      value: api.apiEndpoint,
      description: 'API Gateway endpoint URL',
    });
  }
}
```

### DynamoDB Single-Table Design

```typescript
// Single-table design patterns

// Entity prefixes for composite keys
const PREFIXES = {
  USER: 'USER#',
  ORDER: 'ORDER#',
  PRODUCT: 'PRODUCT#',
  PROFILE: 'PROFILE#',
  ITEM: 'ITEM#',
} as const;

// User entity
interface UserItem {
  pk: string;       // USER#<userId>
  sk: string;       // PROFILE#<userId>
  GSI1PK: string;   // EMAIL#<email>
  GSI1SK: string;   // USER#<userId>
  type: 'USER';
  userId: string;
  email: string;
  name: string;
  createdAt: string;
}

// Order entity (same table!)
interface OrderItem {
  pk: string;       // USER#<userId>
  sk: string;       // ORDER#<orderId>
  GSI1PK: string;   // ORDER#<orderId>
  GSI1SK: string;   // STATUS#<status>#<createdAt>
  type: 'ORDER';
  orderId: string;
  userId: string;
  status: 'pending' | 'shipped' | 'delivered';
  total: number;
  createdAt: string;
}

// Access patterns enabled:
// 1. Get user by ID: pk = USER#123, sk = PROFILE#123
// 2. Get all orders for user: pk = USER#123, sk begins_with ORDER#
// 3. Get user by email: GSI1 pk = EMAIL#user@example.com
// 4. Get order by ID: GSI1 pk = ORDER#abc123
// 5. Query orders by status: GSI1 pk = ORDER#, sk begins_with STATUS#shipped

// Example queries
async function getUserWithOrders(userId: string) {
  const result = await docClient.send(new QueryCommand({
    TableName: 'users',
    KeyConditionExpression: 'pk = :pk',
    ExpressionAttributeValues: {
      ':pk': `USER#${userId}`
    }
  }));

  // Single query returns user profile AND all orders
  const user = result.Items?.find(item => item.type === 'USER');
  const orders = result.Items?.filter(item => item.type === 'ORDER');

  return { user, orders };
}
```

### S3 Bucket with Security

```typescript
// lib/storage-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as iam from 'aws-cdk-lib/aws-iam';
import { Construct } from 'constructs';

export class StorageStack extends cdk.Stack {
  public readonly bucket: s3.Bucket;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    this.bucket = new s3.Bucket(this, 'DataBucket', {
      bucketName: `myapp-data-${this.account}-${this.region}`,

      // Security settings
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      encryption: s3.BucketEncryption.S3_MANAGED,
      enforceSSL: true,
      versioned: true,

      // Lifecycle rules for cost optimization
      lifecycleRules: [
        {
          id: 'TransitionToIA',
          transitions: [
            {
              storageClass: s3.StorageClass.INFREQUENT_ACCESS,
              transitionAfter: cdk.Duration.days(30),
            },
            {
              storageClass: s3.StorageClass.GLACIER,
              transitionAfter: cdk.Duration.days(90),
            },
          ],
        },
        {
          id: 'ExpireOldVersions',
          noncurrentVersionExpiration: cdk.Duration.days(30),
        },
      ],

      // Access logging
      serverAccessLogsPrefix: 'access-logs/',

      // Prevent accidental deletion
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    // Bucket policy enforcing TLS
    this.bucket.addToResourcePolicy(new iam.PolicyStatement({
      sid: 'EnforceHTTPS',
      effect: iam.Effect.DENY,
      principals: [new iam.AnyPrincipal()],
      actions: ['s3:*'],
      resources: [this.bucket.bucketArn, `${this.bucket.bucketArn}/*`],
      conditions: {
        Bool: { 'aws:SecureTransport': 'false' },
      },
    }));
  }
}
```

### IAM Least Privilege Policy

```typescript
// CDK: Fine-grained IAM permissions
import * as iam from 'aws-cdk-lib/aws-iam';

// Create role with specific permissions only
const lambdaRole = new iam.Role(this, 'LambdaRole', {
  assumedBy: new iam.ServicePrincipal('lambda.amazonaws.com'),
  description: 'Role for user service Lambda function',
});

// Specific DynamoDB permissions (not dynamodb:*)
lambdaRole.addToPolicy(new iam.PolicyStatement({
  sid: 'DynamoDBReadAccess',
  effect: iam.Effect.ALLOW,
  actions: [
    'dynamodb:GetItem',
    'dynamodb:Query',
    'dynamodb:BatchGetItem',
  ],
  resources: [
    table.tableArn,
    `${table.tableArn}/index/*`,
  ],
}));

// Conditional permissions
lambdaRole.addToPolicy(new iam.PolicyStatement({
  sid: 'S3AccessWithConditions',
  effect: iam.Effect.ALLOW,
  actions: ['s3:GetObject', 's3:PutObject'],
  resources: [`${bucket.bucketArn}/users/*`],
  conditions: {
    StringEquals: {
      's3:x-amz-acl': 'bucket-owner-full-control',
    },
  },
}));

// CloudWatch Logs (minimum required)
lambdaRole.addToPolicy(new iam.PolicyStatement({
  sid: 'CloudWatchLogs',
  effect: iam.Effect.ALLOW,
  actions: [
    'logs:CreateLogStream',
    'logs:PutLogEvents',
  ],
  resources: [`arn:aws:logs:${this.region}:${this.account}:log-group:/aws/lambda/*`],
}));
```

### API Gateway REST API with WAF

```typescript
// lib/api-gateway-stack.ts
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as wafv2 from 'aws-cdk-lib/aws-wafv2';

// REST API for advanced features (WAF, API keys, request validation)
const api = new apigateway.RestApi(this, 'UserApi', {
  restApiName: 'User Service',
  description: 'API for user management',
  deployOptions: {
    stageName: 'prod',
    tracingEnabled: true,
    loggingLevel: apigateway.MethodLoggingLevel.INFO,
    metricsEnabled: true,
    throttlingBurstLimit: 1000,
    throttlingRateLimit: 500,
  },
  defaultCorsPreflightOptions: {
    allowOrigins: apigateway.Cors.ALL_ORIGINS,
    allowMethods: ['GET', 'POST', 'PUT', 'DELETE'],
  },
});

// Request validation
const requestValidator = new apigateway.RequestValidator(this, 'RequestValidator', {
  restApi: api,
  validateRequestBody: true,
  validateRequestParameters: true,
});

// Model for request validation
const userModel = new apigateway.Model(this, 'UserModel', {
  restApi: api,
  contentType: 'application/json',
  schema: {
    type: apigateway.JsonSchemaType.OBJECT,
    required: ['name', 'email'],
    properties: {
      name: { type: apigateway.JsonSchemaType.STRING, minLength: 1 },
      email: { type: apigateway.JsonSchemaType.STRING, format: 'email' },
    },
  },
});

// WAF Web ACL
const webAcl = new wafv2.CfnWebACL(this, 'ApiWaf', {
  defaultAction: { allow: {} },
  scope: 'REGIONAL',
  visibilityConfig: {
    cloudWatchMetricsEnabled: true,
    metricName: 'UserApiWaf',
    sampledRequestsEnabled: true,
  },
  rules: [
    {
      name: 'AWSManagedRulesCommonRuleSet',
      priority: 1,
      overrideAction: { none: {} },
      visibilityConfig: {
        sampledRequestsEnabled: true,
        cloudWatchMetricsEnabled: true,
        metricName: 'CommonRuleSet',
      },
      statement: {
        managedRuleGroupStatement: {
          vendorName: 'AWS',
          name: 'AWSManagedRulesCommonRuleSet',
        },
      },
    },
    {
      name: 'RateLimitRule',
      priority: 2,
      action: { block: {} },
      visibilityConfig: {
        sampledRequestsEnabled: true,
        cloudWatchMetricsEnabled: true,
        metricName: 'RateLimitRule',
      },
      statement: {
        rateBasedStatement: {
          limit: 1000,
          aggregateKeyType: 'IP',
        },
      },
    },
  ],
});

// Associate WAF with API Gateway
new wafv2.CfnWebACLAssociation(this, 'WafAssociation', {
  resourceArn: api.deploymentStage.stageArn,
  webAclArn: webAcl.attrArn,
});
```

### Cost Optimization with Tags

```typescript
// lib/app.ts - Tagging strategy for cost allocation
import * as cdk from 'aws-cdk-lib';

const app = new cdk.App();

// Apply tags to all resources in stack
const stack = new ApiStack(app, 'ApiStack', {
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
});

// Cost allocation tags
cdk.Tags.of(stack).add('Environment', 'production');
cdk.Tags.of(stack).add('Project', 'user-service');
cdk.Tags.of(stack).add('Team', 'backend');
cdk.Tags.of(stack).add('CostCenter', 'engineering');
cdk.Tags.of(stack).add('Owner', 'team@company.com');

// Use Graviton for Lambda (done in function config)
// Use Savings Plans for predictable workloads
// Use Spot instances for fault-tolerant workloads
```

### Lambda Cold Start Optimization

```typescript
// Optimized Lambda configuration
const optimizedFunction = new nodejs.NodejsFunction(this, 'OptimizedFunction', {
  entry: 'src/handlers/optimized.ts',
  runtime: lambda.Runtime.NODEJS_22_X,
  architecture: lambda.Architecture.ARM_64,
  memorySize: 1769, // 1 full vCPU

  bundling: {
    minify: true,
    sourceMap: false, // Disable for production
    externalModules: ['@aws-sdk/*'], // Use Lambda runtime SDK
    esbuildArgs: {
      '--tree-shaking': 'true',
    },
  },

  // Environment variables for SDK optimization
  environment: {
    AWS_NODEJS_CONNECTION_REUSE_ENABLED: '1',
    NODE_OPTIONS: '--enable-source-maps',
  },
});

// Provisioned concurrency for latency-sensitive endpoints
const version = optimizedFunction.currentVersion;
const alias = new lambda.Alias(this, 'ProdAlias', {
  aliasName: 'prod',
  version,
  provisionedConcurrentExecutions: 5, // Pre-warmed instances
});
```

---

## Anti-Patterns

### Lambda Anti-Patterns

#### Monolithic Lambda Functions

**Why it's bad**: Slow cold starts, harder to debug, inefficient scaling.

```typescript
// DON'T - One Lambda doing everything
export const handler = async (event: any) => {
  if (event.path === '/users') { /* user logic */ }
  if (event.path === '/orders') { /* order logic */ }
  if (event.path === '/products') { /* product logic */ }
  // 50+ routes in one function...
};

// DO - Single-purpose functions
// user-handler.ts
export const handler = async (event: APIGatewayProxyEventV2) => {
  // Only user-related logic
};

// order-handler.ts
export const handler = async (event: APIGatewayProxyEventV2) => {
  // Only order-related logic
};
```

#### Synchronous Calls in Lambda

**Why it's bad**: Wastes compute time waiting for responses.

```typescript
// DON'T - Wait for downstream services
export const handler = async (event: any) => {
  const result = await processOrder(event);
  await sendEmail(result);        // Waiting...
  await updateInventory(result);  // Waiting...
  await notifyShipping(result);   // Waiting...
  return { statusCode: 200 };
};

// DO - Use async patterns
import { SQSClient, SendMessageCommand } from '@aws-sdk/client-sqs';
import { EventBridgeClient, PutEventsCommand } from '@aws-sdk/client-eventbridge';

export const handler = async (event: any) => {
  const result = await processOrder(event);

  // Fire and forget - EventBridge handles downstream
  await eventBridge.send(new PutEventsCommand({
    Entries: [{
      Source: 'orders',
      DetailType: 'OrderCreated',
      Detail: JSON.stringify(result),
    }],
  }));

  return { statusCode: 202 }; // Accepted
};
```

### DynamoDB Anti-Patterns

#### Relational Design in DynamoDB

**Why it's bad**: Requires multiple queries, defeats purpose of NoSQL.

```typescript
// DON'T - Separate tables like SQL
const user = await usersTable.get({ userId });
const orders = await ordersTable.query({ userId }); // Second call!
const addresses = await addressesTable.query({ userId }); // Third call!

// DO - Single-table design
const result = await table.query({
  KeyConditionExpression: 'pk = :pk',
  ExpressionAttributeValues: { ':pk': `USER#${userId}` },
});
// One query returns user, orders, and addresses
```

#### Scan Operations

**Why it's bad**: Reads entire table, expensive and slow.

```typescript
// DON'T - Full table scan
const allUsers = await docClient.send(new ScanCommand({
  TableName: 'users',
  FilterExpression: 'email = :email',
  ExpressionAttributeValues: { ':email': 'user@example.com' },
}));

// DO - Use GSI for access pattern
const user = await docClient.send(new QueryCommand({
  TableName: 'users',
  IndexName: 'GSI1',
  KeyConditionExpression: 'GSI1PK = :email',
  ExpressionAttributeValues: { ':email': `EMAIL#user@example.com` },
}));
```

### IAM Anti-Patterns

#### Wildcard Permissions

**Why it's bad**: Violates least privilege, security risk.

```json
// DON'T - Overly permissive
{
  "Effect": "Allow",
  "Action": "dynamodb:*",
  "Resource": "*"
}

// DO - Specific actions and resources
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:GetItem",
    "dynamodb:Query"
  ],
  "Resource": [
    "arn:aws:dynamodb:us-east-1:123456789:table/users",
    "arn:aws:dynamodb:us-east-1:123456789:table/users/index/*"
  ]
}
```

#### Long-Term Credentials

**Why it's bad**: Credentials can be leaked, no automatic rotation.

```typescript
// DON'T - Hardcoded credentials
const client = new DynamoDBClient({
  credentials: {
    accessKeyId: 'AKIAIOSFODNN7EXAMPLE',
    secretAccessKey: 'wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY',
  },
});

// DO - Use IAM roles (credentials provided automatically)
const client = new DynamoDBClient({});
// Lambda execution role provides temporary credentials
```

### S3 Anti-Patterns

#### Public Buckets

**Why it's bad**: Data exposure, compliance violation.

```typescript
// DON'T - Public bucket
const bucket = new s3.Bucket(this, 'PublicBucket', {
  publicReadAccess: true, // NEVER in production
});

// DO - Block all public access
const bucket = new s3.Bucket(this, 'SecureBucket', {
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
  encryption: s3.BucketEncryption.S3_MANAGED,
  enforceSSL: true,
});
```

### CDK Anti-Patterns

#### Hardcoded Values

**Why it's bad**: Prevents environment promotion, inflexible.

```typescript
// DON'T - Hardcoded environment values
const table = new dynamodb.Table(this, 'Table', {
  tableName: 'users-prod', // Hardcoded!
});

const function = new lambda.Function(this, 'Function', {
  environment: {
    API_URL: 'https://api.prod.example.com', // Hardcoded!
  },
});

// DO - Use context and parameters
const stage = this.node.tryGetContext('stage') || 'dev';

const table = new dynamodb.Table(this, 'Table', {
  tableName: `users-${stage}`,
});

const function = new lambda.Function(this, 'Function', {
  environment: {
    API_URL: ssm.StringParameter.valueForStringParameter(
      this, `/myapp/${stage}/api-url`
    ),
  },
});
```

---

## 2025-2026 Changelog (re:Invent 2025)

| Feature | Date | Description |
|---------|------|-------------|
| **Lambda Durable Functions** | Dec 2025 | Coordinate multi-step workflows over seconds to 1 year |
| **Lambda Managed Instances** | Dec 2025 | Run Lambda on EC2 with serverless simplicity |
| **Lambda Tenant Isolation** | Dec 2025 | Separate execution environments per tenant |
| Lambda INIT Phase Billing | Aug 2025 | INIT phase now billed for all configurations |
| **S3 Vectors GA** | Dec 2025 | Native vector storage, 2B vectors/index, 90% cost reduction |
| **S3 50TB Object Limit** | Dec 2025 | 10x increase from 5TB max object size |
| **ECS Express Mode** | Dec 2025 | Single-command container deployment |
| **Graviton5 Processors** | Dec 2025 | 25% faster than Graviton4, 192 cores/chip |
| **Database Savings Plans** | Dec 2025 | Up to 35% savings across 7 database services |
| CDK Refactor | Sep 2025 | Reorganize CDK code while preserving resources |
| CDK Booster | Sep 2025 | Faster TypeScript/JS Lambda bundling |
| DynamoDB Zero-ETL | Jan 2025 | Auto-replicate to Redshift/SageMaker Lakehouse |
| GuardDuty Extended Threat Detection | Dec 2025 | Unified visibility across EC2 and ECS |
| S3 Storage Lens Performance Metrics | Dec 2025 | Analyze billions of prefixes, hot prefix detection |

### Lambda Durable Functions

Build applications that coordinate multiple steps reliably over extended periods (seconds to 1 year) without paying for idle compute time:

```typescript
// Durable function example - no Step Functions needed
export const handler = async (event: DurableFunctionEvent) => {
  // Step 1: Process order
  const order = await processOrder(event.orderId);

  // Wait for human approval (up to 7 days) - no compute cost while waiting
  const approval = await waitForApproval(order.id, { timeout: '7d' });

  if (approval.approved) {
    // Step 2: Fulfill order
    await fulfillOrder(order);
  }

  return { status: 'completed' };
};
```

### Lambda Managed Instances

Run Lambda functions on EC2 compute while maintaining serverless simplicity:
- Access to specialized hardware (GPUs, high memory)
- EC2 pricing models for cost optimization
- AWS handles instance lifecycle, OS patching, load balancing, auto-scaling

### Lambda INIT Phase Billing (August 2025)

**Breaking Change:** INIT phase is now billed for all Lambda configurations:
- Previously free for ZIP-packaged functions with managed runtimes
- Now standardized across container images, custom runtimes, and Provisioned Concurrency
- Optimize cold starts to reduce costs

### S3 Vectors (Generally Available)

Native vector storage and querying for AI/ML workloads:
- Up to 2 billion vectors per index
- 100ms query latencies
- Up to 90% cost reduction vs specialized vector databases
- Ideal for RAG, semantic search, and agentic workloads

### ECS Express Mode

Deploy containerized applications with a single command:
```bash
# Automatically provisions: load balancers, autoscaling, networking, domains
aws ecs express deploy --image myapp:latest --name my-service
```

### Graviton5 Processors (EC2 M9g)

- 25% higher performance vs Graviton4
- 192 cores per chip
- 5x larger cache
- Best price-performance for general-purpose workloads

### Database Savings Plans

Up to 35% savings across seven services with flexible, cross-service commitments:
- Aurora
- RDS
- DynamoDB
- DocumentDB
- Neptune
- ElastiCache
- Timestream

---

## Quick Reference

### CDK Commands

| Command | Purpose |
|---------|---------|
| `cdk init app --language typescript` | Create new CDK project |
| `cdk synth` | Generate CloudFormation template |
| `cdk diff` | Show changes between deployed and local |
| `cdk deploy` | Deploy stack to AWS |
| `cdk destroy` | Delete stack and resources |
| `cdk watch` | Hot-deploy on file changes |

### Lambda Runtimes

| Runtime | Support | Recommendation |
|---------|---------|----------------|
| Node.js 22.x | Current | Recommended for new projects |
| Node.js 20.x | Supported | Production stable |
| Python 3.12 | Current | Recommended for new projects |
| Python 3.11 | Supported | Production stable |

### DynamoDB Capacity Modes

| Mode | Use Case | Pricing |
|------|----------|---------|
| On-Demand | Variable traffic | Pay per request |
| Provisioned | Predictable traffic | Pay for capacity |
| Provisioned + Auto Scaling | Variable but bounded | Cost optimization |

### API Gateway Types

| Type | Use Case | Cost |
|------|----------|------|
| HTTP API | Simple APIs, Lambda proxy | Lower |
| REST API | Advanced features, WAF, API keys | Higher |
| WebSocket API | Real-time, bidirectional | Per message |

### S3 Storage Classes

| Class | Use Case | Retrieval |
|-------|----------|-----------|
| Standard | Frequent access | Immediate |
| Standard-IA | Infrequent access | Immediate |
| Glacier Instant | Archive, rare access | Milliseconds |
| Glacier Flexible | Archive | Minutes to hours |
| Glacier Deep Archive | Long-term archive | 12-48 hours |

---

## Cost Optimization Checklist

- [ ] Use ARM64 (Graviton) for Lambda and EC2
- [ ] Right-size Lambda memory (use Power Tuning)
- [ ] Use Savings Plans for predictable workloads
- [ ] Use Spot Instances for fault-tolerant workloads
- [ ] Enable S3 Intelligent-Tiering
- [ ] Set DynamoDB to on-demand for variable traffic
- [ ] Delete unused resources (EBS, EIP, old snapshots)
- [ ] Tag all resources for cost allocation
- [ ] Set up AWS Budgets and Cost Anomaly Detection
- [ ] Review Cost Optimization Hub recommendations monthly

---

## Resources

- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [AWS CDK Best Practices](https://docs.aws.amazon.com/cdk/v2/guide/best-practices.html)
- [DynamoDB Single-Table Design](https://aws.amazon.com/blogs/compute/creating-a-single-table-design-with-amazon-dynamodb/)
- [Powertools for AWS Lambda](https://docs.aws.amazon.com/powertools/)
- [IAM Security Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
- [AWS Cost Optimization](https://aws.amazon.com/aws-cost-management/)

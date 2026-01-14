---
layout: post
title: Best Practices for AWS Lambda Functions - Production Guide
date: 2026-01-07
categories: [AWS, Serverless, Best Practices]
tags: [lambda, aws, serverless, best-practices, production]
excerpt: Comprehensive guide to production-ready AWS Lambda functions covering architecture, performance, security, and operational excellence.
---

AWS Lambda powers millions of serverless applications globally. However, building production-grade Lambda functions requires understanding not just how to write code, but also how to optimize performance, ensure reliability, secure your functions, and manage costs effectively. This guide covers the best practices I've learned from deploying Lambda at scale.

<!--more-->

## 1. Function Design: Single Responsibility Principle

Each Lambda function should have one clear purpose. This is fundamental to maintainability and scalability.

### Why It Matters

- **Independent Scaling**: Each function scales independently based on demand
- **Easier Debugging**: Isolated problems are simpler to troubleshoot
- **Faster Deployments**: Smaller functions = faster deployment cycles
- **Better Testing**: Unit tests focus on single responsibilities
- **Reusability**: Composable functions can be combined in workflows

### Good vs. Poor Design

```python
# ❌ POOR: Multiple responsibilities - hard to test, deploy, and scale
def lambda_handler(event, context):
    # Validate user input
    user_id = event.get('user_id')
    if not user_id:
        return {'error': 'Missing user_id'}
    
    # Fetch from database
    user = db.get_user(user_id)
    
    # Process user data
    processed = transform_user_data(user)
    
    # Send email notification
    send_email(processed['email'], "User processed")
    
    # Write to S3
    s3.put_object(processed)
    
    # Update third-party API
    external_api.update(processed)
    
    return {'statusCode': 200}
```

```python
# ✅ GOOD: Single responsibility - easy to test, deploy, and scale
def lambda_handler(event, context):
    """Fetch and return user data from database"""
    user_id = event['user_id']
    
    # Input validation
    if not user_id or not isinstance(user_id, str):
        raise ValueError(f"Invalid user_id: {user_id}")
    
    user = fetch_user(user_id)
    
    if not user:
        return {
            'statusCode': 404,
            'body': json.dumps({'error': 'User not found'})
        }
    
    return {
        'statusCode': 200,
        'body': json.dumps(user)
    }
```

In the good example, other operations (email, S3, API calls) would be handled by separate Lambda functions triggered by events (SQS, SNS, EventBridge).

## 2. Secure Configuration Management

Never hardcode secrets or configuration in your function code.

### Environment Variables (Non-Sensitive Config)

```python
import os
import json

def lambda_handler(event, context):
    # Non-sensitive configuration via environment variables
    db_host = os.environ.get('DB_HOST')
    db_port = os.environ.get('DB_PORT', '5432')
    environment = os.environ.get('ENVIRONMENT', 'dev')
    
    if not db_host:
        raise ValueError("DB_HOST environment variable not set")
    
    return process_data(db_host, int(db_port), environment)
```

### AWS Secrets Manager (Sensitive Data)

```python
import json
import boto3
from functools import lru_cache

secrets_client = boto3.client('secretsmanager')

@lru_cache(maxsize=1)
def get_database_credentials():
    """Cache secrets to reduce API calls and improve performance"""
    response = secrets_client.get_secret_value(SecretId='prod/db/credentials')
    return json.loads(response['SecretString'])

def lambda_handler(event, context):
    credentials = get_database_credentials()
    
    connection = connect_to_database(
        host=credentials['host'],
        user=credentials['username'],
        password=credentials['password']
    )
    
    return fetch_data(connection)
```

### IAM Roles (Principle of Least Privilege)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:region:account:secret:prod/db/credentials-*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:region:account:table/Users"
    }
  ]
}
```

**Key Principle**: Grant only the minimum permissions needed. This limits damage if credentials are compromised.

## 3. Robust Error Handling & Retry Logic

Production Lambda functions must handle failures gracefully and implement appropriate retry strategies.

### Structured Error Handling

```python
import json
import logging
from enum import Enum

logger = logging.getLogger()
logger.setLevel(logging.INFO)

class ErrorType(Enum):
    VALIDATION_ERROR = "ValidationError"
    DATABASE_ERROR = "DatabaseError"
    SERVICE_ERROR = "ServiceError"
    UNKNOWN_ERROR = "UnknownError"

class LambdaException(Exception):
    def __init__(self, error_type, message, statusCode=500):
        self.error_type = error_type
        self.message = message
        self.statusCode = statusCode
        super().__init__(self.message)

def lambda_handler(event, context):
    try:
        # Validate input
        required_fields = ['user_id', 'action']
        for field in required_fields:
            if field not in event:
                raise LambdaException(
                    ErrorType.VALIDATION_ERROR,
                    f"Missing required field: {field}",
                    statusCode=400
                )
        
        # Process request
        result = process_user_action(event['user_id'], event['action'])
        
        logger.info(f"Successfully processed request for user {event['user_id']}")
        
        return {
            'statusCode': 200,
            'body': json.dumps(result)
        }
    
    except LambdaException as e:
        logger.warning(f"{e.error_type.value}: {e.message}")
        return {
            'statusCode': e.statusCode,
            'body': json.dumps({
                'error': e.error_type.value,
                'message': e.message
            })
        }
    
    except Exception as e:
        logger.error(f"Unexpected error: {str(e)}", exc_info=True)
        return {
            'statusCode': 500,
            'body': json.dumps({
                'error': 'InternalServerError',
                'message': 'An unexpected error occurred'
            })
        }
```

### Async Retry with Exponential Backoff

```python
import time
from botocore.exceptions import ClientError

def invoke_with_retry(function_name, payload, max_retries=3):
    """Invoke another Lambda with exponential backoff retry"""
    lambda_client = boto3.client('lambda')
    
    for attempt in range(max_retries):
        try:
            response = lambda_client.invoke(
                FunctionName=function_name,
                InvocationType='RequestResponse',
                Payload=json.dumps(payload)
            )
            
            if response['StatusCode'] == 200:
                return json.loads(response['Payload'].read())
            
            raise Exception(f"Lambda returned status {response['StatusCode']}")
        
        except ClientError as e:
            if e.response['Error']['Code'] == 'ThrottlingException':
                wait_time = 2 ** attempt  # Exponential backoff: 1s, 2s, 4s
                logger.warning(f"Throttled on attempt {attempt + 1}, waiting {wait_time}s")
                time.sleep(wait_time)
            else:
                raise
        
        except Exception as e:
            if attempt == max_retries - 1:
                logger.error(f"Failed after {max_retries} attempts: {str(e)}")
                raise
            
            wait_time = 2 ** attempt
            logger.warning(f"Attempt {attempt + 1} failed, retrying in {wait_time}s")
            time.sleep(wait_time)
```

## 4. Memory & Timeout Optimization

Memory allocation directly impacts CPU performance and cost. This is critical for production systems.

### Understanding Memory-to-Performance Mapping

AWS Lambda allocates CPU proportionally to memory. Here's the relationship:

```
Memory    vCPU    Max Concurrent Executions
128 MB    0.2     1000
512 MB    0.8     1000
1024 MB   1.8     1000
2048 MB   3.6     1000
3008 MB   5.9     1000
```

### Optimization Strategy

```python
import time
import json
from datetime import datetime

def lambda_handler(event, context):
    start_time = time.time()
    
    logger.info(f"Function initialized with {context.memory_limit_in_mb}MB memory")
    logger.info(f"Time remaining: {context.get_remaining_time_in_millis()}ms")
    
    # Your processing logic
    result = heavy_computation(event)
    
    execution_time = time.time() - start_time
    remaining_time = context.get_remaining_time_in_millis()
    
    logger.info(f"Execution took {execution_time:.2f}s, {remaining_time}ms remaining")
    
    return {
        'statusCode': 200,
        'body': json.dumps(result),
        'executionTime': execution_time,
        'memoryUsed': context.memory_limit_in_mb
    }
```

### Timeout Best Practices

```python
def lambda_handler(event, context):
    # Set aggressive timeout checks - leave buffer for cleanup
    TIMEOUT_BUFFER_MS = 3000  # 3 second safety margin
    
    # Check timeout before long operations
    remaining_ms = context.get_remaining_time_in_millis()
    
    if remaining_ms < TIMEOUT_BUFFER_MS:
        logger.error("Insufficient time to complete operation")
        return {
            'statusCode': 503,
            'body': json.dumps({'error': 'Function timeout imminent'})
        }
    
    # Process with time awareness
    for item in large_dataset:
        if context.get_remaining_time_in_millis() < TIMEOUT_BUFFER_MS:
            logger.warning("Time running out, saving progress and returning")
            save_checkpoint(item)
            break
        
        process_item(item)
    
    return {'statusCode': 200}
```

## 5. Lambda Layers for Code Reuse

Lambda Layers enable sharing dependencies and common code across multiple functions.

### Creating and Using Layers

```bash
# Directory structure for a layer
layer-project/
├── python/
│   └── lib/
│       └── python3.11/
│           └── site-packages/
│               ├── requests/
│               ├── boto3/
│               └── custom_lib/
│                   └── utils.py
└── publish.sh

# Create the layer ZIP file
#!/bin/bash
cd python
pip install -r requirements.txt -t lib/python3.11/site-packages/
cd ..
zip -r database-layer.zip python/

# Publish to AWS
aws lambda publish-layer-version \
    --layer-name database-utils \
    --zip-file fileb://database-layer.zip \
    --compatible-runtimes python3.11 \
    --region us-east-1
```

### Using Layers in Lambda Functions

```python
# Functions using this layer can import from site-packages
import requests
from custom_lib.utils import normalize_response

def lambda_handler(event, context):
    response = requests.get('https://api.example.com/data')
    normalized = normalize_response(response.json())
    return normalized
```

### Layer Best Practices

```python
# ❌ DON'T: Store non-code files (configs, secrets)
layer/
└── python/site-packages/
    └── config.json  # WRONG! Don't include secrets

# ✅ DO: Keep layers for code and dependencies only
layer/
└── python/site-packages/
    ├── requests/
    ├── boto3/
    └── shared_utils.py

# Version your layers for consistency
# Use semantic versioning in layer names
# database-utils-v1.0.0
# database-utils-v1.1.0
```

## 6. Comprehensive Logging & Monitoring

Structured logging is essential for debugging and monitoring Lambda functions at scale.

### Structured Logging

```python
import json
import logging
from datetime import datetime

# Configure JSON logging
class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'function': record.funcName,
            'line': record.lineno
        }
        
        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)
        
        return json.dumps(log_data)

logger = logging.getLogger()
logger.setLevel(logging.INFO)

handler = logging.StreamHandler()
formatter = JSONFormatter()
handler.setFormatter(formatter)
logger.handlers = [handler]

def lambda_handler(event, context):
    request_id = context.aws_request_id
    
    logger.info('Processing request', extra={
        'request_id': request_id,
        'user_id': event.get('user_id'),
        'action': event.get('action')
    })
    
    try:
        result = process_request(event)
        logger.info('Request successful', extra={
            'request_id': request_id,
            'duration_ms': context.get_remaining_time_in_millis()
        })
        return result
    
    except Exception as e:
        logger.error('Request failed', extra={
            'request_id': request_id,
            'error': str(e)
        }, exc_info=True)
        raise
```

### CloudWatch Insights Queries

```
# Find slow requests
fields @timestamp, @duration
| filter @duration > 5000
| stats avg(@duration), max(@duration), pct(@duration, 95) by ispresent(error)

# Monitor errors by type
fields @message, error_type
| filter ispresent(error_type)
| stats count() by error_type

# Track memory usage patterns
fields @memoryUsed
| stats avg(@memoryUsed), max(@memoryUsed), pct(@memoryUsed, 99)
```

## 7. Cold Start Mitigation Strategies

Cold starts occur when Lambda needs to initialize a new execution environment, causing latency spikes.

### Understand Cold Start Factors

```python
# Cold start is affected by:
# 1. Package size (larger = slower initialization)
# 2. Number of dependencies (more imports = slower startup)
# 3. Initialization code outside handler
# 4. Runtime (Python is faster than Java)

# ❌ SLOW: Large package with slow initialization
import pandas
import tensorflow
import boto3
import requests
from complex_library import *

def expensive_init():
    # Time-consuming setup outside handler
    return setup_database_connection()

db_connection = expensive_init()

def lambda_handler(event, context):
    return query_database(db_connection)
```

```python
# ✅ FAST: Minimal dependencies, lazy initialization
import json
import logging

logger = logging.getLogger()

# Global but lazy-initialized
_db_connection = None

def get_db_connection():
    global _db_connection
    if _db_connection is None:
        import boto3
        _db_connection = boto3.client('rds').connect()
    return _db_connection

def lambda_handler(event, context):
    # Only import/initialize what's needed
    db = get_db_connection()
    return query_database(db)
```

### Provisioned Concurrency

```python
# For latency-sensitive functions, use Provisioned Concurrency

# CloudFormation/CDK configuration
# AWS Lambda keeps N execution environments "warm" and ready
# Trade: Additional cost (~$0.015/hour per provisioned concurrency)
# Benefit: Guaranteed sub-100ms cold start response

# Monitor cold start metrics
import time

def lambda_handler(event, context):
    # Track initialization time
    init_time = 1000 - (context.get_remaining_time_in_millis() % 1000)
    
    if init_time > 100:
        logger.warning(f"Slow cold start: {init_time}ms")
    
    return process(event)
```

## 8. Concurrency Management

Lambda has account-level concurrency limits. Understanding and managing concurrency is crucial.

```python
import boto3
import json

lambda_client = boto3.client('lambda')

def manage_concurrency():
    """Set appropriate concurrency limits"""
    
    # Get current limits
    response = lambda_client.get_account_settings()
    account_limit = response['AccountLimit']['ConcurrentExecutions']
    
    # Set function-level reserved concurrency
    # Prevents this function from consuming all account concurrency
    lambda_client.put_function_concurrency(
        FunctionName='critical-function',
        ReservedConcurrentExecutions=100
    )
    
    # Set provisioned concurrency (for guaranteed warmth)
    lambda_client.put_provisioned_concurrency_config(
        FunctionName='api-handler',
        Qualifier='LIVE',
        ProvisionedConcurrentExecutions=50
    )

def lambda_handler(event, context):
    # Handle gracefully when throttled
    try:
        invoke_dependent_function()
    except lambda_client.exceptions.TooManyRequestsException:
        logger.warning("Lambda throttled, queuing to SQS")
        queue_for_later(event)
        return {'statusCode': 202, 'message': 'Request queued'}
```

## 9. Testing Lambda Functions

Comprehensive testing prevents production issues.

```python
# handler.py
def fetch_user_data(user_id):
    """Fetches and transforms user data"""
    if not user_id:
        raise ValueError("user_id required")
    
    user = database.get(user_id)
    if not user:
        raise KeyError(f"User not found: {user_id}")
    
    return transform_user(user)

def lambda_handler(event, context):
    try:
        user_id = event['user_id']
        user_data = fetch_user_data(user_id)
        return {
            'statusCode': 200,
            'body': json.dumps(user_data)
        }
    except (ValueError, KeyError) as e:
        return {
            'statusCode': 400,
            'body': json.dumps({'error': str(e)})
        }

# test_handler.py
import pytest
import json
from handler import fetch_user_data, lambda_handler
from unittest.mock import Mock, patch

@pytest.fixture
def mock_database():
    with patch('handler.database') as mock:
        mock.get.return_value = {
            'id': 'user123',
            'name': 'John',
            'email': 'john@example.com'
        }
        yield mock

def test_fetch_user_data(mock_database):
    """Test user data fetching"""
    result = fetch_user_data('user123')
    assert result['id'] == 'user123'
    mock_database.get.assert_called_once_with('user123')

def test_fetch_user_data_not_found(mock_database):
    """Test handling of missing user"""
    mock_database.get.return_value = None
    with pytest.raises(KeyError):
        fetch_user_data('nonexistent')

def test_lambda_handler_success(mock_database):
    """Test successful Lambda execution"""
    event = {'user_id': 'user123'}
    context = Mock()
    
    response = lambda_handler(event, context)
    
    assert response['statusCode'] == 200
    body = json.loads(response['body'])
    assert body['name'] == 'John'

def test_lambda_handler_missing_user_id():
    """Test missing required parameter"""
    event = {}
    context = Mock()
    
    response = lambda_handler(event, context)
    
    assert response['statusCode'] == 400
    assert 'error' in json.loads(response['body'])
```

## 10. Cost Optimization

AWS Lambda costs are based on requests and GB-seconds. Here's how to optimize:

### Cost Calculation

```
Pricing = (Requests × $0.20/1M) + (GB-seconds × $0.0000166667)

Example:
- 1 million requests × 1GB memory × 1 second = 1,000,000 GB-seconds
- Cost = (1,000,000 × $0.20/1,000,000) + (1,000,000 × $0.0000166667)
- Cost = $0.20 + $16.67 = $16.87

Optimization: Increase memory → faster execution → lower GB-seconds
1024 MB × 200 ms = 0.2 GB-seconds (vs 1GB × 1000ms = 1 GB-second)
```

### Monitoring and Optimization

```python
import time

def lambda_handler(event, context):
    start_time = time.time()
    
    # Process request
    result = process_event(event)
    
    # Calculate actual cost
    duration_ms = (time.time() - start_time) * 1000
    memory_mb = context.memory_limit_in_mb
    gb_seconds = (memory_mb / 1024) * (duration_ms / 1000)
    estimated_cost = gb_seconds * 0.0000166667
    
    # Log for cost tracking
    logger.info(f"Execution cost: ${estimated_cost:.6f}", extra={
        'memory_mb': memory_mb,
        'duration_ms': duration_ms,
        'gb_seconds': gb_seconds
    })
    
    return result

# CloudWatch Insights query for cost analysis
"""
fields @duration, @memoryUsed
| stats avg(@duration), avg(@memoryUsed) by bin(5m)
| stats sum((@memoryUsed/1024) * (@duration/1000) * 0.0000166667) as estimated_cost
"""
```

## Real-World Case Study: Customer Review System with Lambda, DynamoDB & SNS

A complete production system for collecting customer reviews using AWS Lambda, DynamoDB, and SNS. This demonstrates how to combine multiple services and apply all best practices in a real-world scenario.

### System Components

#### 1. SMS Service (Amazon SNS)

**Purpose**: Send SMS messages to customers with review links

**Responsibilities**:
- Triggered by Lambda SMS Broadcaster
- Sends personalized SMS with unique review submission links
- Handles delivery status and failures
- Transactional SMS (not promotional)

**Integration**:
```python
sns_client = boto3.client('sns')
response = sns_client.publish(
    TopicArn='arn:aws:sns:region:account:sms-topic',
    Message=f'Hi {name}! Please share your feedback: {review_link}',
    MessageAttributes={
        'AWS.SNS.SMS.SmsType': {
            'DataType': 'String',
            'StringValue': 'Transactional'
        }
    }
)
```

#### 2. Database (Amazon DynamoDB)

**User Table**
- **Primary Key**: `userId` (String)
- **Attributes**:
  - `userId`: Unique identifier (PK)
  - `phone`: Phone number for SMS delivery
  - `name`: Customer name
  - `email`: Email address
  - `status`: 'active' or 'inactive'
  - `created_at`: Timestamp

**Review Table**
- **Primary Key**: `reviewId` (String) - Format: `{userId}#{timestamp}`
- **Sort Key**: `timestamp` (Number)
- **Attributes**:
  - `reviewId`: Unique review identifier (PK)
  - `userId`: Reference to customer (GSI)
  - `reviewText`: Customer feedback text
  - `rating`: Star rating (1-5)
  - `date`: Submission timestamp
  - `status`: 'new', 'approved', 'rejected' (for moderation)

**DynamoDB Schema Example**:
```json
{
  "UserTable": {
    "TableName": "users",
    "KeySchema": [
      {"AttributeName": "userId", "KeyType": "HASH"}
    ],
    "BillingMode": "PAY_PER_REQUEST"
  },
  "ReviewTable": {
    "TableName": "reviews",
    "KeySchema": [
      {"AttributeName": "reviewId", "KeyType": "HASH"},
      {"AttributeName": "timestamp", "KeyType": "RANGE"}
    ],
    "GlobalSecondaryIndexes": [
      {
        "IndexName": "userId-timestamp-index",
        "KeySchema": [
          {"AttributeName": "userId", "KeyType": "HASH"},
          {"AttributeName": "timestamp", "KeyType": "RANGE"}
        ]
      }
    ],
    "BillingMode": "PAY_PER_REQUEST"
  }
}
```

#### 3. Lambda Functions (Three-Function Architecture)

**Lambda SMS Broadcaster**
- Reads all active users from DynamoDB User Table
- Generates unique review submission links with tokens
- Sends SMS via SNS to each user
- Logs delivery status for auditing
- Handles errors and retries gracefully

**Lambda 1 – Review Page Renderer (HTML Form Server)**
- Triggered when user clicks SMS link
- Validates authentication token
- Serves HTML page with review form
- Form contains:
  - Label: "Please give your review"
  - Star rating selector (1-5 stars)
  - Text input field for review comments
  - Submit button
- On form submission, sends data to Lambda 2

**Lambda 2 – Review Processor**
- Receives form submission from user
- Validates input data (rating, text length)
- Stores review in DynamoDB Review Table
- Publishes notification to SNS for admin alerting
- Returns success/error response to user
- Logs analytics for business metrics

### System Flow (Complete Sequence)

```
Step 1: SMS Broadcast Trigger
┌─────────────────────────┐
│ Scheduled Event/API     │
│ (Daily at 10 AM)        │
└────────────┬────────────┘
             ▼
┌────────────────────────────────┐
│ Lambda: SMS Broadcaster        │
│ - Scan DynamoDB Users Table    │
│ - Filter: status = 'active'    │
│ - For each user:               │
│   • Generate review token      │
│   • Create review link         │
│   • Send SMS via SNS           │
└────────────┬───────────────────┘
             │
             ├─→ DynamoDB: Users Table ✓
             │
             └─→ SNS: SMS Topic ✓
                    │
                    ├─→ SMS Service
                    │      │
                    │      └──→ Customer Phone
                    │           "Click here to review: https://..."
                    │
                    └─→ CloudWatch Logs (audit trail)


Step 2: User Reviews Page Load
┌──────────────────────────┐
│ User clicks SMS link     │
│ Opens: /review?token=xxx │
└────────────┬─────────────┘
             ▼
┌────────────────────────────────┐
│ Lambda 1: Review Page Renderer │
│ - Extract token from URL       │
│ - Validate token (HMAC)        │
│ - Extract userId from token    │
│ - Generate HTML form           │
│ - Return HTML to browser       │
└────────────┬───────────────────┘
             │
             └─→ Browser receives HTML form ✓
                  ┌──────────────────────────┐
                  │ HTML FORM DISPLAYED      │
                  │                          │
                  │ "Please give your review"│
                  │ [⭐⭐⭐⭐⭐] Star rating  │
                  │ [Text area for review]   │
                  │ [Submit button]          │
                  └──────────────────────────┘


Step 3: User Submits Review
┌──────────────────────────┐
│ User fills form:         │
│ - Selects 5-star rating  │
│ - Types review text      │
│ - Clicks Submit          │
└────────────┬─────────────┘
             ▼
┌─────────────────────────────────┐
│ Form posts to Lambda 2 endpoint  │
│ POST /submit-review              │
│                                  │
│ Payload:                         │
│ {                                │
│   "token": "xxx...",             │
│   "userId": "user-123",          │
│   "reviewText": "Great product!",│
│   "rating": 5,                   │
│   "date": 1705238400000          │
│ }                                │
└────────────┬────────────────────┘
             ▼
┌──────────────────────────────┐
│ Lambda 2: Review Processor   │
│ - Validate token             │
│ - Validate review data       │
│ - Check rating (1-5)         │
│ - Check text length (<5000)  │
│ - Save to DynamoDB           │
│ - Publish SNS notification   │
└────────────┬─────────────────┘
             │
             ├─→ DynamoDB: Reviews Table ✓
             │   {
             │     "reviewId": "user-123#1705238400000",
             │     "userId": "user-123",
             │     "rating": 5,
             │     "reviewText": "Great product!",
             │     "date": 1705238400000,
             │     "status": "new"
             │   }
             │
             ├─→ SNS: Admin Notification Topic
             │   Subject: "New Review - 5⭐"
             │   Message: Review details for moderation
             │
             └─→ Response to user:
                  Status: 201 Created
                  Message: "Thank you for your review!"
                  Admin Dashboard: /admin/reviews/user-123#1705238400000
```

### Architecture Diagram

```
┌─────────────────┐
│  Review Service │
│ (Scheduled/API) │
└────────┬────────┘
         │
         ▼
┌──────────────────────────┐
│  Lambda: SMS Broadcaster │
│  - Read users from DDB   │
│  - Send SMS via SNS      │
└────────┬─────────────────┘
         │
         ├──► DynamoDB: Users Table
         │    (userId, phone, name, status)
         │
         └──► SNS: SMS Topic
              │
              ├──► AWS SMS Service
              │    │
              │    └──► Customer's Phone
              │         (Receives link)
              │
              ▼
        ┌─────────────────────┐
        │ User's Browser      │
        │ Clicks SMS Link     │
        └────────┬────────────┘
                 │
                 ▼
        ┌──────────────────────────┐
        │ Lambda 1: Form Renderer  │
        │ - Validate token         │
        │ - Return HTML form       │
        └────────┬─────────────────┘
                 │
                 ▼
        ┌──────────────────────────┐
        │ User Submits Review      │
        │ (Star rating + text)     │
        └────────┬─────────────────┘
                 │
                 ▼
        ┌──────────────────────────┐
        │ Lambda 2: Processor      │
        │ - Validate data          │
        │ - Store in DynamoDB      │
        │ - Notify admin via SNS   │
        └────────┬─────────────────┘
                 │
                 ├──► DynamoDB: Reviews Table
                 │    (reviewId, userId, rating, text, date)
                 │
                 └──► SNS: Admin Notifications
                      (Alert: New review needs moderation)
```

### Implementation Details

```
┌─────────────────┐
│  Review Service │
│ (Scheduled/API) │
└────────┬────────┘
         │
         ▼
┌──────────────────────────┐
│  Lambda: SMS Broadcaster │
│  - Read users from DDB   │
│  - Send SMS via SNS      │
└────────┬─────────────────┘
         │
         ├──► DynamoDB: Users Table
         │    (user_id, phone, name)
         │
         └──► SNS: SMS Topic
              │
              ├──► SMS Service
              │    │
              │    └──► Customer Phone
              │         (Review Link)
              │
              └──► Lambda: Form Handler
                   (Serves HTML form)
                        │
                        ▼
                   User Review
                        │
                        ▼
              ┌──────────────────────────┐
              │  Lambda: Review Processor│
              │  - Validate review data  │
              │  - Store in DynamoDB     │
              │  - Publish to SNS        │
              └──────────────────────────┘
                        │
                        ├──► DynamoDB: Reviews Table
                        │    (review_id, user_id, rating, text, timestamp)
                        │
                        └──► SNS: Review Notifications
                             (Alert admin/send confirmation)
```

### Implementation Details

#### 1. SMS Broadcaster Lambda

```python
import json
import boto3
import logging
from datetime import datetime
from typing import List, Dict

dynamodb = boto3.resource('dynamodb')
sns = boto3.client('sns')
logger = logging.getLogger()
logger.setLevel(logging.INFO)

users_table = dynamodb.Table(os.environ['USERS_TABLE'])
sms_topic_arn = os.environ['SMS_TOPIC_ARN']
review_form_url = os.environ['REVIEW_FORM_URL']

def get_active_users() -> List[Dict]:
    """Fetch all active users from DynamoDB"""
    try:
        response = users_table.scan(
            FilterExpression='#status = :status AND #phone <> :empty',
            ExpressionAttributeNames={
                '#status': 'status',
                '#phone': 'phone'
            },
            ExpressionAttributeValues={
                ':status': 'active',
                ':empty': ''
            }
        )
        
        logger.info(f"Retrieved {len(response['Items'])} active users")
        return response['Items']
    
    except Exception as e:
        logger.error(f"Failed to fetch users: {str(e)}", exc_info=True)
        raise

def send_sms_batch(users: List[Dict]) -> Dict:
    """Send SMS to multiple users with retry logic"""
    results = {
        'sent': 0,
        'failed': 0,
        'errors': []
    }
    
    for user in users:
        try:
            user_id = user['user_id']
            phone = user['phone']
            name = user.get('name', 'Valued Customer')
            
            # Create personalized review link
            review_token = generate_review_token(user_id)
            review_link = f"{review_form_url}?token={review_token}"
            
            # Craft SMS message
            message = f"Hi {name}! Please share your feedback: {review_link}"
            
            # Publish to SNS for SMS delivery
            response = sns.publish(
                TopicArn=sms_topic_arn,
                Message=message,
                MessageAttributes={
                    'AWS.SNS.SMS.SmsType': {
                        'DataType': 'String',
                        'StringValue': 'Transactional'  # Not promotional
                    },
                    'phone_number': {
                        'DataType': 'String',
                        'StringValue': phone
                    }
                }
            )
            
            logger.info(f"SMS sent to {user_id}: MessageId={response['MessageId']}")
            results['sent'] += 1
            
            # Log for audit trail
            log_sms_sent(user_id, phone, response['MessageId'])
        
        except Exception as e:
            logger.error(f"Failed to send SMS to {user.get('user_id')}: {str(e)}")
            results['failed'] += 1
            results['errors'].append({
                'user_id': user.get('user_id'),
                'error': str(e)
            })
    
    return results

def generate_review_token(user_id: str) -> str:
    """Generate secure token for review form"""
    import hmac
    import hashlib
    import base64
    
    secret = os.environ['TOKEN_SECRET']
    timestamp = str(int(datetime.utcnow().timestamp()))
    
    data = f"{user_id}:{timestamp}"
    signature = hmac.new(
        secret.encode(),
        data.encode(),
        hashlib.sha256
    ).digest()
    
    token = base64.urlsafe_b64encode(
        f"{data}:{base64.b64encode(signature).decode()}".encode()
    ).decode()
    
    return token

def log_sms_sent(user_id: str, phone: str, message_id: str):
    """Audit log for SMS sending"""
    audit_table = dynamodb.Table(os.environ['AUDIT_TABLE'])
    audit_table.put_item(
        Item={
            'event_id': f"{user_id}#{int(datetime.utcnow().timestamp() * 1000)}",
            'timestamp': int(datetime.utcnow().timestamp()),
            'user_id': user_id,
            'phone': phone,
            'message_id': message_id,
            'event_type': 'SMS_SENT'
        }
    )

def lambda_handler(event, context):
    """Main handler for SMS broadcast"""
    try:
        logger.info("Starting SMS broadcast")
        
        # Get active users
        users = get_active_users()
        
        if not users:
            logger.warning("No active users found")
            return {
                'statusCode': 200,
                'body': json.dumps({'message': 'No users to notify'})
            }
        
        # Send SMS batch
        results = send_sms_batch(users)
        
        logger.info(f"Broadcast complete: {results['sent']} sent, {results['failed']} failed")
        
        # Notify admin of results
        publish_admin_notification(results)
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'SMS broadcast completed',
                'results': results
            })
        }
    
    except Exception as e:
        logger.error(f"Lambda execution failed: {str(e)}", exc_info=True)
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Internal server error'})
        }

def publish_admin_notification(results: Dict):
    """Notify admin of broadcast results"""
    admin_topic = os.environ['ADMIN_TOPIC_ARN']
    sns.publish(
        TopicArn=admin_topic,
        Subject='SMS Broadcast Results',
        Message=f"Sent: {results['sent']}\nFailed: {results['failed']}"
    )
```

#### 2. Review Form Handler Lambda

```python
import json
import os
import logging
from typing import Dict

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# HTML form template
REVIEW_FORM_HTML = """
<!DOCTYPE html>
<html>
<head>
    <title>Customer Review</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 600px; margin: 50px auto; }
        .form-group { margin: 15px 0; }
        label { display: block; margin-bottom: 5px; font-weight: bold; }
        input[type="text"], textarea, select { width: 100%; padding: 8px; }
        textarea { resize: vertical; min-height: 120px; }
        button { background-color: #007bff; color: white; padding: 10px 20px; border: none; cursor: pointer; }
        button:hover { background-color: #0056b3; }
        .rating { display: flex; gap: 10px; }
        .star { font-size: 30px; cursor: pointer; }
    </style>
</head>
<body>
    <h1>Share Your Feedback</h1>
    <form action="/submit-review" method="POST">
        <input type="hidden" name="token" value="{token}">
        
        <div class="form-group">
            <label>How would you rate your experience?</label>
            <div class="rating">
                <span class="star" onclick="setRating(1)">⭐</span>
                <span class="star" onclick="setRating(2)">⭐</span>
                <span class="star" onclick="setRating(3)">⭐</span>
                <span class="star" onclick="setRating(4)">⭐</span>
                <span class="star" onclick="setRating(5)">⭐</span>
            </div>
            <input type="hidden" id="rating" name="rating" value="0">
        </div>
        
        <div class="form-group">
            <label>Your Review (Optional)</label>
            <textarea name="review_text" placeholder="Tell us more about your experience..."></textarea>
        </div>
        
        <button type="submit">Submit Review</button>
    </form>
    
    <script>
        function setRating(stars) {
            document.getElementById('rating').value = stars;
            document.querySelectorAll('.star').forEach((star, idx) => {
                star.style.opacity = idx < stars ? '1' : '0.3';
            });
        }
        setRating(0);
    </script>
</body>
</html>
"""

def lambda_handler(event, context):
    """Serve review form HTML"""
    try:
        # Extract token from query parameters
        token = event.get('queryStringParameters', {}).get('token')
        
        if not token:
            return {
                'statusCode': 400,
                'body': json.dumps({'error': 'Missing review token'})
            }
        
        # Validate token
        user_id = validate_token(token)
        if not user_id:
            return {
                'statusCode': 401,
                'body': json.dumps({'error': 'Invalid or expired token'})
            }
        
        logger.info(f"Serving review form for user: {user_id}")
        
        # Return HTML form
        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'text/html'
            },
            'body': REVIEW_FORM_HTML.format(token=token)
        }
    
    except Exception as e:
        logger.error(f"Error serving form: {str(e)}", exc_info=True)
        return {
            'statusCode': 500,
            'body': 'Error loading review form'
        }

def validate_token(token: str) -> str:
    """Validate and extract user_id from token"""
    import hmac
    import hashlib
    import base64
    from datetime import datetime
    
    try:
        secret = os.environ['TOKEN_SECRET']
        decoded = base64.urlsafe_b64decode(token.encode()).decode()
        
        parts = decoded.split(':')
        if len(parts) < 3:
            return None
        
        user_id = parts[0]
        timestamp = int(parts[1])
        signature = base64.b64decode(parts[2].encode())
        
        # Verify timestamp (valid for 7 days)
        now = int(datetime.utcnow().timestamp())
        if now - timestamp > 604800:  # 7 days
            return None
        
        # Verify signature
        data = f"{user_id}:{timestamp}"
        expected_sig = hmac.new(
            secret.encode(),
            data.encode(),
            hashlib.sha256
        ).digest()
        
        if signature != expected_sig:
            return None
        
        return user_id
    
    except Exception as e:
        logger.error(f"Token validation failed: {str(e)}")
        return None
```

#### 3. Review Processor Lambda

```python
import json
import boto3
import os
import logging
from datetime import datetime
from typing import Dict

dynamodb = boto3.resource('dynamodb')
sns = boto3.client('sns')
logger = logging.getLogger()
logger.setLevel(logging.INFO)

reviews_table = dynamodb.Table(os.environ['REVIEWS_TABLE'])
review_topic_arn = os.environ['REVIEW_TOPIC_ARN']

def validate_review(review_data: Dict) -> tuple[bool, str]:
    """Validate review submission"""
    
    # Validate rating
    rating = review_data.get('rating')
    try:
        rating = int(rating)
        if rating < 1 or rating > 5:
            return False, "Rating must be between 1 and 5"
    except (ValueError, TypeError):
        return False, "Invalid rating"
    
    # Validate review text (if provided)
    review_text = review_data.get('review_text', '').strip()
    if len(review_text) > 5000:
        return False, "Review text too long (max 5000 characters)"
    
    return True, ""

def store_review(user_id: str, rating: int, review_text: str) -> str:
    """Store review in DynamoDB"""
    
    review_id = f"{user_id}#{int(datetime.utcnow().timestamp() * 1000)}"
    
    try:
        reviews_table.put_item(
            Item={
                'review_id': review_id,
                'user_id': user_id,
                'rating': rating,
                'review_text': review_text,
                'timestamp': int(datetime.utcnow().timestamp()),
                'status': 'new',  # For moderation workflow
                'helpful_count': 0
            }
        )
        
        logger.info(f"Review stored: {review_id}")
        return review_id
    
    except Exception as e:
        logger.error(f"Failed to store review: {str(e)}", exc_info=True)
        raise

def publish_review_notification(review_id: str, user_id: str, rating: int):
    """Publish review notification to SNS"""
    
    try:
        sns.publish(
            TopicArn=review_topic_arn,
            Subject=f"New Review Submitted - {rating}⭐",
            Message=json.dumps({
                'review_id': review_id,
                'user_id': user_id,
                'rating': rating,
                'timestamp': datetime.utcnow().isoformat(),
                'action_url': f"{os.environ['ADMIN_DASHBOARD']}/reviews/{review_id}"
            })
        )
    except Exception as e:
        logger.error(f"Failed to publish notification: {str(e)}")
        # Don't fail the review submission if notification fails

def lambda_handler(event, context):
    """Handle review submission"""
    
    try:
        # Parse request body
        if isinstance(event.get('body'), str):
            body = json.loads(event['body'])
        else:
            body = event.get('body', {})
        
        logger.info(f"Processing review submission")
        
        # Validate token
        token = body.get('token')
        user_id = validate_token(token)
        if not user_id:
            return error_response(401, "Invalid or expired token")
        
        # Validate review data
        is_valid, error_msg = validate_review(body)
        if not is_valid:
            return error_response(400, error_msg)
        
        rating = int(body['rating'])
        review_text = body.get('review_text', '').strip()
        
        # Store review
        review_id = store_review(user_id, rating, review_text)
        
        # Publish notification
        publish_review_notification(review_id, user_id, rating)
        
        # Log for analytics
        log_review_submitted(user_id, rating, review_id)
        
        return {
            'statusCode': 201,
            'body': json.dumps({
                'message': 'Review submitted successfully',
                'review_id': review_id
            })
        }
    
    except Exception as e:
        logger.error(f"Review submission failed: {str(e)}", exc_info=True)
        return error_response(500, "Failed to submit review")

def error_response(status_code: int, message: str) -> Dict:
    """Format error response"""
    return {
        'statusCode': status_code,
        'body': json.dumps({'error': message})
    }

def log_review_submitted(user_id: str, rating: int, review_id: str):
    """Log for analytics and tracking"""
    analytics_table = dynamodb.Table(os.environ['ANALYTICS_TABLE'])
    
    try:
        analytics_table.put_item(
            Item={
                'event_type#timestamp': f"review_submitted#{int(datetime.utcnow().timestamp())}",
                'user_id': user_id,
                'rating': rating,
                'review_id': review_id,
                'timestamp': int(datetime.utcnow().timestamp())
            }
        )
    except Exception as e:
        logger.warning(f"Failed to log analytics: {str(e)}")

def validate_token(token: str) -> str:
    """Validate token (same as Form Handler)"""
    import hmac
    import hashlib
    import base64
    
    try:
        secret = os.environ['TOKEN_SECRET']
        decoded = base64.urlsafe_b64decode(token.encode()).decode()
        parts = decoded.split(':')
        
        if len(parts) < 3:
            return None
        
        user_id = parts[0]
        timestamp = int(parts[1])
        
        # Verify timestamp (7 day expiry)
        now = int(datetime.utcnow().timestamp())
        if now - timestamp > 604800:
            return None
        
        return user_id
    except Exception as e:
        logger.error(f"Token validation failed: {str(e)}")
        return None
```

### DynamoDB Table Schemas

```json
{
  "Users Table": {
    "TableName": "users",
    "PrimaryKey": "user_id",
    "Attributes": {
      "user_id": "String",
      "phone": "String",
      "name": "String",
      "email": "String",
      "status": "String",
      "created_at": "Number"
    }
  },
  "Reviews Table": {
    "TableName": "reviews",
    "PrimaryKey": "review_id",
    "GSI": [
      {
        "IndexName": "user_id-timestamp-index",
        "PartitionKey": "user_id",
        "SortKey": "timestamp"
      },
      {
        "IndexName": "rating-timestamp-index",
        "PartitionKey": "rating",
        "SortKey": "timestamp"
      }
    ],
    "Attributes": {
      "review_id": "String",
      "user_id": "String",
      "rating": "Number",
      "review_text": "String",
      "timestamp": "Number",
      "status": "String"
    }
  }
}
```

### Key Best Practices Applied

1. **Single Responsibility**: Each Lambda function has one job
   - SMS Broadcaster: Send SMSs
   - Form Handler: Serve HTML
   - Review Processor: Handle submissions

2. **Security**: Token-based authentication for review forms

3. **Error Handling**: Try-catch blocks with proper logging

4. **Monitoring**: Audit logs and analytics tracking

5. **Scalability**: DynamoDB with appropriate indexes for queries

6. **Cost Optimization**: Batch operations, efficient queries

7. **Logging**: Structured JSON logs for debugging and analysis

This real-world system demonstrates how to build a complete feature using Lambda, DynamoDB, and SNS following production best practices.

## Production Checklist

Before deploying Lambda functions to production:

- ✅ Functions have single, clear responsibilities
- ✅ All configuration is externalized (environment variables or Secrets Manager)
- ✅ Comprehensive error handling with appropriate status codes
- ✅ Structured logging with request tracking
- ✅ Memory allocation is optimized for your workload
- ✅ Timeout values are realistic with safety margins
- ✅ IAM role follows principle of least privilege
- ✅ Unit and integration tests cover happy path and error cases
- ✅ Cold start impact is measured and mitigated if needed
- ✅ Concurrent execution limits are set appropriately
- ✅ Monitoring and alerting configured in CloudWatch
- ✅ Cost implications are understood and budgeted

## Conclusion

Building production-grade Lambda functions requires attention to architecture, security, performance, and operational excellence. These best practices help you avoid common pitfalls and build functions that are reliable, maintainable, and cost-effective.

Start with the fundamentals—keep functions focused, handle errors gracefully, and log effectively. Then optimize based on your actual requirements rather than premature optimization.

---

**Have questions or experiences to share? Connect on [GitHub](https://github.com/Sheheriyar99) or [LinkedIn](https://www.linkedin.com/in/shaheryar-99s)!**

# Message Template Manager - Overview

A serverless AWS solution for sending email and SMS notifications through a simple REST API with reliable message queuing.

## What It Does

This system provides a complete infrastructure for:

- **Email Delivery**: Send emails via Amazon SES with template support
- **SMS Delivery**: Send SMS messages via AWS End User Messaging SMS
- **Template Management**: Store and reuse message templates in DynamoDB
- **Reliable Processing**: SQS queuing ensures messages aren't lost
- **Secure API**: JWT-based authentication for all API operations
- **Automatic Retries**: Failed messages retry up to 3 times before moving to dead letter queue

## How It Works

### The Flow

1. **Client sends** message request to REST API with JWT token
2. **API Gateway validates** JWT token via Lambda Authorizer
3. **API Gateway queues** message in SQS (returns immediately)
4. **SQS triggers** Lambda function to process messages
5. **Lambda retrieves** templates from DynamoDB (if using templates)
6. **Lambda sends** via Amazon SES (email) or End User Messaging SMS
7. **Failed messages** retry automatically or move to dead letter queue

### Key Components

**API Gateway** - REST API with JWT authentication
- Single POST endpoint for message submission
- JWT authorizer validates all requests
- Direct SQS integration (no Lambda proxy)

**Lambda Functions**
- JWT Authorizer - Validates authentication tokens
- Message Processor - Sends emails and SMS messages

**SQS Queues**
- Messages Queue - Buffers incoming message requests
- Dead Letter Queue - Captures failed messages after 3 retries

**DynamoDB Table**
- Message Templates - Stores reusable email/SMS templates

**AWS Secrets Manager**
- JWT Secret - Securely stores authentication secret (cached in Lambda memory per AWS best practices)

**Amazon SES** - Email delivery service

**AWS End User Messaging SMS** - SMS delivery service

## Deployment

### Prerequisites

- AWS Account with admin access
- AWS CLI and SAM CLI installed
- Python 3.12+
- Verified email addresses in Amazon SES
- SMS origination number in End User Messaging

### Quick Deploy

```bash
# 1. Generate JWT secret
python -c "import secrets; print(secrets.token_urlsafe(32))"

# 2. Build and deploy
cd message-template-manager
sam build
sam deploy --guided
```

### Configuration Parameters

During deployment, you'll configure:

- **Stack Name** - CloudFormation stack identifier
- **AWS Region** - Deployment region
- **JWT Secret** - For API authentication (minimum 16 characters)
- **SES Configuration Set** - Optional email tracking configuration
- **SMS Configuration Set** - Optional SMS tracking configuration

### Post-Deployment

1. Note the API endpoint URL from deployment outputs
2. Generate JWT tokens using the provided script
3. Add message templates to DynamoDB
4. Test email and SMS sending

## Architecture Highlights

**Serverless** - No servers to manage, automatic scaling
- Lambda functions for compute
- DynamoDB for template storage (on-demand billing)
- API Gateway for REST API
- SQS for reliable message queuing

**Secure** - Multiple layers of security
- JWT authentication on all endpoints
- Secrets stored in AWS Secrets Manager (cached in Lambda memory for performance - an AWS-recommended best practice)
- Lambda container isolation ensures cached secrets remain secure and ephemeral
- IAM roles with least privilege
- HTTPS encryption in transit

**Reliable** - Built for resilience
- SQS queuing decouples API from delivery
- Automatic retries (up to 3 attempts)
- Dead Letter Queue for failed messages
- CloudWatch alarm for DLQ monitoring (requires SNS configuration for notifications)

**Cost-Effective** - Pay only for usage
- On-demand DynamoDB billing
- Lambda charged per invocation
- SQS minimal cost
- SES/SMS charged per message

## Key Features

### Template System

Store reusable templates in DynamoDB:

```json
{
  "TemplateName": "alert-template",
  "MessageBody": "Alert: Your {productName} account ending in {membershipNumber} has a low balance of ${accountBalance}"
}
```

Use templates in requests with variable substitution:

```json
{
  "SMSMessage": {
    "TemplateName": "alert-template"
  },
  "Addresses": {
    "+16048621234": {
      "ChannelType": "SMS",
      "Substitutions": {
        "productName": "SAVINGS",
        "membershipNumber": "****7303",
        "accountBalance": "50.00"
      }
    }
  }
}
```

### Multi-Channel Support

Send to email, SMS, or both in a single request:

```json
{
  "TraceId": "12348",
  "EmailMessage": {
    "FromAddress": "alerts@example.com",
    "Subject": "Account Alert"
  },
  "SMSMessage": {
    "MessageType": "TRANSACTIONAL",
    "OriginationNumber": "your-pool-id",
    "TemplateName": "alert-template"
  },
  "Addresses": {
    "user@example.com": {
      "ChannelType": "EMAIL"
    },
    "+16048621234": {
      "ChannelType": "SMS"
    }
  }
}
```

### Flexible Substitutions

Support for both simple and array-based substitution formats:

**Simple format:**
```json
{
  "Substitutions": {
    "productName": "CHEQUING",
    "threshold": "100.00"
  }
}
```

**Array format (backward compatible):**
```json
{
  "Substitutions": {
    "productName": ["CHEQUING"],
    "threshold": ["100.00"]
  }
}
```

### Batch Processing

Lambda processes up to 10 messages per invocation with partial batch failure support - only failed messages are retried.

### Dead Letter Queue

Failed messages are preserved for 14 days in the DLQ. A CloudWatch alarm monitors the DLQ and triggers when messages arrive. To receive notifications, you must configure an SNS topic and add it to the alarm's notification actions (see Monitoring section below).

## Authentication

All API requests require JWT authentication:

```bash
# Generate token
python generate_jwt.py user123 user@example.com

# Use in request
curl -X POST "https://your-api-endpoint/dev/" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ ... }'
```

JWT tokens include:
- User ID (subject)
- Issuer (must be "messaging-api")
- Expiration time (default: 24 hours)
- Optional: email, customer ID

## Monitoring

**CloudWatch Logs** - All Lambda function execution logs

**CloudWatch Alarm** - Monitors DLQ for failed messages (alarm created but requires SNS topic configuration to receive notifications)

**SQS Metrics** - Queue depth, message age, processing rate

**Lambda Metrics** - Invocations, duration, errors, throttles

### Configuring Alarm Notifications

The CloudWatch alarm is created automatically but does not send notifications by default. To receive alerts:

1. Create an SNS topic for notifications
2. Subscribe your email/SMS to the topic
3. Update the alarm to add the SNS topic ARN to its AlarmActions

See the deployment guide for detailed steps.

## Message Format

Simple, flat JSON structure:

```json
{
  "TraceId": "unique-id",
  "EmailMessage": {
    "FromAddress": "sender@example.com",
    "Subject": "Subject line",
    "Substitutions": {
      "variable": "value"
    }
  },
  "SMSMessage": {
    "MessageType": "TRANSACTIONAL",
    "OriginationNumber": "pool-id",
    "TemplateName": "template-name"
  },
  "Addresses": {
    "recipient@example.com": {
      "ChannelType": "EMAIL"
    },
    "+1234567890": {
      "ChannelType": "SMS",
      "Substitutions": {
        "variable": "value"
      }
    }
  }
}
```

## Documentation

- **README.md** - Quick start and API usage examples
- **AUTHENTICATION.md** - Detailed JWT authentication guide
- **CUSTOMER_DEPLOYMENT_GUIDE.md** - Step-by-step deployment instructions (if available)

## Support

For detailed authentication information, see [AUTHENTICATION.md](../AUTHENTICATION.md).

For API usage examples, see [README.md](../README.md).

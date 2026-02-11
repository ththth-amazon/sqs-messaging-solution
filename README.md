# SQS-Based Messaging Solution

Send email and SMS notifications through a simple REST API backed by SQS for reliable message processing.

## Architecture

```
API Gateway (JWT Auth) → SQS Queue → Lambda → SES/SMS
                            ↓
                      Dead Letter Queue
                            ↓
                      CloudWatch Alarm
```

## Key Features

- **JWT Authentication**: Secure API access with JSON Web Tokens
- **Automatic Retries**: Messages retry 3 times before moving to DLQ
- **Visibility Timeout**: 5 minutes (prevents duplicate processing)
- **Dead Letter Queue**: Preserves failed messages for 14 days
- **Partial Batch Failures**: Only failed messages retry
- **CloudWatch Alarm**: Monitors DLQ (requires SNS configuration to receive notifications)
- **Template Management**: Store SMS/email templates in DynamoDB
- **Secrets Manager**: JWT secret stored securely in AWS Secrets Manager

## Deployment

### Prerequisites

- AWS CLI configured
- SAM CLI installed
- Verified email addresses in Amazon SES
- SMS origination number in End User Messaging

### Deploy

```bash
sam build
sam deploy --guided
```

Follow the prompts to configure:
- Stack name
- AWS Region
- **JWT Secret** (important: use a strong, random secret for production)
  - **Note:** Must be at least 16 characters long
- **SES Configuration Set** (optional: for email tracking and analytics)
- **SMS Configuration Set** (optional: for SMS tracking and analytics)

The deployment will create:
- API Gateway with JWT authorizer
- SQS queues (main and DLQ)
- Lambda functions (authorizer and message processor)
- DynamoDB table for templates
- Secrets Manager secret for JWT
- CloudWatch alarm for DLQ (requires SNS configuration for notifications)

## Authentication

This API uses JWT (JSON Web Token) authentication. All API requests must include a valid JWT token.

### JWT Token Requirements

Tokens must include:
- `iss` (issuer): `messaging-api`
- `sub` (subject): User identifier
- `exp` (expiration): Token expiration time
- `iat` (issued at): Token creation time

Optional claims:
- `email`: User email
- `customer_id`: Customer identifier

### Generate JWT Tokens

Use the provided `generate_jwt.py` script:

```bash
python generate_jwt.py <user_id> [email] [customer_id]
```

Example:
```bash
python generate_jwt.py user123 user@example.com cust456
```

### Rotate JWT Secret

To rotate the JWT secret:

```bash
aws secretsmanager update-secret \
  --secret-id <your-stack-name>-jwt-secret \
  --secret-string "new-secret-key-here" \
  --region <your-region>
```

The Lambda authorizer will automatically pick up the new secret.

## Setup Message Templates

After deployment, add templates to DynamoDB:

```bash
aws dynamodb put-item \
  --table-name MessageTemplates \
  --item '{
    "TemplateName": {"S": "alert-template"},
    "MessageBody": {"S": "Alert: Your {productName} account ending in {membershipNumber} has a low balance of ${accountBalance}"}
  }'
```

**Template Size Limits:**
- DynamoDB has a 400 KB limit per item (including attribute names and values)
- This is sufficient for most email and SMS templates
- Typical email templates: 5-20 KB
- Typical SMS templates: < 1 KB
- If you need larger templates, consider storing them in Amazon S3 and referencing the S3 key in DynamoDB

## Usage Examples

**Note:** All requests require a JWT token in the Authorization header.

### Email Only

```bash
curl -X POST "https://your-api-endpoint/dev/" \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_JWT_TOKEN' \
  -d '{
    "TraceId": "12345",
    "EmailMessage": {
      "FromAddress": "alerts@example.com",
      "Subject": "Low Balance Alert",
      "Substitutions": {
        "productName": "CHEQUING",
        "membershipNumber": "****5493",
        "accountBalance": "100.00"
      }
    },
    "Addresses": {
      "user@example.com": {
        "ChannelType": "EMAIL"
      }
    }
  }'
```

### SMS with Template

```bash
curl -X POST "https://your-api-endpoint/dev/" \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_JWT_TOKEN' \
  -d '{
    "TraceId": "12346",
    "SMSMessage": {
      "MessageType": "TRANSACTIONAL",
      "OriginationNumber": "your-pool-id",
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
  }'
```

### SMS with Inline Message

```bash
curl -X POST "https://your-api-endpoint/dev/" \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_JWT_TOKEN' \
  -d '{
    "TraceId": "12347",
    "SMSMessage": {
      "MessageType": "TRANSACTIONAL",
      "OriginationNumber": "your-pool-id",
      "MessageBody": "Your payment of $50 is due tomorrow"
    },
    "Addresses": {
      "+16048621234": {
        "ChannelType": "SMS"
      }
    }
  }'
```

### Both Email and SMS

```bash
curl -X POST "https://your-api-endpoint/dev/" \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_JWT_TOKEN' \
  -d '{
    "TraceId": "12348",
    "EmailMessage": {
      "FromAddress": "alerts@example.com",
      "Subject": "Account Alert",
      "Substitutions": {
        "productName": "CHEQUING",
        "membershipNumber": "****5493",
        "accountBalance": "100.00"
      }
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
        "ChannelType": "SMS",
        "Substitutions": {
          "productName": "CHEQUING",
          "membershipNumber": "****7303",
          "accountBalance": "100.00"
        }
      }
    }
  }'
```

## Monitoring

### Check Queue Depth

```bash
aws sqs get-queue-attributes \
  --queue-url <your-queue-url> \
  --attribute-names ApproximateNumberOfMessages
```

### Check Dead Letter Queue

```bash
aws sqs get-queue-attributes \
  --queue-url <your-dlq-url> \
  --attribute-names ApproximateNumberOfMessages
```

### View Lambda Logs

```bash
aws logs tail /aws/lambda/MessageProcessor --follow
```

## Handling Failed Messages

### Redrive from Console

1. Go to SQS Console
2. Select Dead Letter Queue
3. Click "Start DLQ redrive"
4. Choose destination (main queue)

### Manual Inspection

```bash
aws sqs receive-message \
  --queue-url <your-dlq-url> \
  --max-number-of-messages 10
```

## Configuration

### Configuration Sets (Optional)

Configuration sets enable tracking and analytics for email and SMS messages.

**Email Configuration Sets (SES):**
- Track email delivery, opens, clicks, bounces, and complaints
- Send events to CloudWatch, Kinesis, or SNS
- Monitor email reputation and engagement

**SMS Configuration Sets (End User Messaging):**
- Track SMS delivery and failure events
- Monitor SMS sending patterns
- Analyze message costs by destination

**To use configuration sets:**

1. Create a configuration set in AWS Console:
   - For Email: SES Console > Configuration Sets
   - For SMS: End User Messaging SMS Console > Configuration Sets

2. Deploy/update your stack with configuration set names:
   ```bash
   sam deploy --parameter-overrides \
     SESConfigurationSet=my-email-config-set \
     SMSConfigurationSet=my-sms-config-set
   ```

3. Messages will automatically use the configured sets for tracking

### Adjust Retry Count

Change `maxReceiveCount` in template.yaml (default: 3)

### Adjust Visibility Timeout

Change `VisibilityTimeout` based on Lambda duration (default: 300s)

### Batch Size

Change `BatchSize` for Lambda (default: 10)

### Message Retention

- Main Queue: 4 days (default)
- DLQ: 14 days (default)

## Cost Estimate (1M Requests)

| Service | Cost |
|---------|------|
| API Gateway | $3.50 |
| SQS | $0.40 |
| Lambda | $2.50 |
| SES | $100.00 |
| SMS | $10,000.00 |
| **Total** | **$10,106.40** |

## Cleanup

```bash
aws cloudformation delete-stack --stack-name <your-stack-name>
```

## Payload Format

The solution uses a simple, flat JSON structure:

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
    "OriginationNumber": "262782",
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

## Substitution Variables

You can use either format:

**String format (simpler):**
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

Both work! The Lambda handles both formats automatically.

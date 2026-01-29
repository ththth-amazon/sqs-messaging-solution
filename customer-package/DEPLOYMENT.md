# Deployment Guide

Quick guide to deploy the SQS-Based Messaging Solution to your AWS account.

## Prerequisites

1. **AWS Account** with appropriate permissions
2. **AWS CLI** installed and configured
   ```bash
   aws configure
   ```
3. **SAM CLI** installed
   ```bash
   pip install aws-sam-cli
   ```
4. **Python 3.12** installed

## Deployment Steps

### 1. Extract the Package

```bash
unzip sqs-messaging-solution.zip
cd sqs-messaging-solution
```

### 2. Generate a Strong JWT Secret

**Important:** Do NOT use the default secret in production!

Generate a strong random secret:

```bash
# Option 1: Using Python
python -c "import secrets; print(secrets.token_urlsafe(32))"

# Option 2: Using OpenSSL (Linux/Mac)
openssl rand -base64 32
```

Save this secret - you'll need it for deployment and for generating JWT tokens.

### 3. Build the Application

```bash
sam build
```

### 4. Deploy

```bash
sam deploy --guided
```

You'll be prompted for:

- **Stack Name**: e.g., `my-messaging-api`
- **AWS Region**: e.g., `us-east-1`
- **JWTSecret**: Paste your generated secret from step 2
- **Confirm changes before deploy**: Y
- **Allow SAM CLI IAM role creation**: Y
- **Disable rollback**: N
- **Save arguments to configuration file**: Y

### 5. Note Your Outputs

After deployment, save these values:

```
ApiEndpoint: https://xxxxx.execute-api.region.amazonaws.com/dev/
JWTSecretArn: arn:aws:secretsmanager:region:account:secret:xxxxx
```

## Post-Deployment Configuration

### 1. Verify Email Addresses (Amazon SES)

Before sending emails, verify sender addresses:

```bash
aws sesv2 create-email-identity \
  --email-identity your-email@example.com \
  --region YOUR_REGION
```

Check your email and click the verification link.

### 2. Configure SMS (Optional)

If using SMS:
1. Go to AWS Console → Amazon Pinpoint SMS
2. Request a phone number or sender ID
3. Note your pool ID or phone number

### 3. Add Message Templates (Optional)

```bash
# SMS Template
aws dynamodb put-item \
  --table-name MessageTemplates \
  --region YOUR_REGION \
  --item '{
    "TemplateName": {"S": "alert-template"},
    "MessageBody": {"S": "Alert: Your {productName} account ending in {membershipNumber} has a low balance of ${accountBalance}"}
  }'

# Email Template
aws dynamodb put-item \
  --table-name MessageTemplates \
  --region YOUR_REGION \
  --item '{
    "TemplateName": {"S": "email-alert-template"},
    "MessageBody": {"S": "<html><body><h2>Account Alert</h2><p>Your {productName} account ending in {membershipNumber} has a low balance of ${accountBalance}.</p></body></html>"},
    "Subject": {"S": "Low Balance Alert - {productName}"}
  }'
```

## Testing

### 1. Generate a Test JWT Token

```bash
python generate_jwt.py test-user test@example.com
```

Copy the generated token.

### 2. Send a Test Email

```bash
curl -X POST "YOUR_API_ENDPOINT" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d '{
    "TraceId": "test-001",
    "EmailMessage": {
      "FromAddress": "your-verified-email@example.com",
      "Subject": "Test Email"
    },
    "Addresses": {
      "recipient@example.com": {
        "ChannelType": "EMAIL"
      }
    }
  }'
```

### 3. Check Logs

```bash
aws logs tail /aws/lambda/MessageProcessor --follow --region YOUR_REGION
```

## Integration

### Store Your JWT Secret Securely

**In your application:**

```python
# Option 1: Environment Variable
import os
JWT_SECRET = os.environ['JWT_SECRET']

# Option 2: AWS Secrets Manager
import boto3
secrets = boto3.client('secretsmanager')
response = secrets.get_secret_value(SecretId='my-jwt-secret')
JWT_SECRET = response['SecretString']
```

### Generate Tokens in Your Application

```python
import jwt
import datetime

def generate_token(user_id, email=None):
    JWT_SECRET = "your-secret-from-deployment"
    JWT_ISSUER = "messaging-api"
    
    payload = {
        'sub': user_id,
        'iss': JWT_ISSUER,
        'iat': datetime.datetime.utcnow(),
        'exp': datetime.datetime.utcnow() + datetime.timedelta(hours=24)
    }
    
    if email:
        payload['email'] = email
    
    return jwt.encode(payload, JWT_SECRET, algorithm='HS256')
```

See `AUTHENTICATION.md` for detailed authentication documentation.

## Monitoring

### Check Queue Depth

```bash
aws sqs get-queue-attributes \
  --queue-url YOUR_QUEUE_URL \
  --attribute-names ApproximateNumberOfMessages \
  --region YOUR_REGION
```

### Check Dead Letter Queue

```bash
aws sqs get-queue-attributes \
  --queue-url YOUR_DLQ_URL \
  --attribute-names ApproximateNumberOfMessages \
  --region YOUR_REGION
```

### View Lambda Logs

```bash
aws logs tail /aws/lambda/MessageProcessor --follow --region YOUR_REGION
aws logs tail /aws/lambda/JWTAuthorizer --follow --region YOUR_REGION
```

## Cleanup

To remove all resources:

```bash
sam delete --stack-name YOUR_STACK_NAME --region YOUR_REGION
```

## Troubleshooting

### Deployment Fails

- Ensure AWS CLI is configured correctly
- Check you have necessary IAM permissions
- Verify SAM CLI is installed

### 401 Unauthorized

- Check JWT token is included in Authorization header
- Verify token hasn't expired
- Confirm JWT secret matches

### Email Not Sending

- Verify sender email address in SES
- Check Lambda logs for errors
- Ensure SES is out of sandbox mode for production

### SMS Not Sending

- Verify SMS pool ID or phone number
- Check Lambda logs for errors
- Ensure phone numbers are in E.164 format (+1234567890)

## Support

For detailed documentation:
- `README.md` - Full API documentation
- `AUTHENTICATION.md` - JWT authentication guide

For AWS-specific issues:
- [AWS SAM Documentation](https://docs.aws.amazon.com/serverless-application-model/)
- [Amazon SES Documentation](https://docs.aws.amazon.com/ses/)
- [AWS Support](https://console.aws.amazon.com/support/)

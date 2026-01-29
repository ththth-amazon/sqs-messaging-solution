# Complete Deployment and Usage Guide

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
3. [Deployment Steps](#deployment-steps)
4. [Post-Deployment Configuration](#post-deployment-configuration)
5. [Testing Your Deployment](#testing-your-deployment)
6. [Using the API](#using-the-api)
7. [Monitoring and Troubleshooting](#monitoring-and-troubleshooting)
8. [Production Checklist](#production-checklist)

---

## Prerequisites

Before you begin, ensure you have the following:

### Required Software

1. **AWS Account** with administrative access
   - Sign up at https://aws.amazon.com if you don't have one

2. **AWS CLI** (version 2.x or later)
   - Download: https://aws.amazon.com/cli/
   - Verify installation:
     ```bash
     aws --version
     ```

3. **AWS SAM CLI** (version 1.x or later)
   - Install via pip:
     ```bash
     pip install aws-sam-cli
     ```
   - Verify installation:
     ```bash
     sam --version
     ```

4. **Python 3.12 or Later (SAM CLI is not supported higher than 3.12 right now)**
   - Download: https://www.python.org/downloads/
   - Verify installation:
     ```bash
     python --version
     ```

5. **PyJWT Library** (for generating tokens)
   - Install:
     ```bash
     pip install PyJWT
     ```

### AWS Account Preparation

1. **Configure AWS CLI** with your credentials:
   ```bash
   aws configure
   ```
   
   You'll be prompted for:
   - AWS Access Key ID
   - AWS Secret Access Key
   - Default region (e.g., `us-east-1`)
   - Default output format (use `json`)

2. **Verify AWS CLI access**:
   ```bash
   aws sts get-caller-identity
   ```
   
   This should return your account information.

---

## Initial Setup

### Step 1: Extract the Package

1. Locate the `sqs-messaging-solution-customer-package.zip` file
2. Extract it to a working directory:
   ```bash
   unzip sqs-messaging-solution-customer-package.zip
   cd customer-package
   ```

3. Verify the contents:
   ```bash
   ls -la
   ```
   
   You should see:
   - `template.yaml` - SAM infrastructure template
   - `lambda/` - Lambda function code
   - `README.md` - API documentation
   - `AUTHENTICATION.md` - JWT authentication guide
   - `DEPLOYMENT.md` - Quick deployment reference
   - `generate_jwt.py` - Token generation script
   - `.gitignore` - Git ignore file

### Step 2: Generate a Strong JWT Secret

**CRITICAL:** Do NOT use the default secret in production!

Generate a cryptographically secure random secret:

**Option 1: Using Python**
```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

**Option 2: Using OpenSSL (Linux/Mac)**
```bash
openssl rand -base64 32
```

**Option 3: Using PowerShell (Windows)**
```powershell
[Convert]::ToBase64String((1..32 | ForEach-Object { Get-Random -Minimum 0 -Maximum 256 }))
```

**Save this secret securely!** You'll need it for:
- Deployment (next step)
- Generating JWT tokens
- Your application configuration

Example output:
```
aB3xK9mP2qL7nR5wT8yU4zV6cD1eF0gH2iJ3kL4mN5oP6qR7sT8uV9wX0yZ1
```

---

## Deployment Steps

### Step 3: Build the Application

From the `customer-package` directory, run:

```bash
sam build
```

Expected output:
```
Building codeuri: /path/to/lambda runtime: python3.12 metadata: {} architecture: x86_64 functions: AuthorizerFunction, MessageProcessorFunction
...
Build Succeeded
```

If you see errors, ensure:
- You're in the correct directory
- Python 3.12 is installed
- SAM CLI is properly installed

### Step 4: Deploy to AWS

Run the guided deployment:

```bash
sam deploy --guided
```

You'll be prompted for several configuration options:

#### Deployment Prompts and Recommended Answers

1. **Stack Name**: 
   - Prompt: `Stack Name [sam-app]:`
   - Enter: `my-messaging-api` (or your preferred name)
   - This name will be used to identify your resources in AWS

2. **AWS Region**:
   - Prompt: `AWS Region [us-east-1]:`
   - Enter: Your preferred region (e.g., `us-east-1`, `us-west-2`)
   - Choose a region close to your users

3. **Parameter JWTSecret**:
   - Prompt: `Parameter JWTSecret [change-this-secret-key-in-production-REPLACE-ME]:`
   - **PASTE YOUR GENERATED SECRET** from Step 2
   - This is the most important security setting

4. **Confirm changes before deploy**:
   - Prompt: `Confirm changes before deploy [Y/n]:`
   - Enter: `Y`
   - This lets you review changes before applying

5. **Allow SAM CLI IAM role creation**:
   - Prompt: `Allow SAM CLI IAM role creation [Y/n]:`
   - Enter: `Y`
   - Required for Lambda functions to access AWS services

6. **Disable rollback**:
   - Prompt: `Disable rollback [y/N]:`
   - Enter: `N`
   - Rollback on failure is recommended

7. **AuthorizerFunction has no authentication**:
   - Prompt: `AuthorizerFunction has no authentication. Is this okay? [y/N]:`
   - Enter: `y`
   - This is expected - the authorizer itself doesn't need auth

8. **Save arguments to configuration file**:
   - Prompt: `Save arguments to configuration file [Y/n]:`
   - Enter: `Y`
   - Saves settings for future deployments

9. **SAM configuration file**:
   - Prompt: `SAM configuration file [samconfig.toml]:`
   - Press Enter (accept default)

10. **SAM configuration environment**:
    - Prompt: `SAM configuration environment [default]:`
    - Press Enter (accept default)

### Step 5: Wait for Deployment

The deployment process takes 3-5 minutes. You'll see:

```
Deploying with following values
===============================
Stack name                   : my-messaging-api
Region                       : us-east-1
...

CloudFormation stack changeset
-------------------------------------------------------------------------------------------------
Operation                LogicalResourceId        ResourceType             Replacement
-------------------------------------------------------------------------------------------------
+ Add                    ApiGateway               AWS::ApiGateway::...     N/A
+ Add                    AuthorizerFunction       AWS::Lambda::Function    N/A
...
-------------------------------------------------------------------------------------------------

Deploy this changeset? [y/N]: y
```

Type `y` and press Enter.

### Step 6: Save Your Outputs

When deployment completes, you'll see important outputs:

```
CloudFormation outputs from deployed stack
-------------------------------------------------------------------------------------------------
Outputs
-------------------------------------------------------------------------------------------------
Key                 ApiEndpoint
Description         API Gateway endpoint URL for Dev stage
Value               https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev/

Key                 JWTSecretArn
Description         ARN of the JWT secret in Secrets Manager
Value               arn:aws:secretsmanager:us-east-1:123456789012:secret:my-messaging-api-jwt-secret-AbCdEf

Key                 MessagesQueueUrl
Description         SQS Queue URL
Value               https://sqs.us-east-1.amazonaws.com/123456789012/MessagesQueue

Key                 DeadLetterQueueUrl
Description         Dead Letter Queue URL
Value               https://sqs.us-east-1.amazonaws.com/123456789012/MessagesDeadLetterQueue
-------------------------------------------------------------------------------------------------

Successfully created/updated stack - my-messaging-api in us-east-1
```

**SAVE THESE VALUES!** You'll need them for:
- API calls (ApiEndpoint)
- Monitoring (Queue URLs)
- Secret rotation (JWTSecretArn)

Create a file to store these:
```bash
echo "API_ENDPOINT=https://abc123xyz.execute-api.us-east-1.amazonaws.com/dev/" > deployment-info.txt
echo "JWT_SECRET_ARN=arn:aws:secretsmanager:us-east-1:123456789012:secret:my-messaging-api-jwt-secret-AbCdEf" >> deployment-info.txt
echo "QUEUE_URL=https://sqs.us-east-1.amazonaws.com/123456789012/MessagesQueue" >> deployment-info.txt
echo "DLQ_URL=https://sqs.us-east-1.amazonaws.com/123456789012/MessagesDeadLetterQueue" >> deployment-info.txt
```

---

## Post-Deployment Configuration

### Step 7: Verify Email Addresses in Amazon SES

Before sending emails, you must verify sender email addresses.

#### For Sandbox Mode (Testing)

1. **Verify sender email**:
   ```bash
   aws sesv2 create-email-identity \
     --email-identity your-sender@example.com \
     --region us-east-1
   ```

2. **Check your email** for a verification link from Amazon SES

3. **Click the verification link**

4. **Verify recipient email** (required in sandbox mode):
   ```bash
   aws sesv2 create-email-identity \
     --email-identity recipient@example.com \
     --region us-east-1
   ```

5. **Verify the recipient email** as well

#### For Production (Sending to Any Email)

Request production access:

1. Go to AWS Console → Amazon SES
2. Click "Account dashboard" in the left menu
3. Click "Request production access"
4. Fill out the form:
   - Mail type: Transactional
   - Website URL: Your website
   - Use case description: Describe your messaging needs
   - Compliance: Confirm you follow best practices
5. Submit and wait for approval (usually 24 hours)

### Step 8: Configure SMS (Optional)

If you plan to send SMS messages:

#### Option A: Using AWS Console

1. Go to AWS Console → AWS End User Messaging SMS
2. Click "Phone numbers" → "Request phone number"
3. Choose:
   - Country: United States (or your target country)
   - Number type: Toll-free or 10DLC
   - Message type: Transactional
4. Complete registration (may require business verification)
5. Note your phone number or pool ID

#### Option B: Using AWS CLI

```bash
# Request a phone number
aws pinpoint-sms-voice-v2 request-phone-number \
  --iso-country-code US \
  --message-type TRANSACTIONAL \
  --number-type TOLL_FREE \
  --region us-east-1
```

Save the phone number or pool ID for API calls.

### Step 9: Add Message Templates (Optional)

Templates allow you to reuse message content across multiple sends.

#### Add an SMS Template

```bash
aws dynamodb put-item \
  --table-name MessageTemplates \
  --region us-east-1 \
  --item '{
    "TemplateName": {"S": "alert-template"},
    "MessageBody": {"S": "Alert: Your {productName} account ending in {membershipNumber} has a low balance of ${accountBalance}"}
  }'
```

#### Add an Email Template

```bash
aws dynamodb put-item \
  --table-name MessageTemplates \
  --region us-east-1 \
  --item '{
    "TemplateName": {"S": "email-alert-template"},
    "MessageBody": {"S": "<html><body><h2>Account Alert</h2><p>Your {productName} account ending in {membershipNumber} has a low balance of ${accountBalance}.</p><p>Please take action to avoid service interruptions.</p></body></html>"},
    "Subject": {"S": "Low Balance Alert - {productName}"}
  }'
```

#### Verify Templates Were Created

```bash
aws dynamodb scan \
  --table-name MessageTemplates \
  --region us-east-1
```

---

## Testing Your Deployment

### Step 10: Generate a Test JWT Token

Update the `generate_jwt.py` script with your JWT secret:

1. **Open the file**:
   ```bash
   nano generate_jwt.py
   ```
   (or use your preferred text editor)

2. **Replace the JWT_SECRET** on line 11:
   ```python
   JWT_SECRET = "your-actual-secret-from-step-2"
   ```

3. **Save and close** the file

4. **Generate a token**:
   ```bash
   python generate_jwt.py test-user test@example.com customer-001
   ```

Expected output:
```
================================================================================
JWT Token Generated Successfully
================================================================================

User ID: test-user
Email: test@example.com
Customer ID: customer-001

Token (valid for 24 hours):
--------------------------------------------------------------------------------
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ0ZXN0LXVzZXIiLCJpc3MiOiJtZXNzYWdpbmctYXBpIiwiaWF0IjoxNzM3NjU4ODAwLCJleHAiOjE3Mzc3NDUyMDAsImVtYWlsIjoidGVzdEBleGFtcGxlLmNvbSIsImN1c3RvbWVyX2lkIjoiY3VzdG9tZXItMDAxIn0.abc123xyz...
--------------------------------------------------------------------------------

Use in API calls with header:
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
================================================================================
```

5. **Copy the token** (the long string after "Token (valid for 24 hours):")

### Step 11: Test Email Sending

Create a test file `test-email.json`:

```json
{
  "TraceId": "test-001",
  "EmailMessage": {
    "FromAddress": "your-verified-sender@example.com",
    "Subject": "Test Email from Messaging API"
  },
  "Addresses": {
    "your-verified-recipient@example.com": {
      "ChannelType": "EMAIL"
    }
  }
}
```

**Important:** Replace:
- `your-verified-sender@example.com` with your verified sender email
- `your-verified-recipient@example.com` with your verified recipient email

Send the test email:

```bash
curl -X POST "YOUR_API_ENDPOINT" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d @test-email.json
```

Replace:
- `YOUR_API_ENDPOINT` with your ApiEndpoint from Step 6
- `YOUR_JWT_TOKEN` with your token from Step 10

Expected response:
```json
{
  "status": "Message sent to queue"
}
```

### Step 12: Verify Email Was Received

1. **Check your recipient email inbox** (may take 10-30 seconds)
2. **Check spam folder** if not in inbox
3. **View Lambda logs** to see processing:
   ```bash
   aws logs tail /aws/lambda/MessageProcessor --follow --region us-east-1
   ```

Expected log output:
```
2024-01-23T10:30:45.123Z INFO Parsed message: {"TraceId": "test-001", ...}
2024-01-23T10:30:45.456Z INFO Email sent to your-verified-recipient@example.com: 01000123abc...
```

### Step 13: Test SMS Sending (Optional)

If you configured SMS in Step 8:

Create a test file `test-sms.json`:

```json
{
  "TraceId": "test-002",
  "SMSMessage": {
    "MessageType": "TRANSACTIONAL",
    "OriginationNumber": "YOUR_PHONE_NUMBER_OR_POOL_ID",
    "MessageBody": "Test SMS from Messaging API"
  },
  "Addresses": {
    "+1234567890": {
      "ChannelType": "SMS"
    }
  }
}
```

Replace:
- `YOUR_PHONE_NUMBER_OR_POOL_ID` with your phone number from Step 8
- `+1234567890` with your test phone number (must include country code)

Send the test SMS:

```bash
curl -X POST "YOUR_API_ENDPOINT" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d @test-sms.json
```

### Step 14: Test with Template

Create a test file `test-template.json`:

```json
{
  "TraceId": "test-003",
  "EmailMessage": {
    "FromAddress": "your-verified-sender@example.com",
    "TemplateName": "email-alert-template"
  },
  "Addresses": {
    "your-verified-recipient@example.com": {
      "ChannelType": "EMAIL",
      "Substitutions": {
        "productName": "CHEQUING",
        "membershipNumber": "****5493",
        "accountBalance": "100.00"
      }
    }
  }
}
```

Send the templated message:

```bash
curl -X POST "YOUR_API_ENDPOINT" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d @test-template.json
```

---

## Using the API

### Authentication

All API requests require a JWT token in the Authorization header:

```
Authorization: Bearer <your-jwt-token>
```

### API Endpoint

```
POST https://your-api-endpoint.execute-api.region.amazonaws.com/dev/
```

### Request Format

```json
{
  "TraceId": "unique-identifier",
  "EmailMessage": {
    "FromAddress": "sender@example.com",
    "Subject": "Subject line",
    "MessageBody": "Optional inline message",
    "TemplateName": "Optional template name",
    "Substitutions": {
      "variable": "value"
    }
  },
  "SMSMessage": {
    "MessageType": "TRANSACTIONAL",
    "OriginationNumber": "phone-number-or-pool-id",
    "MessageBody": "Optional inline message",
    "TemplateName": "Optional template name"
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

### Common Use Cases

#### 1. Send Email Only

```json
{
  "TraceId": "email-001",
  "EmailMessage": {
    "FromAddress": "alerts@example.com",
    "Subject": "Account Alert"
  },
  "Addresses": {
    "user@example.com": {
      "ChannelType": "EMAIL"
    }
  }
}
```

#### 2. Send SMS Only

```json
{
  "TraceId": "sms-001",
  "SMSMessage": {
    "MessageType": "TRANSACTIONAL",
    "OriginationNumber": "+18005551234",
    "MessageBody": "Your verification code is 123456"
  },
  "Addresses": {
    "+16045551234": {
      "ChannelType": "SMS"
    }
  }
}
```

#### 3. Send Both Email and SMS

```json
{
  "TraceId": "multi-001",
  "EmailMessage": {
    "FromAddress": "alerts@example.com",
    "Subject": "Account Alert"
  },
  "SMSMessage": {
    "MessageType": "TRANSACTIONAL",
    "OriginationNumber": "+18005551234",
    "MessageBody": "Check your email for important account information"
  },
  "Addresses": {
    "user@example.com": {
      "ChannelType": "EMAIL"
    },
    "+16045551234": {
      "ChannelType": "SMS"
    }
  }
}
```

#### 4. Use Template with Variables

```json
{
  "TraceId": "template-001",
  "EmailMessage": {
    "FromAddress": "alerts@example.com",
    "TemplateName": "email-alert-template"
  },
  "Addresses": {
    "user@example.com": {
      "ChannelType": "EMAIL",
      "Substitutions": {
        "productName": "SAVINGS",
        "membershipNumber": "****7890",
        "accountBalance": "50.00"
      }
    }
  }
}
```

### Integration Examples

#### Python

```python
import requests
import jwt
import datetime

# Configuration
API_ENDPOINT = "https://your-api-endpoint.execute-api.us-east-1.amazonaws.com/dev/"
JWT_SECRET = "your-jwt-secret"

# Generate token
def generate_token(user_id):
    payload = {
        'sub': user_id,
        'iss': 'messaging-api',
        'iat': datetime.datetime.utcnow(),
        'exp': datetime.datetime.utcnow() + datetime.timedelta(hours=24)
    }
    return jwt.encode(payload, JWT_SECRET, algorithm='HS256')

# Send message
def send_message(trace_id, from_email, to_email, subject):
    token = generate_token('user123')
    
    headers = {
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {token}'
    }
    
    payload = {
        'TraceId': trace_id,
        'EmailMessage': {
            'FromAddress': from_email,
            'Subject': subject
        },
        'Addresses': {
            to_email: {
                'ChannelType': 'EMAIL'
            }
        }
    }
    
    response = requests.post(API_ENDPOINT, headers=headers, json=payload)
    return response.json()

# Usage
result = send_message('test-001', 'sender@example.com', 'recipient@example.com', 'Test')
print(result)
```

#### Node.js

```javascript
const jwt = require('jsonwebtoken');
const axios = require('axios');

// Configuration
const API_ENDPOINT = 'https://your-api-endpoint.execute-api.us-east-1.amazonaws.com/dev/';
const JWT_SECRET = 'your-jwt-secret';

// Generate token
function generateToken(userId) {
  const payload = {
    sub: userId,
    iss: 'messaging-api',
    iat: Math.floor(Date.now() / 1000),
    exp: Math.floor(Date.now() / 1000) + (24 * 60 * 60)
  };
  
  return jwt.sign(payload, JWT_SECRET, { algorithm: 'HS256' });
}

// Send message
async function sendMessage(traceId, fromEmail, toEmail, subject) {
  const token = generateToken('user123');
  
  const response = await axios.post(
    API_ENDPOINT,
    {
      TraceId: traceId,
      EmailMessage: {
        FromAddress: fromEmail,
        Subject: subject
      },
      Addresses: {
        [toEmail]: {
          ChannelType: 'EMAIL'
        }
      }
    },
    {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`
      }
    }
  );
  
  return response.data;
}

// Usage
sendMessage('test-001', 'sender@example.com', 'recipient@example.com', 'Test')
  .then(result => console.log(result))
  .catch(error => console.error(error));
```

---

## Monitoring and Troubleshooting

### Check Queue Depth

Monitor how many messages are waiting to be processed:

```bash
aws sqs get-queue-attributes \
  --queue-url YOUR_QUEUE_URL \
  --attribute-names ApproximateNumberOfMessages \
  --region us-east-1
```

### Check Dead Letter Queue

See if any messages failed:

```bash
aws sqs get-queue-attributes \
  --queue-url YOUR_DLQ_URL \
  --attribute-names ApproximateNumberOfMessages \
  --region us-east-1
```

If this returns a number > 0, you have failed messages.

### View Lambda Logs

See what's happening during message processing:

```bash
# Message Processor logs
aws logs tail /aws/lambda/MessageProcessor --follow --region us-east-1

# Authorizer logs
aws logs tail /aws/lambda/JWTAuthorizer --follow --region us-east-1
```

### Common Issues and Solutions

#### Issue: 401 Unauthorized

**Symptoms:** API returns `{"message":"Unauthorized"}`

**Causes:**
- Missing Authorization header
- Invalid JWT token
- Expired token
- Wrong JWT secret

**Solutions:**
1. Verify Authorization header format: `Bearer <token>`
2. Generate a fresh token
3. Confirm JWT secret matches deployment
4. Check token hasn't expired (24 hours)

**Test:**
```bash
# Decode token to check expiration
python -c "import jwt; print(jwt.decode('YOUR_TOKEN', options={'verify_signature': False}))"
```

#### Issue: Email Not Sending

**Symptoms:** API succeeds but no email received

**Causes:**
- Email not verified in SES
- SES in sandbox mode
- Wrong sender email
- Email in spam folder

**Solutions:**
1. Verify sender email:
   ```bash
   aws sesv2 list-email-identities --region us-east-1
   ```
2. Check SES sandbox status:
   ```bash
   aws sesv2 get-account --region us-east-1
   ```
3. Check Lambda logs for errors
4. Check recipient spam folder

#### Issue: SMS Not Sending

**Symptoms:** API succeeds but no SMS received

**Causes:**
- Invalid phone number format
- Wrong origination number
- Phone number not registered
- Insufficient SMS quota

**Solutions:**
1. Verify phone number format (must include +country code)
2. Check origination number:
   ```bash
   aws pinpoint-sms-voice-v2 describe-phone-numbers --region us-east-1
   ```
3. Check Lambda logs for specific error
4. Verify SMS quota in AWS Console

#### Issue: Messages in Dead Letter Queue

**Symptoms:** DLQ has messages, CloudWatch alarm triggered

**Causes:**
- Invalid message format
- Service errors (SES/SMS)
- Lambda timeout
- Permission issues

**Solutions:**
1. View messages in DLQ:
   ```bash
   aws sqs receive-message \
     --queue-url YOUR_DLQ_URL \
     --max-number-of-messages 10 \
     --region us-east-1
   ```

2. Check Lambda logs for errors

3. Fix the underlying issue

4. Redrive messages back to main queue:
   ```bash
   # Via AWS Console: SQS → Select DLQ → Start DLQ redrive
   ```

#### Issue: Lambda Timeout

**Symptoms:** Messages fail after 5 minutes

**Causes:**
- Large batch size
- Slow SES/SMS API
- Network issues

**Solutions:**
1. Reduce batch size in template.yaml (default: 10)
2. Increase Lambda timeout (default: 300s)
3. Check AWS service health dashboard

### CloudWatch Alarms

The deployment includes a DLQ alarm. To receive notifications:

1. **Create an SNS topic**:
   ```bash
   aws sns create-topic --name messaging-api-alerts --region us-east-1
   ```

2. **Subscribe your email**:
   ```bash
   aws sns subscribe \
     --topic-arn arn:aws:sns:us-east-1:YOUR_ACCOUNT:messaging-api-alerts \
     --protocol email \
     --notification-endpoint your-email@example.com \
     --region us-east-1
   ```

3. **Confirm subscription** via email

4. **Update the alarm** to use your SNS topic:
   ```bash
   aws cloudwatch put-metric-alarm \
     --alarm-name my-messaging-api-DLQ-Messages \
     --alarm-actions arn:aws:sns:us-east-1:YOUR_ACCOUNT:messaging-api-alerts \
     --region us-east-1
   ```

---

## Production Checklist

Before going to production, ensure:

### Security

- [ ] Strong JWT secret generated (32+ characters)
- [ ] JWT secret stored securely (not in code)
- [ ] Different secrets for dev/staging/production
- [ ] API Gateway throttling configured
- [ ] CloudWatch alarms configured with SNS notifications
- [ ] IAM permissions reviewed (least privilege)

### Amazon SES

- [ ] Production access requested and approved
- [ ] Sender email addresses verified
- [ ] SPF/DKIM/DMARC configured for your domain
- [ ] Bounce and complaint handling configured
- [ ] Sending limits reviewed and increased if needed

### AWS End User Messaging SMS

- [ ] Phone number or sender ID registered
- [ ] 10DLC registration completed (US)
- [ ] SMS quota sufficient for your volume
- [ ] Opt-out handling implemented
- [ ] Compliance requirements met (TCPA, etc.)

### Monitoring

- [ ] CloudWatch alarms configured
- [ ] SNS notifications set up
- [ ] Log retention configured
- [ ] Dashboard created for key metrics
- [ ] On-call rotation established

### Testing

- [ ] Load testing completed
- [ ] Failure scenarios tested
- [ ] DLQ redrive process tested
- [ ] Token expiration tested
- [ ] Multi-channel sending tested

### Documentation

- [ ] API endpoint documented
- [ ] JWT secret location documented
- [ ] Runbook created for common issues
- [ ] Team trained on monitoring and troubleshooting
- [ ] Escalation procedures defined

### Backup and Recovery

- [ ] DynamoDB point-in-time recovery enabled
- [ ] CloudFormation template backed up
- [ ] Deployment process documented
- [ ] Rollback procedure tested

### Cost Management

- [ ] Cost estimates reviewed
- [ ] Budget alerts configured
- [ ] Resource tagging implemented
- [ ] Cost allocation tags applied

---

## Next Steps

Now that your messaging API is deployed and tested:

1. **Integrate with your application** using the examples in the "Using the API" section
2. **Create additional templates** for common message types
3. **Set up monitoring dashboards** in CloudWatch
4. **Configure production access** for SES if sending to unverified emails
5. **Implement error handling** in your application for API failures
6. **Plan for scaling** by monitoring queue depth and Lambda concurrency

## Support Resources

- **README.md** - Complete API documentation
- **AUTHENTICATION.md** - Detailed JWT authentication guide
- **DEPLOYMENT.md** - Quick deployment reference
- **AWS Documentation**:
  - [Amazon SQS](https://docs.aws.amazon.com/sqs/)
  - [AWS Lambda](https://docs.aws.amazon.com/lambda/)
  - [Amazon SES](https://docs.aws.amazon.com/ses/)
  - [AWS End User Messaging](https://docs.aws.amazon.com/sms-voice/)
  - [API Gateway](https://docs.aws.amazon.com/apigateway/)

## Cleanup (If Needed)

To remove all resources and stop incurring charges:

```bash
sam delete --stack-name my-messaging-api --region us-east-1
```

This will delete:
- API Gateway
- Lambda functions
- SQS queues
- DynamoDB table
- CloudWatch alarms
- IAM roles

**Note:** The Secrets Manager secret will be scheduled for deletion (7-30 days) but not immediately deleted.

---

**Congratulations!** You've successfully deployed and configured the SQS-Based Messaging Solution. You're now ready to send emails and SMS messages through a secure, scalable API.

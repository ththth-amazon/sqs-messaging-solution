# SQS-Based Multi-Channel Messaging Solution

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![AWS SAM](https://img.shields.io/badge/AWS-SAM-orange.svg)](https://aws.amazon.com/serverless/sam/)
[![Python 3.12](https://img.shields.io/badge/python-3.12-blue.svg)](https://www.python.org/downloads/)

A production-ready, serverless messaging API built on AWS that enables secure multi-channel notifications (email and SMS) with automatic retries, dead letter queue handling, and JWT authentication.

![Architecture Diagram](architecture-diagram.png)

## 🌟 Features

- **🔐 JWT Authentication**: Secure API access with tokens stored in AWS Secrets Manager
- **🔄 Automatic Retries**: Failed messages retry up to 3 times before moving to DLQ
- **📊 Partial Batch Failures**: Only failed messages retry, not entire batches
- **📝 Template Management**: Store reusable message templates in DynamoDB
- **📧 Multi-Channel**: Send via Amazon SES (email) and AWS End User Messaging (SMS)
- **📈 Monitoring**: CloudWatch alarms alert on failed messages
- **⚡ Serverless**: Pay only for what you use, scales automatically
- **🏗️ Infrastructure as Code**: Deploy with AWS SAM in minutes

## 📋 Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Deployment](#deployment)
- [Usage](#usage)
- [API Reference](#api-reference)
- [Monitoring](#monitoring)
- [Cost Estimation](#cost-estimation)
- [Security](#security)
- [Contributing](#contributing)
- [License](#license)

## 🏗️ Architecture

The solution uses a decoupled, event-driven architecture:

1. **Client** sends authenticated requests with JWT tokens
2. **API Gateway** validates tokens via Lambda Authorizer
3. **SQS Queue** buffers messages for reliable processing
4. **Lambda Processor** sends messages via SES/SMS
5. **Dead Letter Queue** captures failed messages after 3 retries
6. **CloudWatch** monitors and alerts on failures

### AWS Services Used

- **API Gateway**: REST API endpoint with custom authorizer
- **AWS Lambda**: Serverless compute for authorization and message processing
- **Amazon SQS**: Message queuing with DLQ for reliability
- **AWS Secrets Manager**: Secure JWT secret storage
- **Amazon DynamoDB**: Template storage
- **Amazon SES**: Email delivery
- **AWS End User Messaging**: SMS delivery
- **Amazon CloudWatch**: Monitoring and alarms

## 📦 Prerequisites

- AWS Account with appropriate permissions
- [AWS CLI](https://aws.amazon.com/cli/) configured
- [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html) installed
- Python 3.12 or later
- Verified email addresses in Amazon SES
- (Optional) SMS phone number in AWS End User Messaging

## 🚀 Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/sqs-messaging-solution.git
cd sqs-messaging-solution/customer-package
```

### 2. Generate JWT Secret

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

Save this secret - you'll need it for deployment.

### 3. Build and Deploy

```bash
sam build
sam deploy --guided
```

Follow the prompts and paste your JWT secret when asked.

### 4. Test the API

```bash
# Generate a test token
python generate_jwt.py test-user test@example.com

# Send a test email
curl -X POST "YOUR_API_ENDPOINT" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d '{
    "TraceId": "test-001",
    "EmailMessage": {
      "FromAddress": "sender@example.com",
      "Subject": "Test Email"
    },
    "Addresses": {
      "recipient@example.com": {
        "ChannelType": "EMAIL"
      }
    }
  }'
```

## 📖 Deployment

For detailed deployment instructions, see [CUSTOMER_DEPLOYMENT_GUIDE.md](CUSTOMER_DEPLOYMENT_GUIDE.md).

### Configuration Parameters

| Parameter | Description | Default | Required |
|-----------|-------------|---------|----------|
| JWTSecret | Secret key for JWT validation | (must change) | Yes |
| StackName | CloudFormation stack name | - | Yes |
| Region | AWS region for deployment | us-east-1 | Yes |

### Post-Deployment Steps

1. **Verify email addresses** in Amazon SES
2. **Configure SMS** phone number (if using SMS)
3. **Add message templates** to DynamoDB (optional)
4. **Set up CloudWatch alarms** with SNS notifications

## 💻 Usage

### Authentication

All API requests require a JWT token:

```bash
Authorization: Bearer <your-jwt-token>
```

Generate tokens using the provided script:

```bash
python generate_jwt.py <user_id> [email] [customer_id]
```

### Send Email

```json
{
  "TraceId": "unique-id",
  "EmailMessage": {
    "FromAddress": "sender@example.com",
    "Subject": "Your Subject"
  },
  "Addresses": {
    "recipient@example.com": {
      "ChannelType": "EMAIL"
    }
  }
}
```

### Send SMS

```json
{
  "TraceId": "unique-id",
  "SMSMessage": {
    "MessageType": "TRANSACTIONAL",
    "OriginationNumber": "+18005551234",
    "MessageBody": "Your message"
  },
  "Addresses": {
    "+16045551234": {
      "ChannelType": "SMS"
    }
  }
}
```

### Use Templates

```json
{
  "TraceId": "unique-id",
  "EmailMessage": {
    "FromAddress": "sender@example.com",
    "TemplateName": "alert-template"
  },
  "Addresses": {
    "recipient@example.com": {
      "ChannelType": "EMAIL",
      "Substitutions": {
        "productName": "SAVINGS",
        "accountBalance": "100.00"
      }
    }
  }
}
```

## 📚 API Reference

For complete API documentation, see [customer-package/README.md](customer-package/README.md).

### Endpoints

- `POST /dev/` - Send message (email, SMS, or both)

### Request Format

```json
{
  "TraceId": "string (required)",
  "EmailMessage": {
    "FromAddress": "string (required)",
    "Subject": "string (optional)",
    "MessageBody": "string (optional)",
    "TemplateName": "string (optional)",
    "Substitutions": {}
  },
  "SMSMessage": {
    "MessageType": "TRANSACTIONAL|PROMOTIONAL",
    "OriginationNumber": "string (required)",
    "MessageBody": "string (optional)",
    "TemplateName": "string (optional)"
  },
  "Addresses": {
    "recipient": {
      "ChannelType": "EMAIL|SMS",
      "Substitutions": {}
    }
  }
}
```

### Response Format

```json
{
  "status": "Message sent to queue"
}
```

## 📊 Monitoring

### CloudWatch Metrics

The solution automatically publishes metrics:

- **SQS**: Queue depth, message age, messages sent/received
- **Lambda**: Invocations, duration, errors, throttles
- **API Gateway**: Request count, latency, errors

### View Logs

```bash
# Message Processor logs
aws logs tail /aws/lambda/MessageProcessor --follow

# Authorizer logs
aws logs tail /aws/lambda/JWTAuthorizer --follow
```

### Check Queue Status

```bash
# Main queue
aws sqs get-queue-attributes \
  --queue-url YOUR_QUEUE_URL \
  --attribute-names ApproximateNumberOfMessages

# Dead letter queue
aws sqs get-queue-attributes \
  --queue-url YOUR_DLQ_URL \
  --attribute-names ApproximateNumberOfMessages
```

### Handle Failed Messages

```bash
# View messages in DLQ
aws sqs receive-message \
  --queue-url YOUR_DLQ_URL \
  --max-number-of-messages 10

# Redrive from DLQ (via AWS Console)
# SQS → Select DLQ → Start DLQ redrive
```

## 💰 Cost Estimation

Estimated costs for 1 million messages per month (US sending only):

| Service | Usage | Monthly Cost |
|---------|-------|--------------|
| API Gateway | 1M requests | $3.50 |
| Amazon SQS | 1M messages | $0.40 |
| AWS Lambda | 1M invocations | $2.50 |
| Amazon SES | 1M emails | $100.00 |
| AWS End User Messaging | 1M SMS | $10,020.00 |
| **Total** | | **$10,126.40** |

**Note**: SMS costs vary significantly by destination country. Email costs assume $0.10 per 1,000 emails.

## 🔒 Security

### Best Practices Implemented

- ✅ JWT authentication for all API requests
- ✅ Secrets stored in AWS Secrets Manager (encrypted)
- ✅ IAM least privilege for Lambda functions
- ✅ HTTPS-only API Gateway
- ✅ Authorization caching (5 minutes)
- ✅ No hardcoded credentials

### Security Recommendations

1. **Use strong JWT secrets** (32+ characters, random)
2. **Rotate secrets periodically** (every 90 days)
3. **Enable CloudTrail** for audit logging
4. **Set up budget alerts** to detect anomalies
5. **Use different secrets** for dev/staging/production
6. **Enable MFA** on AWS accounts
7. **Review IAM permissions** regularly

## 🤝 Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for details.

### Development Setup

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/sqs-messaging-solution.git
cd sqs-messaging-solution

# Install dependencies
pip install -r customer-package/lambda/requirements.txt
pip install pytest boto3 moto

# Run tests
pytest tests/

# Build
sam build

# Deploy to dev
sam deploy --config-env dev
```

### Reporting Issues

Please use GitHub Issues to report bugs or request features. Include:
- Description of the issue
- Steps to reproduce
- Expected vs actual behavior
- AWS region and SAM CLI version

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 📚 Additional Resources

- [Complete Deployment Guide](CUSTOMER_DEPLOYMENT_GUIDE.md)
- [Authentication Guide](customer-package/AUTHENTICATION.md)
- [API Documentation](customer-package/README.md)
- [AWS Blog Post](aws-blog-post.md)
- [Architecture Diagram](architecture-diagram.drawio)

### AWS Documentation

- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/sqs/)
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/)
- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/)
- [AWS End User Messaging](https://docs.aws.amazon.com/sms-voice/)
- [AWS SAM Documentation](https://docs.aws.amazon.com/serverless-application-model/)

## 🙏 Acknowledgments

Built with AWS serverless services following the [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/).

## 📞 Support

For questions or support:
- Open an [issue](https://github.com/YOUR_USERNAME/sqs-messaging-solution/issues)
- Check the [documentation](customer-package/README.md)
- Review [troubleshooting guide](CUSTOMER_DEPLOYMENT_GUIDE.md#monitoring-and-troubleshooting)

---

**Made with ❤️ using AWS Serverless**

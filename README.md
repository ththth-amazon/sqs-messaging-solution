# Customer Distribution Package

This folder contains everything needed to distribute the messaging solution to customers.

## Contents

### Ready to Send

**📦 sqs-messaging-solution-customer-package.zip**
- Complete, production-ready package
- Send this file to customers
- Contains everything they need to deploy

### Source Files (customer-package folder)

The unzipped contents include:

- **README.md** - Complete API documentation (generic, no specific details)
- **AUTHENTICATION.md** - Comprehensive JWT authentication guide
- **DEPLOYMENT.md** - Step-by-step deployment instructions
- **template.yaml** - SAM template with all infrastructure
- **lambda/** - Lambda function code
  - `authorizer.py` - JWT authorizer
  - `message_processor.py` - Message processor
  - `requirements.txt` - Python dependencies
- **generate_jwt.py** - Helper script for generating JWT tokens
- **.gitignore** - Git ignore file

## What Customers Get

When they deploy, they get their own isolated:
- ✅ API Gateway endpoint (with JWT authentication)
- ✅ Lambda Authorizer
- ✅ Message Processor Lambda
- ✅ SQS queues (main + dead letter)
- ✅ DynamoDB table for templates
- ✅ Secrets Manager secret (their own)
- ✅ CloudWatch monitoring and alarms

## Customer Deployment Process

1. Extract the zip file
2. Generate a strong JWT secret
3. Run `sam build`
4. Run `sam deploy --guided`
5. Configure SES email addresses
6. (Optional) Configure SMS pool
7. Test with provided scripts

## Security Notes

- Each customer deployment is completely isolated
- They set their own JWT secret during deployment
- Their secret is stored in their AWS Secrets Manager
- No shared resources between deployments

## Updating the Package

If you make changes to the source code:

1. Update files in the main project folder
2. Copy updated files to `customer-package/`
3. Recreate the zip:
   ```bash
   Compress-Archive -Path customer-package/* -DestinationPath sqs-messaging-solution-customer-package.zip -Force
   ```

## Support

For customer support questions, refer them to:
- `README.md` - API usage and examples
- `AUTHENTICATION.md` - JWT authentication details
- `DEPLOYMENT.md` - Deployment troubleshooting

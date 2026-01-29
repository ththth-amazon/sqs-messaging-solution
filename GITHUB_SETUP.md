# GitHub Setup Instructions

## Step 1: Create GitHub Repository

1. Go to https://github.com/ththth-amazon
2. Click the "+" icon in the top right
3. Select "New repository"
4. Fill in:
   - **Repository name**: `sqs-messaging-solution` (or your preferred name)
   - **Description**: `Production-ready serverless messaging API with SQS, Lambda, and multi-channel support`
   - **Visibility**: Public (for open source) or Private
   - **DO NOT** initialize with README, .gitignore, or license (we already have these)
5. Click "Create repository"

## Step 2: Connect Local Repository to GitHub

After creating the repository on GitHub, run these commands:

```powershell
# Add the remote repository
git remote add origin https://github.com/ththth-amazon/sqs-messaging-solution.git

# Push to GitHub
git push -u origin main
```

## Step 3: Authentication

When you push for the first time, Git Credential Manager will:
1. Open a browser window
2. Ask you to sign in to GitHub
3. Store your credentials securely

## Step 4: Verify

After pushing, visit:
https://github.com/ththth-amazon/sqs-messaging-solution

You should see all your files!

## Next Steps

1. **Add repository description** on GitHub
2. **Add topics/tags**: `aws`, `serverless`, `sqs`, `lambda`, `messaging`, `api-gateway`
3. **Enable GitHub Actions** (already configured in `.github/workflows/ci.yml`)
4. **Add secrets** for CI/CD:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
   - `JWT_SECRET_DEV`
   - `JWT_SECRET_PROD`
5. **Create first release**: Go to Releases → Draft a new release → v1.0.0

## Troubleshooting

### If git command not found
Restart your terminal or run:
```powershell
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
```

### If authentication fails
Use a Personal Access Token:
1. Go to https://github.com/settings/tokens
2. Generate new token (classic)
3. Select scopes: `repo`, `workflow`
4. Use token as password when prompted

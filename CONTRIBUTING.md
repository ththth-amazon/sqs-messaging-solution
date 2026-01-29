# Contributing to SQS-Based Multi-Channel Messaging Solution

Thank you for your interest in contributing! This document provides guidelines and instructions for contributing to this project.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Development Setup](#development-setup)
- [How to Contribute](#how-to-contribute)
- [Coding Standards](#coding-standards)
- [Testing](#testing)
- [Pull Request Process](#pull-request-process)
- [Reporting Bugs](#reporting-bugs)
- [Suggesting Enhancements](#suggesting-enhancements)

## Code of Conduct

This project adheres to a code of conduct. By participating, you are expected to uphold this code:

- Be respectful and inclusive
- Welcome newcomers and help them learn
- Focus on what is best for the community
- Show empathy towards other community members

## Getting Started

1. Fork the repository on GitHub
2. Clone your fork locally
3. Create a new branch for your contribution
4. Make your changes
5. Test your changes
6. Submit a pull request

## Development Setup

### Prerequisites

- AWS Account (for testing)
- AWS CLI configured
- AWS SAM CLI installed
- Python 3.12 or later
- Git

### Local Setup

```bash
# Clone your fork
git clone https://github.com/YOUR_USERNAME/sqs-messaging-solution.git
cd sqs-messaging-solution

# Create a virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r customer-package/lambda/requirements.txt

# Install development dependencies
pip install pytest pytest-cov moto boto3 black flake8 mypy
```

### Running Locally

```bash
# Build the SAM application
sam build

# Run local API
sam local start-api

# Invoke function locally
sam local invoke MessageProcessorFunction -e events/test-event.json
```

## How to Contribute

### Types of Contributions

We welcome various types of contributions:

- **Bug fixes**: Fix issues in the code
- **Features**: Add new functionality
- **Documentation**: Improve or add documentation
- **Tests**: Add or improve test coverage
- **Examples**: Add usage examples
- **Performance**: Optimize code performance
- **Security**: Improve security

### Contribution Workflow

1. **Check existing issues** to see if your contribution is already being worked on
2. **Create an issue** if one doesn't exist (for bugs or features)
3. **Fork the repository** and create a branch
4. **Make your changes** following our coding standards
5. **Write tests** for your changes
6. **Update documentation** as needed
7. **Submit a pull request**

## Coding Standards

### Python Style Guide

We follow [PEP 8](https://www.python.org/dev/peps/pep-0008/) with some modifications:

- Line length: 100 characters (not 79)
- Use type hints where appropriate
- Use docstrings for functions and classes

### Code Formatting

Use `black` for automatic formatting:

```bash
black customer-package/lambda/
```

### Linting

Use `flake8` for linting:

```bash
flake8 customer-package/lambda/ --max-line-length=100
```

### Type Checking

Use `mypy` for type checking:

```bash
mypy customer-package/lambda/
```

### Code Structure

```python
"""
Module docstring explaining the purpose.
"""
import standard_library
import third_party
import local_modules


def function_name(param: str) -> dict:
    """
    Function docstring explaining what it does.
    
    Args:
        param: Description of parameter
        
    Returns:
        Description of return value
        
    Raises:
        ExceptionType: When this exception is raised
    """
    # Implementation
    pass
```

## Testing

### Writing Tests

- Write unit tests for all new functions
- Write integration tests for Lambda handlers
- Use `pytest` for testing
- Use `moto` for mocking AWS services

### Test Structure

```python
import pytest
from moto import mock_sqs, mock_dynamodb
from lambda.message_processor import lambda_handler


@mock_sqs
@mock_dynamodb
def test_message_processor_success():
    """Test successful message processing."""
    # Arrange
    event = {...}
    context = {}
    
    # Act
    result = lambda_handler(event, context)
    
    # Assert
    assert result['statusCode'] == 200
```

### Running Tests

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=customer-package/lambda --cov-report=html

# Run specific test file
pytest tests/test_message_processor.py

# Run specific test
pytest tests/test_message_processor.py::test_message_processor_success
```

### Test Coverage

- Aim for at least 80% code coverage
- All new code should have tests
- Critical paths should have 100% coverage

## Pull Request Process

### Before Submitting

1. **Update documentation** if you changed functionality
2. **Add tests** for new features or bug fixes
3. **Run all tests** and ensure they pass
4. **Format code** with `black`
5. **Lint code** with `flake8`
6. **Update CHANGELOG.md** with your changes

### PR Title Format

Use conventional commit format:

- `feat: Add SMS template support`
- `fix: Resolve DLQ redrive issue`
- `docs: Update deployment guide`
- `test: Add integration tests for authorizer`
- `refactor: Simplify message processing logic`
- `chore: Update dependencies`

### PR Description Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing completed

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] Tests pass locally
- [ ] No new warnings
```

### Review Process

1. Maintainers will review your PR
2. Address any feedback or requested changes
3. Once approved, a maintainer will merge your PR
4. Your contribution will be included in the next release

## Reporting Bugs

### Before Reporting

1. Check if the bug has already been reported
2. Verify it's actually a bug (not expected behavior)
3. Test with the latest version

### Bug Report Template

```markdown
**Describe the bug**
Clear description of what the bug is

**To Reproduce**
Steps to reproduce:
1. Deploy with '...'
2. Send request '...'
3. See error

**Expected behavior**
What you expected to happen

**Actual behavior**
What actually happened

**Environment**
- AWS Region: [e.g., us-east-1]
- SAM CLI version: [e.g., 1.100.0]
- Python version: [e.g., 3.12]
- OS: [e.g., macOS 14.0]

**Logs**
```
Paste relevant logs here
```

**Additional context**
Any other relevant information
```

## Suggesting Enhancements

### Enhancement Request Template

```markdown
**Is your feature request related to a problem?**
Clear description of the problem

**Describe the solution you'd like**
Clear description of what you want to happen

**Describe alternatives you've considered**
Other solutions you've thought about

**Additional context**
Any other relevant information

**Would you like to implement this?**
- [ ] Yes, I'd like to implement this
- [ ] No, just suggesting
```

## Development Guidelines

### Adding New Features

1. **Discuss first**: Open an issue to discuss the feature
2. **Keep it simple**: Follow KISS principle
3. **Maintain compatibility**: Don't break existing functionality
4. **Document thoroughly**: Update all relevant documentation
5. **Test extensively**: Add comprehensive tests

### Modifying Infrastructure

When changing `template.yaml`:

1. Test deployment in a dev environment
2. Document any new parameters
3. Update deployment guide
4. Consider backward compatibility
5. Test rollback scenarios

### Security Considerations

- Never commit secrets or credentials
- Use AWS Secrets Manager for sensitive data
- Follow AWS security best practices
- Report security issues privately (see SECURITY.md)

## Release Process

Maintainers follow this process for releases:

1. Update version in relevant files
2. Update CHANGELOG.md
3. Create release branch
4. Test thoroughly
5. Create GitHub release with notes
6. Update customer package zip
7. Announce release

## Questions?

If you have questions:

1. Check existing documentation
2. Search closed issues
3. Open a new issue with the "question" label
4. Join our community discussions

## Recognition

Contributors will be:
- Listed in CONTRIBUTORS.md
- Mentioned in release notes
- Credited in the project

Thank you for contributing! 🎉

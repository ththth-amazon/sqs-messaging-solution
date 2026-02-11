# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Optional SES Configuration Set parameter for email tracking and analytics
- Environment variable `SES_CONFIGURATION_SET` to configure SES configuration set at deployment time
- Optional SMS Configuration Set parameter for SMS tracking and analytics
- Environment variable `SMS_CONFIGURATION_SET` to configure SMS configuration set at deployment time

### Fixed
- Fixed deprecation warning in `generate_jwt.py` by replacing `datetime.datetime.utcnow()` with `datetime.datetime.now(datetime.UTC)` for timezone-aware datetime objects
- Fixed IAM permissions for Lambda to access SES configuration sets by adding explicit `ses:SendEmail` and `ses:SendRawEmail` permissions with wildcard resource access

### Changed
- Updated README.md to document JWTSecret minimum length requirement (16 characters)

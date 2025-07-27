# AWS Sandbox Nuke

An automated AWS resource cleanup tool using [aws-nuke](https://github.com/ekristen/aws-nuke) with GitHub Actions workflows for scheduled and on-demand execution.

## Overview

This project provides automated AWS resource cleanup for sandbox environments using aws-nuke. It includes:

- **Scheduled cleanup**: Automatic nightly cleanup at midnight Paris time
- **On-demand cleanup**: Manual execution with dry-run option
- **Pushover notifications**: Success/failure notifications via Pushover
- **Safety filters**: Protects critical AWS SSO and SAML resources

## Workflows

### 1. Scheduled Nuke ([.github/workflows/nuke-scheduled.yaml](.github/workflows/nuke-scheduled.yaml))
- **Trigger**: Daily at `22:00 UTC` (midnight Paris time)
- **Mode**: Always runs with `--no-dry-run` (real deletions)
- **Purpose**: Automated nightly cleanup of sandbox resources

### 2. On-Demand Nuke ([.github/workflows/nuke.yaml](.github/workflows/nuke.yaml))
- **Trigger**: Manual workflow dispatch
- **Mode**: Dry-run by default, optional `--no-dry-run`
- **Purpose**: Testing and manual cleanup with safety controls

### 3. Pushover Notifications ([.github/workflows/pushover-notification.yaml](.github/workflows/pushover-notification.yaml))
- **Trigger**: Repository dispatch from other workflows
- **Purpose**: Send success/failure notifications via Pushover

## Configuration

### AWS Nuke Config ([aws-nuke-config.yaml](aws-nuke-config.yaml))

The configuration includes several protection presets:

- **ignore-tag**: Resources tagged with `ignore-nuke: "true"` are preserved
- **sso**: Protects AWS SSO resources and roles
- **saml**: Protects SAML federation and Azure AD integration resources

**Protected Resources:**
- AWS SSO instances, assignments, and permission sets
- Azure AD federated roles and policies
- Lambda functions for IdP integration
- CloudFormation stacks for federation setup

**Regions:** `global`, `eu-west-1`

## Required Secrets

Configure these secrets in your GitHub repository:

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS access key for aws-nuke user |
| `AWS_SECRET_KEY` | AWS secret key for aws-nuke user |
| `AWS_ACCOUNT_IDS` | Target AWS account ID |
| `SECRET_GITHUB` | GitHub token for triggering workflows |
| `PUSHOVER_USER_KEY` | Pushover user key for notifications |
| `PUSHOVER_API_TOKEN` | Pushover API token for notifications |

## Usage

### Manual Execution

1. Go to the **Actions** tab in GitHub
2. Select **"AWS Nuke On Demand"**
3. Click **"Run workflow"**
4. Choose whether to run in dry-run mode (default) or with real deletions

### Dry Run (Safe)
```bash
# This is the default - no resources are actually deleted
aws-nuke nuke -c aws-nuke-config.yaml --no-alias-check --force
```

### Real Execution (Destructive)
```bash
# Check the "no-dry-run" option in the workflow dispatch
aws-nuke nuke -c aws-nuke-config.yaml --no-dry-run --no-alias-check --force
```

### Testing Notifications

You can manually trigger the notification workflow using the curl command:

```bash
curl -X POST \
    -H "Authorization: token YOUR_GITHUB_TOKEN" \
    -H "Accept: application/vnd.github.v3+json" \
    https://api.github.com/repos/YOUR_REPO/dispatches \
    -d "$(jq -n \
        --arg event_type "pushover-notification" \
        --argjson client_payload "{\"failed\": \"true\"}" \
        '{event_type: $event_type, client_payload: $client_payload}')"
```

## Safety Features

1. **Dry-run by default**: Manual executions default to dry-run mode
2. **Resource filtering**: Critical infrastructure is protected via presets
3. **Tag-based exclusion**: Resources can be tagged `ignore-nuke: "true"`
4. **Notifications**: Always notified of execution results
5. **Account validation**: Only specified accounts can be targeted

## AWS IAM Requirements

The AWS user/role needs the following permissions:
- Full access to resources you want to delete
- `iam:ListAccountAliases` for alias checking
- Read permissions for resource discovery

⚠️ **Warning**: This tool is designed for sandbox/development environments. Never run against production accounts.

## Monitoring

- Check the **Actions** tab for workflow execution history
- Pushover notifications provide immediate success/failure alerts
- Workflow logs contain detailed information about what was/would be deleted

## Customization

To modify what gets deleted or protected:

1. Edit the presets in [aws-nuke-config.yaml](aws-nuke-config.yaml)
2. Add new resource filters as needed
3. Update the regions list if targeting different regions
4. Modify the schedule in [.github/workflows/nuke-scheduled.yaml](.github/workflows/nuke-scheduled.yaml)
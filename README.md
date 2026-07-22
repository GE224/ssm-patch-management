# ssm-patch-management
AWS Systems Manager patch management lab - IAM role, patch baseline, maintenance window, Session Manager, and two deliberate break/debug exercises.

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for both break/debug walkthroughs:
1. IAM instance profile removal (instance connectivity failure — SSM `PingStatus` gave a false positive)
2. Bad `send-command` parameter on `AWS-ConfigureAWSPackage` (single command failure on a healthy, fully-online instance)

# Lab 2: Answers

## 1. Why are the credentials stored in a Secret and not in the ConfigMap?
A ConfigMap is meant for non-sensitive configuration stored in plain text and visible in clear text via `kubectl describe`. A Secret is dedicated to sensitive data, providing better security controls such as RBAC restrictions, omission from standard descriptions, and storage in memory or tmpfs.

## 2. What happens if the Job is executed again tomorrow? How would a production platform provide credentials?
Since Onyxia credentials are temporary, they will have expired by tomorrow, causing `aws s3 cp` to fail with an authentication error. A production platform avoids manual credential copying by using Workload Identity or a secure secrets manager with automatic rotation.

## 3. How would you turn this Job into a daily ingestion?
You can replace the `Job` resource with a `CronJob` object by specifying a schedule (e.g., `"0 2 * * *"`) and moving the pod template under `jobTemplate`. In a real-world pipeline, you would also read directly from the source system, ensure idempotency using date-partitioned paths, and set up monitoring alerts.

# 1. Scenario:

Your application pod in Kubernetes is failing to connect to an RDS MySQL database. How will you troubleshoot?

Answer:
First, I’d verify connectivity. From within the pod, I’d run:

kubectl exec -it <pod-name> -- nslookup <rds-endpoint>


to check DNS resolution. Then, I’d test connectivity using:

kubectl exec -it <pod-name> -- nc -zv <rds-endpoint> 3306


If that fails, I’d check the RDS security group inbound rules to allow the EKS node subnet CIDRs. I’d also confirm that the database is in the same VPC or peered VPC and that there’s no Network ACL blocking traffic. Finally, I’d verify the credentials and secrets in Kubernetes using:

kubectl get secret db-credentials -o yaml

# 2. Scenario:

You need to automate RDS instance creation and parameter setup using Terraform. How would you design it?

Answer:
I’d use the aws_db_instance resource to create the RDS, along with a db_subnet_group for subnet placement. Then, I’d externalize the configuration in variables.tf (like engine version, instance size, etc.) and define a parameter_group for custom MySQL settings.
To ensure safety, I’d add lifecycle rules:

lifecycle {
  prevent_destroy = true
}


and enable automated backups via:

backup_retention_period = 7

# 3. Scenario:

Your database disk usage is growing unexpectedly fast. What steps will you take?

Answer:
First, I’d identify large tables using queries like:

SELECT table_schema, table_name, ROUND((data_length + index_length)/1024/1024, 2) AS size_mb
FROM information_schema.tables
ORDER BY size_mb DESC;


Then, I’d check for old logs or binary logs not purged, and enable expire_logs_days. On the DevOps side, I’d set up CloudWatch metrics for FreeStorageSpace and create an alarm. If growth is justified, I’d plan to scale up storage or switch to gp3 volume type for elasticity.

# 4. Scenario:

How do you securely store and manage database credentials in a CI/CD pipeline?

Answer:
I’d never hardcode credentials. Instead, I’d store them in AWS Secrets Manager or HashiCorp Vault, then inject them dynamically at runtime.
In Jenkins, I’d use Credentials Binding Plugin, and in GitHub Actions, I’d use secrets with environment variables. For Kubernetes, I’d create a Secret and mount it as an environment variable.

5. Scenario:

You need to deploy a new version of the database schema using a CI/CD pipeline. How would you do it safely?

Answer:
I’d use Flyway or Liquibase to version-control database changes. The pipeline would:

Take a backup.

Run migrations in a staging environment first.

Validate schema changes.

Deploy migrations to production using blue-green or canary style if possible.

Rollback scripts would also be versioned to handle failures safely.

6. Scenario:

A developer accidentally deleted a table in the production database. How would you recover it?

Answer:
If automated backups or point-in-time recovery (PITR) are enabled, I’d restore a new DB instance to the timestamp before deletion, export the deleted table using mysqldump, and import it back into production.
Example:

mysqldump -h restored-db-endpoint -u user -p db_name deleted_table > deleted_table.sql
mysql -h prod-db-endpoint -u user -p db_name < deleted_table.sql

7. Scenario:

How do you ensure high availability of your database?

Answer:
For AWS RDS, I’d enable Multi-AZ deployment — this creates a standby replica in another Availability Zone. I’d also use read replicas for scaling reads. Additionally, I’d monitor health with CloudWatch alarms and route traffic through a failover mechanism using RDS endpoint.

8. Scenario:

You are asked to migrate an on-prem MySQL database to AWS RDS. What’s your approach?

Answer:
I’d use AWS Database Migration Service (DMS) for near-zero downtime migration.
Steps:

Create a replication instance.

Configure source (on-prem) and target (RDS) endpoints.

Run schema conversion (if needed) using AWS Schema Conversion Tool (SCT).

Test and validate migration.

Cut over once the delta sync completes.

9. Scenario:

Your database query latency suddenly increased. How would you diagnose it?

Answer:
I’d check CloudWatch metrics for CPU, IOPS, and connections. Then, I’d enable Performance Insights to identify slow queries. If queries are suboptimal, I’d analyze EXPLAIN plans and optimize indexes.
From the DevOps side, I’d ensure that the DB instance isn’t under-provisioned and autoscaling is configured correctly.

10. Scenario:

How would you automate database backups and verify their integrity?

Answer:
I’d schedule automatic backups using AWS RDS built-in snapshot features or cron-based scripts for EC2-hosted DBs.
To verify integrity, I’d periodically restore the snapshot to a test environment and run checksum validation to ensure data consistency.

11. Scenario:

You need to create multiple database users with different privileges. How would you handle it in Terraform?

Answer:
I’d use the mysql_user and mysql_grant resources from the Terraform MySQL provider.
For example:

resource "mysql_user" "app_user" {
  user = "appuser"
  host = "%"
  plaintext_password = var.app_password
}
resource "mysql_grant" "app_grant" {
  user       = mysql_user.app_user.user
  host       = "%"
  database   = "app_db"
  privileges = ["SELECT", "INSERT", "UPDATE"]
}


This ensures version-controlled, auditable user management.

12. Scenario:

Your RDS CPU usage is consistently high. What actions will you take?

Answer:
I’d first identify whether the CPU spike is due to queries or instance size. Enable Performance Insights to identify heavy queries.
If the workload is legitimate, I’d scale vertically (increase instance size) or horizontally (use read replicas). I’d also review indexes and connection pooling in the application.

13. Scenario:

How would you connect an EKS application to an RDS database securely?

Answer:
I’d place RDS in private subnets and allow inbound traffic only from the EKS node group security group.
In the app Deployment YAML, I’d inject credentials via Kubernetes Secrets, and access the DB through its internal endpoint — not public.

14. Scenario:

How do you handle schema drift in DevOps environments?

Answer:
I’d use schema migration tools like Flyway, which maintain a changelog of executed scripts. The CI/CD pipeline would check whether new migrations exist and apply them automatically, preventing out-of-sync environments.

15. Scenario:

How would you monitor database health in AWS?

Answer:
I’d use Amazon CloudWatch metrics for CPU, free storage, connections, and latency.
I’d also enable Enhanced Monitoring and Performance Insights for deeper analysis.
Custom dashboards can be created in Grafana connected to CloudWatch data sources.

16. Scenario:

If RDS becomes unavailable in one region, how will you ensure business continuity?

Answer:
I’d configure cross-region read replicas and use Route 53 failover routing.
In disaster recovery, promote the replica to master and update application connection strings automatically via environment variables or parameter store.

17. Scenario:

How can you run database migrations automatically during deployment?

Answer:
In Jenkins or GitHub Actions, after the build and before application deployment, I’d add a stage to run migration tools (Flyway/Liquibase) with DB credentials fetched securely.
Example in GitHub Actions:

- name: Run DB migrations
  run: flyway migrate -url=jdbc:mysql://$DB_HOST/$DB_NAME -user=$DB_USER -password=$DB_PASS

18. Scenario:

How do you manage different DB configurations for dev, staging, and prod?

Answer:
I’d externalize configuration values using environment variables or .tfvars files in Terraform.
For example:

terraform apply -var-file="dev.tfvars"


Each environment gets its own DB instance size, credentials, and subnet settings.

19. Scenario:

You have to create RDS snapshots every night at 1 AM. How do you achieve this?

Answer:
I’d use an AWS Lambda function triggered by an EventBridge rule to call create-db-snapshot.
Alternatively, I could enable RDS automated backups directly via Terraform.

20. Scenario:

What’s your strategy for database rollback during failed deployment?

Answer:
Before migration, I’d take a backup or snapshot. During deployment, I’d use transactional migrations with rollback scripts. If failure occurs, I can restore the backup or use Flyway’s undo scripts.

21. Scenario:

How do you ensure your DB password doesn’t expire in a running pod?

Answer:
I’d rotate credentials using AWS Secrets Manager rotation policies and update the secret mounted in Kubernetes.
To ensure the pod picks up the new secret, I’d use annotations to trigger redeployments on secret changes.

22. Scenario:

How can you test database connection health automatically in CI/CD?

Answer:
Add a step in the pipeline that runs:

mysqladmin ping -h $DB_HOST -u $DB_USER -p$DB_PASS


If the connection fails, the pipeline should exit with non-zero status to prevent deployment.

23. Scenario:

How do you manage database schema changes across microservices?

Answer:
Each microservice should own its schema. I’d use separate DB users and migration pipelines for each service.
This isolates impact and ensures schema version control per microservice repository.

24. Scenario:

You must ensure encryption of data in RDS. How?

Answer:
At rest: Enable KMS encryption in RDS using storage_encrypted = true.
In transit: Enforce SSL connections and configure application DB drivers to use TLS.

25. Scenario:

How would you handle connection pooling in Kubernetes?

Answer:
I’d use a connection pooler like pgBouncer or ProxySQL as a sidecar or separate deployment.
It maintains persistent DB connections, reducing overhead and improving performance under load.

26. Scenario:

Database patching is pending for compliance. How would you plan it?

Answer:
I’d create a maintenance window and take a snapshot. Apply patches in staging first, validate, then schedule downtime or rolling patch in production during off-peak hours.

27. Scenario:

How do you ensure that Terraform doesn’t recreate an RDS instance during minor config changes?

Answer:
I’d use the lifecycle block:

lifecycle {
  ignore_changes = [parameter_group_name, backup_window]
}


to prevent unnecessary recreation.

28. Scenario:

You need to audit who accessed the database. What’s your approach?

Answer:
I’d enable RDS CloudTrail integration for connection events and enable general logs and audit plugins in MySQL. Logs would be exported to CloudWatch or S3 for analysis.

29. Scenario:

Database migration failed mid-way, leaving schema partially applied. How to handle it?

Answer:
I’d first restore from backup or manually revert using undo scripts. Then, rerun migrations with a flag like flyway repair to fix version mismatches before retrying.

30. Scenario:

How do you make database access available only within the VPC?

Answer:
I’d ensure RDS is deployed with publicly_accessible = false, and security groups allow inbound connections only from application subnets or node groups within the same VPC.

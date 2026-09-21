# postgres-backup

Helm chart that schedules a CronJob dumping a PostgreSQL database to an AWS S3 bucket.

Postgres twin of [mysql-backup](https://github.com/softonic/mysql-backup-chart). It runs the
[softonic/postgres-backup](https://github.com/softonic/postgres-backup) image, whose bundled
`pg_dump` is version 15 (so it supports PostgreSQL 14 and 15 servers).

The point is redundancy outside Google: Cloud SQL's own automated backups live in GCP, so a dump
pushed to S3 is what survives losing the GCP project.

## Usage

```yaml
postgres-backup:
  enabled: true
  postgres:
    # Read the Cloud SQL private IP from the Crossplane connection secret instead of
    # hardcoding it, so a recreated instance does not silently break the backup.
    hostFrom:
      secretKeyRef:
        name: sonarqube-db-connection
        key: privateIP
    database: sonarDB
    secret:
      name: sonarqube-backup-credentials
      keys:
        user: username
        password: password
  s3:
    bucket: pgdump-softonic-infrastructure
    filePrefix: sonarqube/sonarqube
    secret:
      name: aws-credentials
      keys:
        accessKeyId: accessKeyId
        secretAccessKey: secretAccessKey
  backup:
    schedule: "0 1 * * *"
```

Uploads `s3://<bucket>/<filePrefix>-<epoch>.sql.gz`.

## Values

| Key | Default | Description |
|---|---|---|
| `postgres.host` | `null` | Database host. Mutually exclusive with `postgres.hostFrom`. |
| `postgres.hostFrom.secretKeyRef.name` | `null` | Secret to read the host from. |
| `postgres.hostFrom.secretKeyRef.key` | `privateIP` | Key within that secret. |
| `postgres.port` | `5432` | Database port. |
| `postgres.database` | `null` | **Required.** Database to dump. |
| `postgres.secret.name` | `null` | **Required.** Secret holding the database user and password. |
| `postgres.secret.keys.user` | `username` | Key holding the user. |
| `postgres.secret.keys.password` | `password` | Key holding the password. |
| `s3.bucket` | `null` | **Required.** Destination bucket. |
| `s3.filePrefix` | `null` | **Required.** Object key prefix. |
| `s3.secret.name` | `aws-credentials` | Secret holding the AWS credentials. |
| `s3.endpointUrl` | `null` | Override the S3 endpoint (MinIO and friends). |
| `backup.schedule` | `0 1 * * *` | Cron schedule. |
| `backup.ttlSecondsAfterFinished` | unset | Seconds before completed jobs are deleted. |
| `backup.image.repository` / `.tag` | `softonic/postgres-backup` / `0.1.0` | Image. |
| `backup.resources` | 1 CPU / 1Gi limits | Container resources. |
| `serviceAccount.create` | `true` | Create a ServiceAccount named after the release. |

`backup.podLabels`, `backup.annotations`, `backup.nodeSelector`, `backup.tolerations` and
`backup.affinity` behave as usual.

## Verifying a backup

```bash
kubectl create job -n <ns> --from=cronjob/<release>-postgres-backup <release>-postgres-backup-manual
kubectl logs -n <ns> job/<release>-postgres-backup-manual
```

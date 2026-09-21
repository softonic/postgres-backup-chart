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
| `serviceAccount.create` | `false` | Create a ServiceAccount named after the release. Off by default: the pod never calls the Kubernetes API. |

`backup.podLabels`, `backup.annotations`, `backup.nodeSelector`, `backup.tolerations` and
`backup.affinity` behave as usual.

## Required: grant the backup user read access

Creating the database user is not enough. On Cloud SQL PostgreSQL a new user gets
`cloudsqlsuperuser`, which is a *sibling* of the application's user, not an ancestor, so it
inherits nothing and `pg_dump` stops on the first table:

```
pg_dump: error: query failed: ERROR:  permission denied for table active_rule_parameters
```

(This is the opposite of Cloud SQL MySQL, where API-created users get broad privileges
automatically — which is why `mysql-backup` never needed this step.)

Grant it once per instance, as a role that owns the data:

```sql
GRANT pg_read_all_data TO "<backup user>";
```

`pg_read_all_data` is a predefined role in PostgreSQL 14+. It covers every table in every
schema, present and future, and grants no write access.

Not automated by this chart on purpose: the `sql.gcp.upbound.io` Crossplane provider has no
grant resource, so doing it declaratively would mean a privileged Job holding admin
credentials in every namespace, to cover an instance recreation that would require redoing
users by hand anyway.

Handy one-liner, using this chart's own image (it ships `psql`):

```bash
kubectl run pg-grant --rm -i --restart=Never -n <ns> \
  --image=softonic/postgres-backup:0.1.0 \
  --overrides='{"spec":{"containers":[{"name":"pg-grant","image":"softonic/postgres-backup:0.1.0",
    "command":["sh","-c","psql -h $PGHOST -U $PGUSER -d <database> -v ON_ERROR_STOP=1 -c '\''GRANT pg_read_all_data TO \"<backup user>\";'\''"],
    "env":[{"name":"PGHOST","valueFrom":{"secretKeyRef":{"name":"<connection secret>","key":"privateIP"}}},
           {"name":"PGUSER","value":"<owner user>"},
           {"name":"PGPASSWORD","valueFrom":{"secretKeyRef":{"name":"<owner secret>","key":"password"}}}]}]}}'
```

## Verifying a backup

```bash
kubectl create job -n <ns> --from=cronjob/<release>-postgres-backup <release>-postgres-backup-manual
kubectl logs -n <ns> job/<release>-postgres-backup-manual
```

A run that works looks like this:

```
[11:43:02] Started pg_dump of 'sonarDB' on 10.221.112.134:5432
[11:43:31] Compressing dump (1283293857 bytes)
[11:45:17] Started s3 upload to s3://.../sonarqube-1789991117.sql.gz
[11:45:34] Done
```

Note the pod uses `restartPolicy: OnFailure`, so a failing dump retries quietly rather than
stopping. Check the log, not just the job's existence.

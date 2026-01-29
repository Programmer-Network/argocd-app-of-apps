# Vault Raft Backup

This CronJob takes Vault Raft snapshots every 6 hours and uploads them to R2.

## Prerequisites

### 1. Create Vault Backup Policy

Create a policy in Vault that allows Raft snapshots:

```bash
vault policy write vault-backup - <<EOF
path "sys/storage/raft/snapshot" {
  capabilities = ["read"]
}
EOF
```

### 2. Create a Periodic Token

Create a periodic token for the backup job:

```bash
vault token create \
  -policy=vault-backup \
  -period=168h \
  -display-name="vault-raft-backup" \
  -orphan
```

Save the token output.

### 3. Create the Kubernetes Secret

Create the secret with the Vault token:

```bash
kubectl create secret generic vault-backup-token \
  -n vault \
  --from-literal=token=<YOUR_TOKEN>
```

### 4. Configure Vault Kubernetes Auth for R2 Secret

Add the vault-backup role to Vault's Kubernetes auth:

```bash
vault write auth/kubernetes/role/vault-backup-role \
  bound_service_account_names=vault-backup \
  bound_service_account_namespaces=vault \
  policies=default \
  ttl=1h
```

Update the Vault policy for this role to access R2 credentials:

```bash
vault policy write vault-backup-r2 - <<EOF
path "secret/data/longhorn-r2" {
  capabilities = ["read"]
}
EOF

vault write auth/kubernetes/role/vault-backup-role \
  bound_service_account_names=vault-backup \
  bound_service_account_namespaces=vault \
  policies=vault-backup-r2 \
  ttl=1h
```

## Backup Location

Backups are stored in: `s3://database-backups/vault/`

## Retention

The job keeps the last 10 snapshots (approximately 2.5 days of backups).

## Restore

To restore from a snapshot:

```bash
# Download the snapshot
aws s3 cp s3://database-backups/vault/vault-raft-YYYYMMDD-HHMMSS.snap /tmp/restore.snap \
  --endpoint-url https://c12f98215fd46083c0a2afb795d825a2.r2.cloudflarestorage.com

# Restore (requires root token or operator permissions)
vault operator raft snapshot restore /tmp/restore.snap
```

# RGW — RADOS Gateway (Object Storage)

RGW (`radosgw`) is the Ceph object storage gateway. It presents an S3-compatible and Swift-compatible HTTP API backed by RADOS. `cephadm` manages `radosgw` containers — each daemon is configured via the Monitor config database rather than `ceph.conf`.

**Prerequisites:** Cluster at `HEALTH_OK`. Hosts intended for RGW must be added to cephadm and labeled. Port 80 (or 8080) must be open on the public network from client hosts to RGW hosts.

---

## Label RGW Hosts

```bash
ceph orch host label add ceph01 rgw
ceph orch host label add ceph02 rgw
```

---

## Option A: Simple Single-Site Deployment (No Realm)

For a self-contained object store with no multisite replication, deploy without specifying a realm or zone. RGW binds to port 80 by default.

```bash
ceph orch apply rgw default --placement="label:rgw count-per-host:1"
```

This creates a service named `rgw.default` with one daemon per labeled host.

Watch daemons come up:

```bash
ceph orch ps --daemon-type rgw
```

```
NAME              HOST    PORTS  STATUS         REFRESHED  AGE  MEM USED  MEM LIMIT  VERSION  IMAGE ID      CONTAINER ID
rgw.default.0     ceph01  *:80   running (1m)   10s ago    1m    342.7M     -         18.2.7   <id>          <id>
rgw.default.1     ceph02  *:80   running (1m)   10s ago    1m    341.2M     -         18.2.7   <id>          <id>
```

---

## Option B: Deploy with Service Spec (Recommended)

A service spec gives you full control over placement, port, and frontend settings. This is the recommended method for any cluster intended to serve production traffic.

Create `rgw.yaml`:

```yaml
service_type: rgw
service_id: default
placement:
  label: rgw
  count_per_host: 1
spec:
  rgw_frontend_type: beast
  rgw_frontend_port: 8080
```

Apply the spec:

```bash
ceph orch apply -i rgw.yaml
```

To run two daemons per host on consecutive ports (common for NVMe-backed gateways):

```yaml
service_type: rgw
service_id: default
placement:
  label: rgw
  count_per_host: 2
spec:
  rgw_frontend_type: beast
  rgw_frontend_port: 8080
```

When `count_per_host: 2` is used, the second daemon binds to `rgw_frontend_port + 1` (i.e., 8081).

---

## Realm / Zonegroup / Zone Setup (Single-Site)

For a production single-site deployment, create a realm, zonegroup, and zone before deploying RGW daemons. This structure is required for future multisite expansion and enables proper period tracking.

### Create the Realm

```bash
radosgw-admin realm create --rgw-realm=corp --default
```

### Create the Zonegroup

```bash
radosgw-admin zonegroup create \
  --rgw-zonegroup=us \
  --master \
  --default
```

### Create the Zone

```bash
radosgw-admin zone create \
  --rgw-zonegroup=us \
  --rgw-zone=us-east-1 \
  --master \
  --default
```

### Commit the Period

```bash
radosgw-admin period update --rgw-realm=corp --commit
```

Expected output:

```json
{
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "epoch": 1,
    "predecessor_uuid": "...",
    "sync_status": [],
    "period_map": {
        "id": "...",
        "zonegroups": [...],
        ...
    },
    ...
}
```

### Deploy RGW into the Realm/Zone

Create `rgw-realm.yaml`:

```yaml
service_type: rgw
service_id: corp-us-east-1
placement:
  label: rgw
  count_per_host: 1
spec:
  rgw_realm: corp
  rgw_zonegroup: us
  rgw_zone: us-east-1
  rgw_frontend_type: beast
  rgw_frontend_port: 8080
```

Apply the spec:

```bash
ceph orch apply -i rgw-realm.yaml
```

---

## Create an S3 User

RGW users are independent of Ceph cluster users. Create an S3 user with `radosgw-admin`.

```bash
radosgw-admin user create \
  --uid=testuser \
  --display-name="Test User" \
  --email=testuser@example.com
```

Expected output (truncated):

```json
{
    "user_id": "testuser",
    "display_name": "Test User",
    "email": "testuser@example.com",
    "suspended": 0,
    "max_buckets": 1000,
    "keys": [
        {
            "user": "testuser",
            "access_key": "ABCDEF1234567890ABCD",
            "secret_key": "aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789abcd"
        }
    ],
    ...
}
```

Record the `access_key` and `secret_key` — they are not retrievable after this point without re-running `radosgw-admin user info`.

Retrieve user info later:

```bash
radosgw-admin user info --uid=testuser
```

---

## Test S3 Access with the AWS CLI

Install the AWS CLI on a client host:

```bash
pip install awscli
```

Configure a profile pointing to the RGW endpoint:

```bash
aws configure --profile ceph
# AWS Access Key ID: ABCDEF1234567890ABCD
# AWS Secret Access Key: aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789abcd
# Default region name: us-east-1
# Default output format: json
```

Test with the endpoint URL (adjust host and port to match your deployment):

```bash
# List buckets
aws s3 ls --endpoint-url http://ceph01:8080 --profile ceph

# Create a bucket
aws s3 mb s3://test-bucket --endpoint-url http://ceph01:8080 --profile ceph

# Upload a file
echo "hello ceph" > /tmp/test.txt
aws s3 cp /tmp/test.txt s3://test-bucket/ --endpoint-url http://ceph01:8080 --profile ceph

# List objects in bucket
aws s3 ls s3://test-bucket --endpoint-url http://ceph01:8080 --profile ceph

# Download the file back
aws s3 cp s3://test-bucket/test.txt /tmp/test-downloaded.txt \
  --endpoint-url http://ceph01:8080 --profile ceph

cat /tmp/test-downloaded.txt
```

Expected output from `aws s3 ls s3://test-bucket`:

```
2025-03-01 10:00:00         11 test.txt
```

---

## TLS

RGW supports three TLS certificate management modes, all handled through cephadm's Certificate Manager.

### Option 1: cephadm-signed (Default)

If `ssl: true` is set without a certificate, cephadm generates and signs a certificate automatically.

```yaml
service_type: rgw
service_id: default
placement:
  label: rgw
  count_per_host: 1
spec:
  ssl: true
  certificate_source: cephadm-signed
  rgw_frontend_port: 8443
```

### Option 2: Inline Certificate

Embed an existing certificate directly in the spec:

```yaml
service_type: rgw
service_id: default
placement:
  label: rgw
  count_per_host: 1
spec:
  ssl: true
  certificate_source: inline
  rgw_frontend_port: 8443
  ssl_cert: |
    -----BEGIN CERTIFICATE-----
    (PEM certificate contents here)
    -----END CERTIFICATE-----
  ssl_key: |
    -----BEGIN PRIVATE KEY-----
    (PEM private key contents here)
    -----END PRIVATE KEY-----
```

### Option 3: Reference a Registered Certificate

Register the certificate with certmgr first, then reference it:

```bash
ceph orch certmgr cert set \
  --cert-name rgw_ssl_cert \
  --service-name rgw.default \
  -i /path/to/server_cert.pem

ceph orch certmgr key set \
  --key-name rgw_ssl_key \
  --service-name rgw.default \
  -i /path/to/server_key.pem
```

```yaml
service_type: rgw
service_id: default
placement:
  label: rgw
  count_per_host: 1
spec:
  ssl: true
  certificate_source: reference
  rgw_frontend_port: 8443
```

Apply:

```bash
ceph orch apply -i rgw.yaml
```

Test with HTTPS (use `--no-verify-ssl` for self-signed certificates during testing):

```bash
aws s3 ls --endpoint-url https://ceph01:8443 --profile ceph --no-verify-ssl
```

---

## Bucket Quotas

Quotas can be set at the user level (applies to all buckets owned by the user) or at the individual bucket level.

### User-Level Quota

```bash
# Set a 100 GiB user quota and maximum 1000 buckets
radosgw-admin quota set \
  --uid=testuser \
  --quota-scope=user \
  --max-size=107374182400 \
  --max-objects=10000000

# Enable the quota
radosgw-admin quota enable --uid=testuser --quota-scope=user

# Verify
radosgw-admin user info --uid=testuser | python3 -m json.tool | grep -A5 user_quota
```

### Bucket-Level Quota

```bash
# Set a 10 GiB quota on a specific bucket
radosgw-admin quota set \
  --uid=testuser \
  --bucket=test-bucket \
  --quota-scope=bucket \
  --max-size=10737418240 \
  --max-objects=1000000

# Enable bucket quota
radosgw-admin quota enable --uid=testuser --bucket=test-bucket --quota-scope=bucket
```

### Global Bucket Quota (Default for All New Buckets)

```bash
radosgw-admin global quota set \
  --quota-scope=bucket \
  --max-size=107374182400

radosgw-admin global quota enable --quota-scope=bucket
```

---

## Multisite Overview

Multisite RGW replicates object data and metadata across two or more Ceph clusters in different physical locations. It is the basis for active/active object storage with geographic redundancy.

The architecture uses:
- **Realm** — the top-level namespace, shared across all sites
- **Zonegroup** — a geographic region; contains one or more zones
- **Zone** — a single Ceph cluster endpoint; one zone is master per zonegroup
- **Period** — a committed snapshot of the realm configuration

Key operations in a multisite setup:
1. Create the realm and master zone on the primary cluster
2. Pull the realm and create a secondary zone on each secondary cluster
3. Deploy RGW daemons bound to each zone
4. Synchronization starts automatically; monitor with `radosgw-admin sync status`

For the full multisite deployment procedure, including zone credentials, zone endpoint registration, and sync status monitoring, see the backup-restore guide (operations section).

---

## User Management Reference

```bash
# List all users
radosgw-admin user list

# Suspend a user (blocks all access without deleting data)
radosgw-admin user suspend --uid=testuser

# Re-enable a suspended user
radosgw-admin user enable --uid=testuser

# Delete a user and all their data
radosgw-admin user rm --uid=testuser --purge-data

# Modify user display name
radosgw-admin user modify --uid=testuser --display-name="Test User Updated"

# Show bucket stats for a user
radosgw-admin bucket stats --uid=testuser
```

---

## Verification Commands

```bash
# Check RGW daemons are running
ceph orch ps --daemon-type rgw

# Check the service spec
ceph orch ls --service-type rgw

# Check RGW pools were created
ceph osd lspools | grep -E "rgw|zone"

# Verify realm/zone configuration
radosgw-admin realm list
radosgw-admin zonegroup list
radosgw-admin zone list

# Check sync status (multisite only)
radosgw-admin sync status

# Show all RGW-related config keys
ceph config get client.rgw

# Test endpoint health
curl -s http://ceph01:8080/
```

Expected output from `curl http://ceph01:8080/`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
  <Owner>
    <ID>anonymous</ID>
    <DisplayName></DisplayName>
  </Owner>
  <Buckets></Buckets>
</ListAllMyBucketsResult>
```

An S3 "anonymous" bucket list response confirms RGW is accepting connections.

---

<!-- Sources:
  https://github.com/ceph/ceph/blob/main/doc/cephadm/services/rgw.rst
  https://github.com/ceph/ceph/blob/main/doc/radosgw/admin.rst
  https://github.com/ceph/ceph/blob/main/doc/radosgw/multisite.rst
  https://github.com/ceph/ceph/blob/main/doc/radosgw/s3.rst
-->

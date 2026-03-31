# Certificate Management

Ceph exposes TLS surface area in three distinct places: the Dashboard web UI (served by ceph-mgr), the RADOS Gateway (HTTP/S object storage endpoint), and inter-daemon messenger traffic (msgr2). Each has its own configuration path. This guide covers all three plus rotation and expiry monitoring.

---

## Dashboard TLS

The Dashboard is hosted by the active `ceph-mgr` daemon and defaults to HTTPS on port 8443. All HTTP connections are secured with SSL/TLS by default. The certificate is stored in the monitor configuration database and pushed to every mgr instance.

> **Note:** The Dashboard supports **RSA private keys only**. ECDSA/EC keys will cause the module to fail with `MGR_MODULE_ERROR: Module 'dashboard' has failed: key type unsupported`. Always generate or request RSA certificates.

### Self-Signed (Quick Start)

Generate and install a self-signed certificate with a single command:

```bash
ceph dashboard create-self-signed-cert
```

Restart the manager to pick up the new certificate:

```bash
ceph mgr module disable dashboard
ceph mgr module enable dashboard
```

Verify the dashboard is reachable and confirm the certificate in use:

```bash
ceph mgr services
# Look for the "dashboard" key — copy the URL
openssl s_client -connect <mgr-host>:8443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

Browsers will warn on self-signed certificates. This is acceptable for internal use but must not be used in production-facing environments.

### Custom Certificate

Generate a 10-year RSA key pair (replace the subject with your organisation's values):

```bash
openssl req -new -nodes -x509 \
  -subj "/O=MyOrg/CN=ceph-dashboard.example.com" \
  -days 3650 \
  -keyout dashboard.key \
  -out dashboard.crt.csr \
  -extensions v3_ca
```

If you need a CSR to submit to an internal CA instead:

```bash
openssl req -new -nodes \
  -subj "/O=MyOrg/CN=ceph-dashboard.example.com" \
  -keyout dashboard.key \
  -out dashboard.csr
# Submit dashboard.csr to your CA, receive back dashboard.crt
```

Install the certificate and key cluster-wide (applies to all mgr instances):

```bash
ceph dashboard set-ssl-certificate -i dashboard.crt
ceph dashboard set-ssl-certificate-key -i dashboard.key
```

To install per-instance certificates (where `$name` is the ceph-mgr instance name, typically the hostname):

```bash
ceph dashboard set-ssl-certificate $name -i dashboard.crt
ceph dashboard set-ssl-certificate-key $name -i dashboard.key
```

Restart the manager to apply:

```bash
ceph mgr module disable dashboard
ceph mgr module enable dashboard
```

Confirm the new certificate is live:

```bash
openssl s_client -connect <mgr-host>:8443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

**Disable TLS** (behind a TLS-terminating reverse proxy only — credentials will be sent unencrypted):

```bash
ceph config set mgr mgr/dashboard/ssl false
# Default port shifts to 8080 when SSL is disabled
```

---

## RGW TLS

RGW HTTPS is managed through the cephadm Certificate Manager (`certmgr`). There are three certificate sources: `inline` (embed cert/key in the spec), `reference` (register with certmgr first), and `cephadm-signed` (cephadm auto-generates the certificate). The older `rgw_frontend_ssl_certificate` field still works but is deprecated — use `ssl_cert` / `ssl_key` instead.

### Via Service Spec

**Option 1 — cephadm-signed (automatic, no external cert required)**

```yaml
# rgw-tls-auto.yaml
service_type: rgw
service_id: myrgw
placement:
  label: rgw
  count_per_host: 1
spec:
  ssl: true
  certificate_source: cephadm-signed
  rgw_frontend_port: 443
```

```bash
ceph orch apply -i rgw-tls-auto.yaml
```

**Option 2 — inline certificate (embed PEM directly in the spec)**

```yaml
# rgw-tls-inline.yaml
service_type: rgw
service_id: myrgw
placement:
  label: rgw
  count_per_host: 1
spec:
  ssl: true
  certificate_source: inline
  rgw_frontend_port: 443
  ssl_cert: |
    -----BEGIN CERTIFICATE-----
    <base64-encoded certificate>
    -----END CERTIFICATE-----
  ssl_key: |
    -----BEGIN PRIVATE KEY-----
    <base64-encoded key>
    -----END PRIVATE KEY-----
```

```bash
ceph orch apply -i rgw-tls-inline.yaml
```

**Option 3 — reference (register cert with certmgr, reference by name)**

Register the certificate and key with certmgr:

```bash
ceph orch certmgr cert set \
  --cert-name rgw_ssl_cert \
  --service-name rgw.myrgw \
  -i /path/to/server_cert.pem

ceph orch certmgr key set \
  --key-name rgw_ssl_key \
  --service-name rgw.myrgw \
  -i /path/to/server_key.pem
```

Apply the spec referencing those registered credentials:

```yaml
# rgw-tls-reference.yaml
service_type: rgw
service_id: myrgw
placement:
  label: rgw
  count_per_host: 1
spec:
  ssl: true
  certificate_source: reference
  rgw_frontend_port: 443
```

```bash
ceph orch apply -i rgw-tls-reference.yaml
```

**Wildcard SANs** (virtual-hosted bucket access, e.g. `*.s3.example.com`):

```yaml
# rgw-tls-wildcard.yaml
service_type: rgw
service_id: myrgw
placement:
  label: rgw
  count_per_host: 1
spec:
  ssl: true
  certificate_source: cephadm-signed
  rgw_frontend_port: 443
  wildcard_enabled: true
  zonegroup_hostnames:
    - s3.example.com
```

```bash
ceph orch apply -i rgw-tls-wildcard.yaml
```

Verify RGW is serving HTTPS:

```bash
ceph orch ps --daemon-type rgw
openssl s_client -connect <rgw-host>:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

### Via Ingress

The ingress service (haproxy + keepalived) terminates TLS at a virtual IP. It supports the same three certificate sources as the RGW service spec.

If the backend RGW service already has SSL enabled, haproxy uses `ssl verify none` on the backend side (because backends are addressed by IP, not FQDN).

**Ingress with cephadm-signed certificate:**

```yaml
# ingress-tls.yaml
service_type: ingress
service_id: rgw.myrgw
placement:
  hosts:
    - host1
    - host2
    - host3
spec:
  backend_service: rgw.myrgw
  virtual_ip: 10.0.0.100/24
  frontend_port: 443
  monitor_port: 1967
  ssl: true
  certificate_source: cephadm-signed
```

```bash
ceph orch apply -i ingress-tls.yaml
```

**Ingress with inline certificate:**

```yaml
# ingress-tls-inline.yaml
service_type: ingress
service_id: rgw.myrgw
placement:
  hosts:
    - host1
    - host2
    - host3
spec:
  backend_service: rgw.myrgw
  virtual_ip: 10.0.0.100/24
  frontend_port: 443
  monitor_port: 1967
  ssl: true
  certificate_source: inline
  ssl_cert: |
    -----BEGIN CERTIFICATE-----
    <base64-encoded certificate>
    -----END CERTIFICATE-----
  ssl_key: |
    -----BEGIN PRIVATE KEY-----
    <base64-encoded key>
    -----END PRIVATE KEY-----
  # Optionally secure the haproxy stats endpoint with its own cert:
  enable_stats: true
  monitor_ssl: true
  monitor_cert_source: reuse_service_cert
```

```bash
ceph orch apply -i ingress-tls-inline.yaml
```

Confirm the virtual IP is active and the certificate is correct:

```bash
ceph orch ps --service-name ingress.rgw.myrgw
openssl s_client -connect 10.0.0.100:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

---

## Messenger v2 Encryption

Messenger v2 (msgr2) is the default on-wire protocol since Nautilus (14.2). It adds two connection modes that govern what protection is applied to all cluster-internal traffic (OSD replication, MDS, monitor heartbeats, client I/O):

| Mode | Authentication | Integrity | Confidentiality |
|------|---------------|-----------|----------------|
| `crc` | cephx mutual auth | crc32c | None — traffic is visible on the wire |
| `secure` | cephx mutual auth | AES-GCM cryptographic | Full encryption (AES-GCM, typically faster than SHA-256 on modern CPUs) |

**Enable msgr2 on monitors** (required before daemons start advertising v2 addresses):

```bash
ceph mon enable-msgr2
```

Verify monitors are advertising both v1 and v2 addresses:

```bash
ceph mon dump
# Expected output includes lines like:
# 0: [v2:10.0.0.10:3300/0,v1:10.0.0.10:6789/0] mon.ceph-mon01
```

### Connection Mode Configuration

There are six configuration knobs — three global and three monitor-specific (monitors often require stronger security than general cluster traffic):

| Config key | Scope | Default |
|------------|-------|---------|
| `ms_cluster_mode` | Daemon-to-daemon (OSD, MDS, etc.) | `crc` |
| `ms_service_mode` | Client-to-daemon | `crc` |
| `ms_client_mode` | Client outbound | `crc` |
| `ms_mon_cluster_mode` | Mon cluster traffic | `secure` |
| `ms_mon_service_mode` | Mon service traffic | `secure` |
| `ms_mon_client_mode` | Client-to-mon | `secure` |

**Set all cluster traffic to secure mode** (recommended for untrusted networks):

```bash
ceph config set global ms_cluster_mode secure
ceph config set global ms_service_mode secure
ceph config set global ms_client_mode secure
```

**Set monitor traffic to secure mode** (usually already the default — confirm):

```bash
ceph config set global ms_mon_cluster_mode secure
ceph config set global ms_mon_service_mode secure
ceph config set global ms_mon_client_mode secure
```

Verify the active configuration:

```bash
ceph config get global ms_cluster_mode
ceph config get global ms_service_mode
ceph config get global ms_client_mode
```

**Revert to crc mode** (higher throughput, acceptable on isolated networks):

```bash
ceph config set global ms_cluster_mode crc
ceph config set global ms_service_mode crc
ceph config set global ms_client_mode crc
```

Bind configuration — disable legacy v1 protocol once all clients support v2:

```bash
# Disable v1 (caution: legacy clients will be unable to connect)
ceph config set global ms_bind_msgr1 false

# Re-enable v1 if needed
ceph config set global ms_bind_msgr1 true
```

---

## Certificate Rotation

Rotate Dashboard and RGW certificates before expiry. Plan rotations at least 30 days before the `notAfter` date.

### Dashboard Certificate Rotation

**1. Generate a new RSA key and certificate (or submit CSR to CA):**

```bash
openssl req -new -nodes -x509 \
  -subj "/O=MyOrg/CN=ceph-dashboard.example.com" \
  -days 3650 \
  -keyout dashboard-new.key \
  -out dashboard-new.crt \
  -extensions v3_ca
```

**2. Install the new certificate and key:**

```bash
ceph dashboard set-ssl-certificate -i dashboard-new.crt
ceph dashboard set-ssl-certificate-key -i dashboard-new.key
```

**3. Restart the manager to apply:**

```bash
ceph mgr module disable dashboard
ceph mgr module enable dashboard
```

**4. Verify the new certificate is active:**

```bash
openssl s_client -connect <mgr-host>:8443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

### RGW Certificate Rotation (inline spec)

**1. Update the spec with the new certificate and key:**

Edit `rgw-tls-inline.yaml` replacing the `ssl_cert` and `ssl_key` PEM blocks with the new values.

**2. Apply the updated spec:**

```bash
ceph orch apply -i rgw-tls-inline.yaml
```

cephadm will redeploy only the daemons whose config changed. Verify:

```bash
ceph orch ps --daemon-type rgw
openssl s_client -connect <rgw-host>:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

### RGW Certificate Rotation (certmgr reference)

**1. Register the new certificate with certmgr** (overwriting the old entry):

```bash
ceph orch certmgr cert set \
  --cert-name rgw_ssl_cert \
  --service-name rgw.myrgw \
  -i /path/to/new_cert.pem

ceph orch certmgr key set \
  --key-name rgw_ssl_key \
  --service-name rgw.myrgw \
  -i /path/to/new_key.pem
```

**2. Redeploy the RGW service to pick up the new cert:**

```bash
ceph orch redeploy rgw.myrgw
```

**3. Verify:**

```bash
openssl s_client -connect <rgw-host>:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

---

## Monitor Expiry

### Manual Expiry Check

Check Dashboard certificate expiry:

```bash
openssl s_client -connect <mgr-host>:8443 </dev/null 2>/dev/null \
  | openssl x509 -noout -dates
```

Check RGW certificate expiry:

```bash
openssl s_client -connect <rgw-host>:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -dates
```

Check a certificate file directly:

```bash
openssl x509 -in /path/to/cert.pem -noout -dates
```

### Expiry Monitoring Script

Save as `/usr/local/bin/check-ceph-certs.sh` and run from cron or a monitoring system:

```bash
#!/usr/bin/env bash
# check-ceph-certs.sh — alert when Ceph TLS certificates expire within WARN_DAYS

set -euo pipefail

WARN_DAYS=${WARN_DAYS:-30}
WARN_SECS=$(( WARN_DAYS * 86400 ))
NOW=$(date +%s)
EXIT_CODE=0

check_endpoint() {
  local label="$1"
  local host="$2"
  local port="$3"

  local not_after
  not_after=$(openssl s_client -connect "${host}:${port}" </dev/null 2>/dev/null \
    | openssl x509 -noout -enddate 2>/dev/null \
    | cut -d= -f2)

  if [[ -z "$not_after" ]]; then
    echo "ERROR: could not retrieve certificate from ${label} (${host}:${port})"
    EXIT_CODE=2
    return
  fi

  local expiry_secs
  expiry_secs=$(date -d "$not_after" +%s 2>/dev/null \
    || date -j -f "%b %d %T %Y %Z" "$not_after" +%s)   # macOS fallback

  local days_left=$(( (expiry_secs - NOW) / 86400 ))

  if (( expiry_secs - NOW < WARN_SECS )); then
    echo "WARNING: ${label} certificate expires in ${days_left} days (${not_after})"
    EXIT_CODE=1
  else
    echo "OK: ${label} certificate valid for ${days_left} more days (${not_after})"
  fi
}

# Edit these to match your cluster endpoints
check_endpoint "Dashboard"  ceph-mgr01.example.com  8443
check_endpoint "RGW"        rgw01.example.com        443

exit "$EXIT_CODE"
```

Make executable and test:

```bash
chmod +x /usr/local/bin/check-ceph-certs.sh
/usr/local/bin/check-ceph-certs.sh
```

Schedule daily via cron (as root):

```bash
# /etc/cron.d/ceph-cert-check
0 6 * * * root WARN_DAYS=30 /usr/local/bin/check-ceph-certs.sh \
  | logger -t ceph-cert-check
```

Integrate with Prometheus/Alertmanager by scraping certificate expiry via the `ssl_exporter` or `blackbox_exporter` (probe type `ssl`) pointed at the same endpoints.

<!-- Sources:
  https://github.com/ceph/ceph/blob/main/doc/mgr/dashboard.rst
  https://github.com/ceph/ceph/blob/main/doc/cephadm/services/rgw.rst
  https://github.com/ceph/ceph/blob/main/doc/rados/configuration/msgr2.rst
-->

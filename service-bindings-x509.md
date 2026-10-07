# X.509 Service Binding Certificate Management

How SAP BTP X.509 service binding certificates work, how to create bindings with custom validity, and how to renew expired certificates before or after they cause production outages.

---

## Table of Contents

1. [What is X.509 in BTP Service Bindings?](#what-is-x509-in-btp-service-bindings)
2. [How PLD Configures X.509 (mta.yaml)](#how-pld-configures-x509-mtayaml)
3. [Checking Certificate Expiry](#checking-certificate-expiry)
4. [Renewing Expired Certificates](#renewing-expired-certificates)
5. [Requesting Longer Validity on Bind](#requesting-longer-validity-on-bind)
6. [App-by-App Renewal Reference](#app-by-app-renewal-reference)
7. [Incident: Oct 7 2026 Production Outage](#incident-oct-7-2026-production-outage)

---

## What is X.509 in BTP Service Bindings?

When an app is bound to a BTP service (XSUAA, connectivity, destination), BTP injects credentials into `VCAP_SERVICES`. By default these are client ID + secret. With `credential-type: x509`, BTP instead generates a short-lived X.509 client certificate.

The app uses this certificate for **mTLS authentication** against the service's `.cert.` endpoint (e.g. `https://<tenant>.authentication.cert.<region>.hana.ondemand.com/oauth/token`).

**Key properties:**
- Default validity: **90 days**
- BTP does **not** auto-rotate or send expiry warnings
- Credentials are injected at **restage time** — `cf restart` does NOT pick up new certs
- Each `cf bind-service` call generates a brand-new certificate

---

## How PLD Configures X.509 (mta.yaml)

In `mta.yaml`, each module declares X.509 on its service requires:

```yaml
modules:
  - name: pld-ui
    requires:
      - name: pld-uaa
        parameters:
          config:
            credential-type: x509
            x509:
              key-length: 2048
              validity: 90
              validity-type: DAYS
      - name: connectivity-service
        parameters:
          config:
            xsuaa:
              credential-type: x509
              x509:
                key-length: 2048
                validity: 90
                validity-type: DAYS
      - name: destination-service
        parameters:
          config:
            xsuaa:
              credential-type: x509
              x509:
                key-length: 2048
                validity: 90
                validity-type: DAYS
```

> **Note:** `validity: 90` means all certs expire 90 days after the last deploy/restage. If all apps were deployed on the same day, all certs expire on the same day.

---

## Checking Certificate Expiry

### Check a single app

```bash
cf env <app-name> | python3 -c "
import sys, json, re, subprocess, tempfile, os
from datetime import datetime, timezone

text = sys.stdin.read()
match = re.search(r'VCAP_SERVICES: (\{.*?\})\n\n', text, re.DOTALL)
if not match:
    print('Could not parse VCAP_SERVICES')
    sys.exit(0)

now = datetime.now(timezone.utc)
for svc_type, instances in json.loads(match.group(1)).items():
    for inst in instances:
        cert = inst.get('credentials', {}).get('certificate', '')
        if cert:
            cert_clean = cert.replace('\\\\n', '\n').replace('\\n', '\n')
            with tempfile.NamedTemporaryFile(mode='w', suffix='.pem', delete=False) as f:
                f.write(cert_clean); fname = f.name
            result = subprocess.run(
                ['openssl', 'x509', '-noout', '-dates', '-in', fname],
                capture_output=True, text=True
            )
            os.unlink(fname)
            expiry_line = [l for l in result.stdout.splitlines() if 'notAfter' in l]
            if expiry_line:
                exp_str = expiry_line[0].replace('notAfter=', '')
                try:
                    exp = datetime.strptime(exp_str, '%b %d %H:%M:%S %Y %Z').replace(tzinfo=timezone.utc)
                    status = 'EXPIRED' if exp < now else f'OK (expires {(exp-now).days}d)'
                except:
                    status = 'UNKNOWN'
                print(f'  [{status}] {svc_type}/{inst.get(\"name\")}: {exp_str}')
"
```

### Scan all apps at once

```bash
for app in pld-ui-green pld-model-ui-green pld-cpse-ui-green pld-custom-ui-green pld-utilities-green; do
  echo "=== $app ==="
  cf env $app | python3 -c "
import sys,json,re,subprocess,tempfile,os
from datetime import datetime,timezone
text=sys.stdin.read()
m=re.search(r'VCAP_SERVICES: (\{.*?\})\n\n',text,re.DOTALL)
if not m: print('  Could not parse'); sys.exit(0)
now=datetime.now(timezone.utc)
for t,insts in json.loads(m.group(1)).items():
  for i in insts:
    cert=i.get('credentials',{}).get('certificate','')
    if cert:
      with tempfile.NamedTemporaryFile(mode='w',suffix='.pem',delete=False) as f:
        f.write(cert.replace('\\\\n','\n').replace('\\n','\n')); fname=f.name
      r=subprocess.run(['openssl','x509','-noout','-enddate','-in',fname],capture_output=True,text=True)
      os.unlink(fname)
      exp=r.stdout.strip().replace('notAfter=','')
      try:
        d=datetime.strptime(exp,'%b %d %H:%M:%S %Y %Z').replace(tzinfo=timezone.utc)
        s='EXPIRED' if d<now else f'OK ({(d-now).days}d left)'
      except: s='?'
      print(f'  [{s}] {t}/{i.get(\"name\")}: {exp}')
"
done
```

---

## Renewing Expired Certificates

For each expired service on each app:

```bash
# 1. Unbind the service
cf unbind-service <app-name> <service-name>

# 2. Re-bind (generates a fresh certificate)
cf bind-service <app-name> <service-name>

# 3. Restage the app (NOT just restart — restage injects new VCAP_SERVICES)
cf restage <app-name>
```

### Verify the new cert was picked up

```bash
cf env <app-name> | grep -A2 "notAfter"
# Should show a date ~90 days from today
```

### Confirm no more errors in logs

```bash
cf logs <app-name> --recent | grep -E "certificate expired|fetchClientCredentialsToken"
# No output = fixed
```

---

## Requesting Longer Validity on Bind

To avoid the 90-day expiry cycle, request a longer validity at bind time:

```bash
# 1 year validity for connectivity
cf bind-service <app-name> connectivity-service \
  -c '{"xsuaa":{"credential-type":"x509","x509":{"key-length":2048,"validity":365,"validity-type":"DAYS"}}}'

# 1 year validity for destination
cf bind-service <app-name> destination-service \
  -c '{"xsuaa":{"credential-type":"x509","x509":{"key-length":2048,"validity":365,"validity-type":"DAYS"}}}'

# 1 year validity for XSUAA
cf bind-service <app-name> pld-uaa \
  -c '{"credential-type":"x509","x509":{"key-length":2048,"validity":365,"validity-type":"DAYS"}}'
```

> You can also update `mta.yaml` to set `validity: 365` globally so all future MTA deploys use 1-year certs.

---

## App-by-App Renewal Reference

### pld-ui-green (Approuter)

Services with X.509: `connectivity-service`, `destination-service`, `pld-uaa`

```bash
cf unbind-service pld-ui-green connectivity-service
cf bind-service pld-ui-green connectivity-service
cf unbind-service pld-ui-green destination-service
cf bind-service pld-ui-green destination-service
cf restage pld-ui-green
```

### pld-model-ui-green

Services with X.509: `destination-service`, `pld-uaa`

```bash
cf unbind-service pld-model-ui-green destination-service
cf bind-service pld-model-ui-green destination-service
cf restage pld-model-ui-green
```

### pld-cpse-ui-green

Services with X.509: `connectivity-service`, `destination-service`, `pld-uaa`

```bash
cf unbind-service pld-cpse-ui-green connectivity-service
cf bind-service pld-cpse-ui-green connectivity-service
cf unbind-service pld-cpse-ui-green destination-service
cf bind-service pld-cpse-ui-green destination-service
cf restage pld-cpse-ui-green
```

### pld-custom-ui-green

Services with X.509: `connectivity-service`, `destination-service`, `pld-uaa`

```bash
cf unbind-service pld-custom-ui-green connectivity-service
cf bind-service pld-custom-ui-green connectivity-service
cf unbind-service pld-custom-ui-green destination-service
cf bind-service pld-custom-ui-green destination-service
cf restage pld-custom-ui-green
```

### pld-utilities-green (and other backend apps)

Services with X.509: `pld-uaa`, `pld-external-uaa`, `pld-internal-uaa`, `connectivity-service`, `destination-service`

```bash
for svc in connectivity-service destination-service pld-uaa; do
  cf unbind-service pld-utilities-green $svc
  cf bind-service pld-utilities-green $svc
done
cf restage pld-utilities-green
```

---

## Incident: Oct 7 2026 Production Outage

**What happened:** All PLD production apps went down simultaneously at ~04:00 UTC on Oct 7, 2026.

**Root cause:** All `connectivity-service` and `destination-service` X.509 bindings expired at the same time (`notAfter=Oct 7 04:00 2026 GMT`) because they were all created during the same deployment run ~90 days earlier.

**Symptoms:**
- Approuter (`pld-ui-green`) returned HTTP 500 for all `/model-ui/*` and `/srv/*` routes
- Logs showed `fetchClientCredentialsToken` failing with "not successful after 4 attempts"
- Backend apps (`pld-utilities-green`) showed `ssl/tls alert certificate expired (SSL alert number 45)`
- UI5 apps showed `ModuleError: script load error` in browser console

**Resolution:**
1. Diagnosed using `cf env` + `openssl x509 -noout -dates` — found connectivity and destination certs expired
2. Re-bound and restaged each affected app
3. Total downtime: ~2 hours

**Prevention:**
- Set a calendar reminder ~2 weeks before next expiry
- Consider increasing validity to 365 days in `mta.yaml`
- Run the cert scan script periodically (e.g. weekly in CI) to catch expiry early

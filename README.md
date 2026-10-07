# SAP BTP Runbooks

Personal operational runbooks for SAP BTP Cloud Foundry — deployment procedures, service binding management, certificate renewal, and troubleshooting guides for the **Product Landscape Designer (PLD)** ecosystem.

---

## Runbooks

| Runbook | Description |
|---|---|
| [PLD Deployment Guide](./pld-deployment.md) | Full deployment walkthrough — MTA build, `cf push`, manifest, environment setup |
| [X.509 Service Binding & Certificate Renewal](./service-bindings-x509.md) | How service bindings work, how to create them, and how to renew expired X.509 certificates |

---

## Quick Reference

### Login to Production

```bash
cf login -a https://api.cf.<region>.hana.ondemand.com --sso
# Passcode: https://login.cf.<region>.hana.ondemand.com/passcode
# Org: <your-org-name>
# Space: Production
```

### Check All Cert Expiries

```bash
for app in pld-ui-green pld-model-ui-green pld-cpse-ui-green pld-custom-ui-green pld-utilities-green; do
  echo "=== $app ==="
  cf env $app | python3 -c "
import sys,json,re,subprocess,tempfile,os
from datetime import datetime,timezone
text=sys.stdin.read()
m=re.search(r'VCAP_SERVICES: (\{.*?\})\n\n',text,re.DOTALL)
if not m: sys.exit(0)
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
        s='EXPIRED' if d<now else 'OK'
      except: s='?'
      print(f'  [{s}] {t}/{i.get(\"name\")}: {exp}')
"
done
```

---

## Environment Info

| Environment | API Endpoint | Org |
|---|---|---|
| Production | `https://api.cf.<region>.hana.ondemand.com` | `<your-prod-org>` |
| Canary | `https://api.cf.<region>.hana.ondemand.com` | `<your-canary-org>` |

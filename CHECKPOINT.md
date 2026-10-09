# BGPALERT — Project Checkpoint

**Checkpoint date:** 2026-10-09  
**Project path:** `D:\BGPALERT`  
**Current phase:** BGPalerter core monitoring is running successfully.

---

## 1. Project Goal

Build a Docker-based BGP monitoring/NOC stack for **AS24207**.

Planned architecture:

```text
RIPE RIS Live
     |
     v
BGPalerter
     |
     +--> BGP alerts/logs
     |
     v
Promtail
     |
     v
Loki
     |
     v
Grafana
```

Primary use case:

- Monitor prefixes originated by AS24207.
- Detect BGP events such as hijack, new prefix, visibility, path, misconfiguration and RPKI-related events.
- Centralize alerts/logs in Loki.
- Visualize events in Grafana.
- Eventually use the project as a practical NOC/QoSE/BGP learning project.

---

## 2. Current Host Structure

```text
D:\BGPALERT
├── compose.yml
└── volume
    ├── config.yml
    ├── prefixes.yml
    └── logs
```

Current `compose.yml`:

```yaml
services:
  bgpalerter:
    image: nttgin/bgpalerter:latest
    volumes:
      - ./volume/config.yml:/opt/bgpalerter/config.yml
      - ./volume/prefixes.yml:/opt/bgpalerter/prefixes.yml
      - ./volume/logs:/opt/bgpalerter/logs
```

---

## 3. BGPalerter Status

Image:

```text
nttgin/bgpalerter:latest
```

Detected version:

```text
BGPalerter 2.0.1
```

Container status:

```text
bgpalerter-1   nttgin/bgpalerter:latest   "npm run serve"   Up
```

The container successfully starts using the image's default:

```text
npm run serve
```

---

## 4. AS24207 Prefix Generation

BGPalerter's generator was used to generate the monitoring prefix list for:

```text
AS24207
```

Generated file:

```text
D:\BGPALERT\volume\prefixes.yml
```

Observed result:

```text
Total prefixes detected: 95
File size: approximately 12.6 KB
675 YAML lines
```

The generated file contains IPv4 and IPv6 prefixes, for example:

```text
222.165.212.0/24
203.153.99.0/24
2404:f600::/35
2404:f600:c000::/36
2001:df2:f140::/48
```

BGPalerter successfully reports:

```text
Monitoring <prefix>
```

for the generated AS24207 prefixes.

---

## 5. RIPEstat Generator Note

During prefix generation, some RIPEstat requests timed out:

```text
TypeError: fetch failed
ConnectTimeoutError
UND_ERR_CONNECT_TIMEOUT
```

One example reported:

```text
RIPEstat prefix-overview query failed:
cannot retrieve information for 203.153.101.0/24
```

However, the generator continued and successfully detected:

```text
Total prefixes detected: 95
```

Therefore the generated `prefixes.yml` is usable, but the RIPEstat timeout should be remembered as a possible source of incomplete prefix metadata.

Do not treat this as a Babel/BGPalerter startup failure.

---

## 6. Important Debugging Discovery

The Docker image uses:

```dockerfile
ENTRYPOINT ["npm"]
CMD ["run", "serve"]
```

Therefore:

```text
docker compose run bgpalerter generate
```

does NOT directly execute BGPalerter's `generate` command. Docker effectively passes the command through the `npm` entrypoint.

BGPalerter provides the npm script:

```text
generate-prefixes
```

but `babel-node` in this image/environment produced:

```text
SyntaxError: Cannot use import statement outside a module
```

The `.babelrc` itself was valid, because the Babel CLI successfully transformed:

```text
index.js
```

from ES module syntax into CommonJS.

Successful workaround:

```text
Babel CLI compile
        |
        v
/opt/bgpalerter/dist-temp/index.js
        |
        v
Node.js directly
        |
        v
BGPalerter generate
```

The working generation command was:

```powershell
docker compose exec bgpalerter node /opt/bgpalerter/dist-temp/index.js generate -a 24207 -o /tmp/prefixes.yml
```

The compiled `src` directory was also required because `index.js` imports modules from `src`.

---

## 7. Persistence Verification

The host file:

```text
D:\BGPALERT\volume\prefixes.yml
```

is successfully mounted into the container:

```text
/opt/bgpalerter/prefixes.yml
```

Verified with:

```powershell
docker compose exec bgpalerter ls -lh /opt/bgpalerter/prefixes.yml
```

Result was approximately:

```text
12.6K
```

Therefore the prefix configuration is persistent outside the container.

---

## 8. BGPalerter Logs

Inside the container:

```text
/opt/bgpalerter/logs
```

Current files include:

```text
.error
.reports
error-2026-10-09.log
error.log -> error-2026-10-09.log
reports-2026-10-09.log
reports.log -> reports-2026-10-09.log
```

At checkpoint time:

```text
reports-2026-10-09.log = 0 bytes
```

This is not necessarily an error. It means no reportable BGP event had been written yet.

`error.log` contained informational startup events such as:

```text
ris connector connected
Performing expiration check on VRPs
Performing TA deletion check
Performing TA expiration check
Subscribed to monitored resources
Subscribed to beacons
```

Important:

`error.log` is not necessarily an actual error-only stream; it contains informational runtime messages too.

---

## 9. RIPE RIS Live Status

BGPalerter successfully connected to RIPE RIS Live.

Observed:

```text
ris connector connected
```

and:

```text
Subscribed to monitored resources
Subscribed to beacons
```

This confirms the core monitoring connection is operational.

---

## 10. Current Architecture Status

### Completed

- [x] Docker Compose project initialized
- [x] BGPalerter container running
- [x] BGPalerter config generated
- [x] AS24207 prefix list generated
- [x] `prefixes.yml` copied to host
- [x] `config.yml` persisted on host
- [x] BGPalerter logs persisted on host
- [x] Prefix list successfully mounted into container
- [x] BGPalerter monitoring AS24207 prefixes
- [x] RIPE RIS Live connection established

### Not yet completed

- [ ] Promtail
- [ ] Loki
- [ ] Grafana
- [ ] Loki labels/schema
- [ ] Grafana dashboard
- [ ] BGP event classification dashboard
- [ ] Alert notifications
- [ ] Git repository initialization
- [ ] GitHub repository
- [ ] README documentation
- [ ] `.gitignore`
- [ ] Versioned configuration strategy
- [ ] Automated prefix regeneration
- [ ] Testing/recovery procedure

---

## 11. Planned Next Phase

Build the logging pipeline one component at a time:

```text
BGPalerter
    |
    | reports.log / error.log
    v
Promtail
    |
    v
Loki
    |
    v
Grafana
```

Recommended order:

1. Add Loki to Compose.
2. Verify Loki health.
3. Add Promtail.
4. Configure Promtail to read:
   `./volume/logs/*.log`
5. Send logs to Loki.
6. Verify logs through Loki queries.
7. Add Grafana.
8. Connect Grafana to Loki.
9. Build BGPalerter dashboard.

Do not add everything at once; isolate each component so failures are easy to troubleshoot.

---

## 12. Git/GitHub Checkpoint

Recommended repository name:

```text
bgpalert
```

Recommended initial repository structure:

```text
bgpalert/
├── compose.yml
├── README.md
├── CHECKPOINT.md
├── .gitignore
└── volume/
    ├── config.yml
    ├── prefixes.yml
    └── logs/
```

Important security rule:

Do NOT commit runtime logs or generated secrets/tokens.

Before the first Git commit, review:

```text
volume/logs/
```

and any future credentials/configuration containing secrets.

The current project does not contain a GitHub repository yet.

---

## 13. Recovery Starting Point

If continuing this project in a new session, start by checking:

```powershell
cd D:\BGPALERT
docker compose ps
docker compose logs --tail=50 bgpalerter
```

Expected state:

```text
bgpalerter-1   Up
```

Then verify:

```powershell
docker compose exec bgpalerter ls -lh /opt/bgpalerter/prefixes.yml
```

Expected:

```text
approximately 12.6K
```

Then continue from:

```text
Loki -> Promtail -> Grafana
```

---

## 14. Learning Notes

This checkpoint intentionally records the reasoning behind the setup rather than only the commands.

Key concepts learned:

- Docker `ENTRYPOINT` and `CMD` interaction.
- Why `docker compose run bgpalerter generate` became an npm command.
- Difference between `babel-node` and Babel CLI compilation.
- CommonJS vs ES module execution.
- Node.js module resolution and why `/tmp/dist` could not find `yargs`.
- Why compiling under `/opt/bgpalerter/dist-temp` allowed Node to resolve `/opt/bgpalerter/node_modules`.
- Bind mounts for persistent Docker configuration.
- BGPalerter prefix monitoring based on AS-originated prefixes.
- RIPE RIS Live as the BGP update source.
- Difference between BGPalerter runtime information and actual BGP alert reports.

---

**Checkpoint status: CORE BGP MONITORING OPERATIONAL**

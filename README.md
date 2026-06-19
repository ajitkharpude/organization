# Organization Service — Graylog Logging Setup

This document explains how **organization-service** (the `evoke` Spring Boot app) sends logs to Graylog, and how to configure it for **your own Graylog server**.

> ✅ Good news: this codebase already ships with a complete Graylog integration (Log4j2 + GELF/UDP). You don't need to write new logging code — you only need to **point it at your Graylog server** and make sure the right profile/config is active.

---

## 1. How logging is wired in this project

| Layer | File | Purpose |
|---|---|---|
| Dependency | `pom.xml` | Pulls in `biz.paluch.logging:logstash-gelf` (the GELF appender for Log4j2) and `spring-boot-starter-log4j2` (replaces default Logback) |
| Default config | `src/main/resources/application.yml` | Sets `logging.config: classpath:log4j2-local.xml` — used for local dev (no Graylog forwarding) |
| Server config | `src/main/resources/application-server.yml` | Sets `logging.config: classpath:log4j2-graylog.xml` — used in staging/production (forwards to Graylog) |
| Local Log4j2 config | `src/main/resources/log4j2-local.xml` | Console + rolling file only. Has a GELF appender too, but it's intended for local/dev use |
| Production Log4j2 config | `src/main/resources/log4j2-graylog.xml` | Console (INFO+) + rolling file (DEBUG+) + **GELF → Graylog (WARN+)**, fully async |
| Request tracing | `src/main/java/com/clouds/org/infrastructure/web/LoggingFilter.java` | Adds `traceId`, `httpMethod`, `requestUri` to MDC per request, logs inbound/outbound lines, skips `/actuator/health` |
| Graylog settings (informational) | `application.yml` → `graylog:` block | `enabled`, `host`, `port`, `protocol`, `application-name` — **note:** these are plain custom properties, not auto-wired into the GELF appender. The appender reads `GRAYLOG_IP` / `GRAYLOG_PORT` env vars directly (see below). |

### What gets sent to Graylog

- Transport: **GELF over UDP**
- Minimum level forwarded: **WARN** (INFO/DEBUG only go to console/file, not Graylog)
- Every GELF message includes these extra fields:
  - `service` = `organization-service`
  - `version` = app version (e.g. `1.0.4`)
  - `environment` = value of `SPRING_PROFILES_ACTIVE` (defaults to `production`)
  - `pod` = value of `HOSTNAME` (useful in Kubernetes)
  - `namespace` = value of `K8S_NAMESPACE` (defaults to `local`)
  - `traceId`, `httpMethod`, `requestUri`, `userId`, `orgId`, `requestId` — pulled from MDC (per-request context)

---

## 2. Point it at your existing Graylog server

You said you **already have a Graylog server running** — great, you just need an **input** on it and the right **environment variables** on the app side.

### Step 1 — Create a GELF UDP input on your Graylog server

1. Log in to your Graylog web UI.
2. Go to **System → Inputs**.
3. From the "Select input" dropdown choose **GELF UDP**, click **Launch new input**.
4. Configure:
   - **Title**: `organization-service`
   - **Bind address**: `0.0.0.0`
   - **Port**: `12812` (or any port you prefer — just match it in the app config below)
5. Click **Save**. The input should show as **RUNNING**.
6. Make sure that port is reachable from the app server/container (check firewall / security group / Kubernetes NetworkPolicy rules for **UDP**, not TCP).

> The appender already in this repo (`log4j2-graylog.xml`) defaults to `udp:10.13.10.12:12812` — that's almost certainly **not** your server. You must override it (Step 2).

### Step 2 — Set environment variables on the app side

The GELF appender reads these at runtime (no code changes needed):

| Variable | Required | Default in repo | Set it to |
|---|---|---|---|
| `GRAYLOG_IP` | ✅ Yes | `10.13.10.12` | Your Graylog server's IP/hostname |
| `GRAYLOG_PORT` | ✅ Yes | `12812` | The GELF UDP input port you created |
| `SPRING_PROFILES_ACTIVE` | Recommended | _(unset → `production`)_ | `server` to activate `application-server.yml`, plus an env tag like `staging`/`production` |
| `APP_NAME` / `HOSTNAME` / `K8S_NAMESPACE` | Optional | various | Used only as descriptive GELF fields, tweak as desired |

**Example — running locally with `java -jar`:**

```bash
export SPRING_PROFILES_ACTIVE=server
export GRAYLOG_IP=192.168.1.50
export GRAYLOG_PORT=12812

java -jar target/evoke-1.0.4.jar
```

**Example — Docker:**

```bash
docker run -d \
  -e SPRING_PROFILES_ACTIVE=server \
  -e GRAYLOG_IP=192.168.1.50 \
  -e GRAYLOG_PORT=12812 \
  -p 8081:8081 \
  your-registry/organization-service:1.0.4
```

**Example — Kubernetes (env section of your Deployment):**

```yaml
env:
  - name: SPRING_PROFILES_ACTIVE
    value: "server"
  - name: GRAYLOG_IP
    value: "192.168.1.50"
  - name: GRAYLOG_PORT
    value: "12812"
  - name: K8S_NAMESPACE
    valueFrom:
      fieldRef:
        fieldPath: metadata.namespace
```

> ⚠️ **Important:** the `SPRING_PROFILES_ACTIVE=server` profile is what activates `application-server.yml`, which is what switches `logging.config` to `log4j2-graylog.xml`. Without this profile active, the app uses `log4j2-local.xml` (no real Graylog forwarding configured for your environment) by default. Combine profiles if needed, e.g. `SPRING_PROFILES_ACTIVE=server,prod`.

### Step 3 — (Optional) Hardcode values instead of env vars

If you don't want to rely on environment variables, you can directly edit the host/port in `src/main/resources/log4j2-graylog.xml` (and `log4j2-local.xml` if you also use that profile):

```xml
<GELF name="Graylog"
    host="udp:192.168.1.50"
    port="12812"
    ...
```

Environment variables are recommended over hardcoding so the same JAR/image can be promoted across dev → staging → production without rebuilding.

---

## 3. Verify it's working

1. Start the app with the env vars from Step 2.
2. Trigger a `WARN` or `ERROR` log line — easiest way is to hit an endpoint that doesn't exist, or check the app's startup logs for any warnings.
3. In Graylog, go to **Search** and query:
   ```
   service:organization-service
   ```
4. You should see entries with fields like `service`, `version`, `environment`, `pod`, `traceId`, etc.
5. If nothing shows up:
   - Confirm the GELF UDP input on Graylog shows **RUNNING** and has received messages (Graylog UI shows a message count per input).
   - Confirm `GRAYLOG_IP`/`GRAYLOG_PORT` are correct and reachable (UDP, not blocked by firewall).
   - Confirm `SPRING_PROFILES_ACTIVE` includes `server` so `log4j2-graylog.xml` is actually loaded — check the app's console output at startup; Log4j2 logs which config file it loaded.
   - Remember: only **WARN and above** go to Graylog by design. INFO-level logs won't appear there — check console/file logs for those.

---

## 4. Quick reference — all relevant files in this repo

```
organization_service/
├── pom.xml                                  # logstash-gelf + log4j2 dependencies
└── src/main/resources/
    ├── application.yml                      # default profile → log4j2-local.xml
    ├── application-server.yml               # "server" profile → log4j2-graylog.xml
    ├── log4j2-local.xml                      # dev config (console+file+GELF)
    └── log4j2-graylog.xml                    # production config (console+file+GELF, WARN+ to Graylog)
```

```
src/main/java/com/clouds/org/infrastructure/web/LoggingFilter.java   # adds traceId/method/uri to MDC per request
```

---

## 5. Summary checklist

- [ ] GELF UDP input created on your Graylog server (note the port)
- [ ] UDP port open between app and Graylog server (firewall/security group)
- [ ] `GRAYLOG_IP` and `GRAYLOG_PORT` env vars set to match your server
- [ ] `SPRING_PROFILES_ACTIVE` includes `server` so `log4j2-graylog.xml` is loaded
- [ ] App restarted after setting env vars
- [ ] Test log line visible in Graylog search (`service:organization-service`)

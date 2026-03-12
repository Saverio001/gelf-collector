# Fix History

A log of every issue encountered setting up this stack and how it was resolved.

---

## 1. Loki permission denied reading config file

**Error:**
```
failed parsing config: open /etc/loki/loki-config.yml: permission denied
```

**Cause:** The Loki image runs as UID 10001, not root. The bind-mounted config
file was owned by root and not world-readable.

**Fix:** Added a `loki-init` one-shot container that runs as root and:
- Copies `loki-config.yml` into a named Docker volume (`loki-config`)
- Sets ownership of all Loki volumes to `10001:10001`

Loki mounts the volume instead of the host file directly, and is declared
with `user: "10001:10001"`. It depends on `loki-init` completing successfully
before starting.

---

## 2. Grafana datasource pointing to localhost instead of loki

**Error:**
```
dial tcp [::1]:3100: connect: connection refused
lokiHost=localhost:3100
```

**Cause:** A datasource named `loki-1` had been created manually in the UI
with URL `http://localhost:3100`. This took precedence over the provisioned
datasource.

**Fix:** Added a `deleteDatasources` block to the provisioning file to remove
stale manually-created datasources on startup. The provisioned datasource
explicitly sets `url: http://loki:3100`.

---

## 3. Grafana container had wrong DNS alias

**Symptom:** `docker exec grafana wget http://loki:3100/ready` returned 503,
but `docker exec loki wget http://localhost:3100/ready` returned `ready`.

**Cause:** After incremental container restarts, Docker assigned the `loki`
DNS alias to the Grafana container, causing it to resolve `loki` to itself.

**Fix:** Full `docker compose down && docker compose up -d` to recreate all
containers and network aliases cleanly.

---

## 4. Vector loading wrong config file

**Symptom:** Vector kept generating demo log messages instead of listening on
configured ports.

**Cause:** The Vector image defaults to loading `/etc/vector/vector.yaml`.
Our config was mounted at `/etc/vector/vector.toml` but never loaded.

**Fix:** Added `command: ["--config", "/etc/vector/vector.toml"]` to the
Vector service in `docker-compose.yml`.

---

## 5. GELF source type does not exist in Vector 0.36

**Error:**
```
unknown variant `gelf`, expected one of `amqp`, `socket`, `syslog`, ...
```

**Cause:** Vector 0.36 removed the dedicated `gelf` source type. GELF is
now handled by the generic `socket` source with JSON decoding.

**Fix:** Replaced `type = "gelf"` with:
```toml
type = "socket"
mode = "tcp"
decoding.codec = "json"
framing.method = "newline_delimited"
```

---

## 6. VRL if/else syntax errors

**Error:**
```
unexpected syntax token: "Else"
```

**Cause:** VRL requires `else` to appear on the same line as the closing
brace of the preceding block. Multi-line `if/else if` chains written with
each clause on its own line are not valid.

**Fix:** Replaced the level-name mapping logic with a simpler approach —
pass the raw numeric level value through as a label. The syslog source
already provides human-readable severity names natively.

---

## 7. GELF TCP null-byte framing rejected by Vector

**Error:**
```
Failed deserializing frame: Error("trailing characters", line: 1, column: 97)
```

**Cause:** GELF's original wire format uses a null byte (`\0`) as the message
delimiter. Vector's socket source with JSON codec uses newline delimiting and
treats the null byte as a trailing invalid character.

**Fix:** Added `framing.method = "newline_delimited"` to the `gelf_tcp` source,
and changed the test sender to use `\n` instead of `\0` as the terminator:

```bash
printf '{"version":"1.1","host":"testhost","short_message":"Hello","level":6}\n' \
  | nc -q1 localhost 12201
```

Note: applications sending GELF must also be configured to use newline
framing rather than null-byte framing.

---

## 8. Grafana dashboard not visible after provisioning

**Cause:** The dashboard JSON referenced a hardcoded datasource UID (`"loki"`)
that did not match the UID Grafana auto-generated when provisioning the
datasource (`a8cb3cda-...`).

**Fix:** Added an explicit `uid: loki-homelab` field to the datasource
provisioning YAML, and updated all datasource references in the dashboard
JSON to use the same UID.

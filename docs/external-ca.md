# Using an External Certificate Authority

By default, the `setup` service generates its own certificate authority the first
time the stack starts and uses it to issue TLS certificates for every component.
This guide explains how to use your own (corporate or organizational) CA instead —
either on a fresh install, or by replacing the self-generated certificates after
the stack has already run.

All certificates live in the shared `certs` Docker volume. Every service reads
from that one volume, so swapping the CA in the volume re-keys the entire stack,
including the Kafka profile.

## How certificate generation works

On startup the `setup` service (see `stack-setup.yml`):

1. Creates a CA (`ca/ca.crt`, `ca/ca.key`) — **only if no CA exists in the volume**.
2. Creates per-service certificates signed by that CA — only if `certs.zip` does
   not already exist in the volume.
3. Creates the Kafka certificate, PKCS12 keystore, and TLS client config from the
   same CA — only if they are missing.

Because each step is skipped when its output already exists, you control the CA
simply by placing your own `ca/ca.crt` and `ca/ca.key` in the volume before the
instance certificates are generated.

Volume layout after setup completes:

```text
certs/
├── ca/
│   ├── ca.crt                  # CA certificate (all clients trust this)
│   ├── ca.key                  # CA private key (used to sign service certs)
│   └── ca.pem                  # ca.crt + ca.key concatenated
├── ca.zip                      # only present when the CA was self-generated
├── certs.zip                   # marker + archive for the service certs
├── instances.yml               # SANs used for the service certs
├── es01/  es02/  es03/         # Elasticsearch node certs (es01 also has es01.chain.pem)
├── ml01/  fz01/                # optional ES node certs (ml / frozen profiles)
├── kibana/                     # Kibana cert
├── fleet-server/               # Fleet/APM server cert
├── ems-server/                 # Elastic Maps Server cert
└── kafka/
    ├── kafka.crt  kafka.key    # Kafka broker cert (PEM)
    ├── kafka.keystore.p12      # broker keystore (password: KAFKA_SSL_KEYSTORE_PASSWORD)
    ├── kafka_creds             # keystore password file for the broker image
    └── client-ssl.properties   # ready-made Kafka CLI/client TLS config
```

## CA requirements

- PEM-encoded certificate and key, named exactly `ca.crt` and `ca.key`
- The key must be **unencrypted** (no passphrase) — `elasticsearch-certutil` uses
  it non-interactively to sign the service certificates
- The certificate must be a CA certificate (`basicConstraints = CA:TRUE`)
- An intermediate CA works too: provide the intermediate's cert/key as
  `ca.crt`/`ca.key`; clients inside the stack trust `ca.crt` directly. If external
  clients only trust your root, hand them the full chain

To create a dedicated CA for the stack with OpenSSL:

```bash
openssl req -x509 -newkey rsa:4096 -nodes -days 3650 \
  -subj "/CN=My Org Elastic Stack CA" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign" \
  -keyout ca.key -out ca.crt
```

## Scenario A: fresh install with an external CA

Do this **before the first `docker compose up`** (or after removing the certs
volume — see Scenario B).

```bash
# 1. Put your CA files in a local directory named "ca"
ls my-ca/
# ca.crt  ca.key

# 2. Create the setup container (this also creates the empty certs volume)
docker compose create setup

# 3. Copy the CA directory into the certs volume
docker compose cp my-ca setup:/usr/share/elasticsearch/config/certs/ca

# 4. Start the stack (add your usual -f files / --profile flags)
docker compose up -d
```

The setup service logs `Using provided external CA` and issues every service
certificate from your CA.

## Scenario B: replacing self-generated certificates after a first run

The instance certificates are only generated when they are missing, so to re-key
the stack you must clear the certs volume, seed your CA, and start again. Your
data is safe — Elasticsearch data, Kibana data, Kafka topics, etc. live in other
volumes.

```bash
# 1. Stop the stack. Include every -f file and --profile you normally run so
#    all containers using the certs volume are stopped, e.g.:
docker compose --profile kafka --profile logstash down

# 2. Remove ONLY the certs volume (do NOT use `down -v` - that deletes data volumes).
#    The volume is named <COMPOSE_PROJECT_NAME>_certs; with the default
#    COMPOSE_PROJECT_NAME=es-cluster from .env:
docker volume rm es-cluster_certs

# 3. Seed your CA (same as Scenario A)
docker compose create --force-recreate setup
docker compose cp my-ca setup:/usr/share/elasticsearch/config/certs/ca

# 4. Start the stack again with your usual flags
docker compose --profile kafka --profile logstash up -d
```

On startup the setup service regenerates all service certificates — now signed by
your CA — and every container picks them up when it starts.

## What this means per profile

Every service reads its certificate and the CA from the shared volume at container
start, so no per-service configuration changes are needed. After a CA swap,
`docker compose ... up -d` (which recreates the stopped containers) is enough.

| Profile / file | Services using the certs volume | Notes after CA replacement |
|---|---|---|
| base (`docker-compose.yml`) | `es01` `es02` `es03` `kibana` `fleet-server` | Restart is enough. Browsers must now trust your CA for `https://localhost:5601` / `:9200` |
| `--profile ml` / `--profile frozen` | `ml01`, `fz01` | Same as the ES nodes |
| `--profile monitoring` | `metricbeat01` | Reads `certs/ca/ca.crt`; restart is enough |
| `--profile filebeat` | `filebeat01` | Same |
| `--profile logstash` | `logstash01` | Same |
| `--profile kafka` | `kafka` `kafka-setup` `kafka-ui` `logstash-kafka-in` `logstash-kafka-out` | Keystore + `client-ssl.properties` are regenerated from the new CA automatically. External Kafka clients must switch their truststore to the new CA |
| `--profile agent` / air-gapped | `container-agent` (via `elastic-stack.yml`) | Re-enrolls against Fleet using the new CA on restart |
| `elastic-maps-server.yml` | `ems-server` | Restart is enough |

External clients you must update after a CA swap:

- Anything hitting Elasticsearch/Kibana/Fleet from outside Docker (scripts using
  `curl --cacert`, Beats/Agents on other hosts enrolled with the old CA)
- External Kafka producers/consumers connecting to `${DOCKER_HOST_IP}:9094` —
  point their truststore at the new `ca.crt` (`security.protocol=SSL`,
  `ssl.truststore.type=PEM`)

## Verifying

```bash
# The CA in the volume is yours
docker compose cp setup:/usr/share/elasticsearch/config/certs/ca/ca.crt /tmp/stack-ca.crt
openssl x509 -in /tmp/stack-ca.crt -noout -subject -issuer

# Service certs chain to it (run after the stack is up; the openssl here is the
# host's - the Elasticsearch image does not ship one)
docker exec es01 cat config/certs/es01/es01.crt | openssl x509 -noout -issuer
docker exec kafka cat /certs/kafka/kafka.crt | openssl x509 -noout -issuer

# Elasticsearch over TLS with your CA
curl --cacert /tmp/stack-ca.crt -u elastic:$ELASTIC_PASSWORD https://localhost:9200

# Kafka broker presents a cert signed by your CA
openssl s_client -connect localhost:9094 -CAfile /tmp/stack-ca.crt </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer
```

## Troubleshooting

- **`Using provided external CA` not shown / CA overwritten**: the CA is only
  treated as external when `ca/ca.crt` and `ca/ca.key` exist **and** `ca.zip` does
  not. Make sure you copied both files and did not copy a `ca.zip` into the volume.
- **Setup fails signing certs**: the CA key is probably encrypted. Provide an
  unencrypted key (`openssl rsa -in encrypted.key -out ca.key`).
- **Old certs still served**: the service certs are only regenerated when
  `certs.zip` is missing — remove the certs volume (Scenario B step 2), don't just
  replace `ca/` in place.
- **Kafka clients fail after the swap**: regenerate their truststore from the new
  `ca.crt`; the broker keystore itself is rebuilt automatically.

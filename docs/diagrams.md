# Dataflow and Pipeline Diagrams

Dataflow and pipeline diagrams for the base stack and each docker compose
profile. All diagrams are [Mermaid](https://mermaid.js.org/) and render directly
on GitHub.

## Base Stack (`docker compose up -d`)

The setup service generates a CA and TLS certificates into the shared `certs`
volume, then the 3-node Elasticsearch cluster, Kibana, and Fleet Server (which
also provides the APM server) come online. All connections are TLS.

```mermaid
flowchart LR
    setup["setup<br/>(cert generation)"] -. writes .-> certs[("certs volume")]
    certs -. mounted by .-> es01 & kibana & fleet

    subgraph cluster["Elasticsearch Cluster"]
        es01["es01 (9200)"] <--> es02["es02"] <--> es03["es03"]
        es01 <--> es03
    end

    kibana["Kibana (5601)"] --> es01
    fleet["Fleet Server + APM (8220/8200)"] --> es01
    agent["Elastic Agent"] --> fleet
    user(("Browser")) --> kibana
```

## Machine Learning Profile (`--profile ml`)

```mermaid
flowchart LR
    subgraph cluster["Elasticsearch Cluster"]
        es01["es01"] <--> es02["es02"] <--> es03["es03"]
        ml01["ml01<br/>(dedicated ML node,<br/>ELSER capable)"] <--> es01
    end
    kibana["Kibana ML UI"] --> es01
```

## Frozen Tier Profile (`--profile frozen`)

```mermaid
flowchart LR
    subgraph cluster["Elasticsearch Cluster"]
        es01["es01 (hot)"] <--> es02 & es03
        fz01["fz01<br/>(data_frozen)"] <--> es01
    end
    miniosetup["minio-setup"] -- creates bucket/user --> minio
    es01 -- snapshots --> minio[("MinIO S3<br/>(9000 / GUI 9001)")]
    fz01 -- searchable snapshots --> minio
```

## Monitoring Profile (`--profile monitoring`)

```mermaid
flowchart LR
    es[("Elasticsearch")] & kibana["Kibana"] & logstash["Logstash"] & docker["Docker host"] -- metrics --> mb["metricbeat01"]
    mb -- monitoring indices --> es
    kibana -- Stack Monitoring UI --> es
```

## Filebeat Profile (`--profile filebeat`)

```mermaid
flowchart LR
    files["filebeat_ingest_data/*.log"] --> fb["filebeat01"]
    containers["Docker container logs"] --> fb
    fb --> es[("Elasticsearch")]
    es --> kibana["Kibana Logs Stream"]
```

## Logstash Profile (`--profile logstash`)

```mermaid
flowchart LR
    files["logstash_ingest_data/*"] --> ls["logstash01<br/>(config/logstash.conf)"]
    ls -- "logstash-YYYY.MM.dd" --> es[("Elasticsearch")]
    es --> kibana["Kibana"]
```

## APM Profile (`--profile apm`)

```mermaid
flowchart LR
    user(("Browser")) --> webapp["webapp (8000)<br/>APM-instrumented demo app"]
    webapp -- traces/errors --> fleet["Fleet Server APM (8200)"]
    kibana["Kibana"] -- APM traces --> es
    fleet --> es[("Elasticsearch")]
    kibana2["Kibana + Elasticsearch<br/>self-instrumentation"] -- APM --> fleet
```

## Agent Profile (`--profile agent`)

```mermaid
flowchart LR
    files["agent_ingest_data/*"] -- Custom Logs integration --> agent["container-agent"]
    udp["syslog UDP (9003)"] --> agent
    tcp["syslog TCP (9004)"] --> agent
    agent -- enrolled via --> fleet["Fleet Server (8220)"]
    agent -- "logs-generic-*, logs-udp.*, logs-tcp.*" --> es[("Elasticsearch")]
    es --> kibana["Kibana"]
```

## Kafka Profile (`--profile kafka`)

Dual Logstash pipeline with a TLS-secured Kafka broker (KRaft, no Zookeeper).
The broker certificate is issued from the stack CA by the setup service.

```mermaid
flowchart LR
    files["kafka_ingest_data/*"] --> lsin["logstash-kafka-in<br/>(producer)"]
    gen["demo generator"] --> lsin
    ksetup["kafka-setup<br/>(creates topics)"] -. SSL .-> kafka
    lsin -- SSL --> kafka[["Kafka broker<br/>internal 9092 / external 9094<br/>topic(s): KAFKA_TOPIC / KAFKA_TOPICS"]]
    kafka -- SSL --> lsout["logstash-kafka-out<br/>(consumer)"]
    kui["Kafka UI (8082)"] -. SSL .-> kafka
    ext["External producers/consumers"] -. "SSL via DOCKER_HOST_IP:9094" .-> kafka
    lsout -- "kafka-logs-*" --> es[("Elasticsearch")]
    es --> kibana["Kibana"]
```

## MCP Profile (`--profile mcp`)

```mermaid
flowchart LR
    llm["LLM client / agent"] -- "streamable HTTP<br/>localhost:8090/mcp" --> mcp["mcp-server"]
    mcp -- queries --> es[("Elasticsearch (9200)")]
```

## Air-Gapped Deployment (`-f air-gapped.yml`)

```mermaid
flowchart LR
    subgraph registries["Local registries (offline)"]
        epr["Elastic Package Registry<br/>(EPR, ~15G)"]
        ear["Elastic Artifact Registry<br/>(EAR, ~8G)"]
    end
    kibana["Kibana"] -- integration packages --> epr
    agents["Elastic Agents"] -- agent binaries /<br/>Elastic Defend artifacts --> ear
    kibana --> es[("Elasticsearch")]
    fleet["Fleet Server"] --> es
```

## Elastic Maps Server (`-f elastic-maps-server.yml`)

```mermaid
flowchart LR
    kibana["Kibana map visualizations"] -- tiles/boundaries --> ems["ems-server"]
    ems -. basemap data .-> mapsdata[("mapsdata volume")]
    kibana --> es[("Elasticsearch")]
```

## Certificate / TLS Trust Flow

Every service trusts every other service through the single CA in the shared
`certs` volume (self-generated by default, or externally provided).

```mermaid
flowchart TB
    ca["CA<br/>certs/ca/ca.crt + ca.key"] -- signs --> esc["es01/es02/es03<br/>ml01 / fz01 certs"]
    ca -- signs --> kbc["kibana cert"]
    ca -- signs --> flc["fleet-server cert"]
    ca -- signs --> emsc["ems-server cert"]
    ca -- signs --> kfc["kafka cert + PKCS12 keystore"]
    clients["All clients<br/>(Beats, Logstash, Agents,<br/>Kafka clients, curl)"] -- trust via ca.crt --> ca
```

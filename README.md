# Fluent Bit Kubernetes Log Processor Configuration

This repository contains the configuration files for Fluent Bit, which is used to collect, process, and forward logs from a Kubernetes cluster to Elasticsearch. The configuration uses a custom Lua script to dynamically set indices based on the Kubernetes namespace and container name.

## Table of Contents

- [Overview](#overview)
- [Configuration Files](#configuration-files)
  - [setIndex.lua](#setindexlua)
  - [fluent-bit.conf](#fluent-bitconf)
- [Usage](#usage)
- [Environment Variables](#environment-variables)
- [License](#license)

## Overview

Fluent Bit is a lightweight log shipper and processor used to collect, process, and forward logs from various sources. In this setup:

- Logs from Kubernetes containers are collected.
- A Lua script (`setIndex.lua`) is used to dynamically set the Elasticsearch index based on the container's namespace and container name.
- Logs from the `logging` namespace are excluded from processing.
- The logs are then pushed to an Elasticsearch cluster for storage and analysis.

### Key Features:
- Skip logs from the `logging` namespace.
- Dynamically set the Elasticsearch index based on Kubernetes namespace and container name.
- Send logs to an Elasticsearch server for indexing and analysis.
- Include Kubernetes metadata (such as pod name, namespace, container name) in the log data.

## Configuration Files

### 1. `setIndex.lua`

This Lua script is used to determine the Elasticsearch index name based on the Kubernetes namespace and container name. It skips logs from the `logging` namespace and formats the index accordingly.

```lua
function set_index(tag, timestamp, record)
    index = "donfortune-"
    if record["kubernetes"] ~= nil then
        if record["kubernetes"]["namespace_name"] == "logging" then
            return -1, timestamp, record  -- Skip logs from the logging namespace
        end
        if record["kubernetes"]["namespace_name"] ~= nil then
            if record["kubernetes"]["container_name"] ~= nil then
                record["es_index"] = index
                    .. record["kubernetes"]["namespace_name"]
                    .. "-"
                    .. record["kubernetes"]["container_name"]
                return 1, timestamp, record
            end
            record["es_index"] = index
                .. record["kubernetes"]["namespace_name"]
            return 1, timestamp, record
        end
    end
    return 1, timestamp, record
end 
```

### 2. `fluent-bit.conf`

This is the main Fluent Bit configuration file. It configures Fluent Bit's service settings, input sources (log files), filters (Lua script, Kubernetes metadata), and output destinations (Elasticsearch).
```
[SERVICE]
    Daemon Off
    Flush {{ .Values.flush }}
    Log_Level {{ .Values.logLevel }}
    Parsers_File /fluent-bit/etc/parsers.conf
    Parsers_File /fluent-bit/etc/conf/custom_parsers.conf
    HTTP_Server On
    HTTP_Listen 0.0.0.0
    HTTP_Port {{ .Values.metricsPort }}
    Health_Check On

[INPUT]
    Name tail
    Path /var/log/containers/*.log
    multiline.parser docker, cri
    Tag kube.*
    Mem_Buf_Limit 5MB
    Skip_Long_Lines On

[INPUT]
    Name systemd
    Tag host.*
    Systemd_Filter _SYSTEMD_UNIT=kubelet.service
    Read_From_Tail On

[FILTER]
    Name kubernetes
    Match kube.*
    Merge_Log On
    Keep_Log Off
    K8S-Logging.Parser On
    K8S-Logging.Exclude On

[FILTER]
    Name lua
    Match kube.*
    script /fluent-bit/scripts/setIndex.lua
    call set_index

[OUTPUT]
    Name es
    Match kube.*
    Type _doc
    Host elasticsearch-master
    Port 9200
    # Elasticsearch username for authentication
    # This is the user that Fluent Bit will use to connect to Elasticsearch.
    # The username should have appropriate permissions to push logs into Elasticsearch.
    HTTP_User elastic  # <-- Here you reference the Elasticsearch username.

    # Elasticsearch password for authentication
    # This is the password corresponding to the username defined above.
    # It ensures that only authorized users can send logs to Elasticsearch.
    HTTP_Passwd cbTQj1qxRIPNF5uc  # <-- Here you reference the Elasticsearch password.

    tls On
    tls.verify Off
    Logstash_Format On
    Logstash_Prefix logstash
    Retry_Limit False
    Suppress_Type_Name On

[OUTPUT]
    Name es
    Match host.*
    Type _doc
    Host elasticsearch-master
    Port 9200
    # Elasticsearch username for authentication
    HTTP_User elastic  # <-- Again, reference the username here for the system logs.

    # Elasticsearch password for authentication
    HTTP_Passwd cbTQj1qxRIPNF5uc  # <-- And reference the password here.

    tls On
    tls.verify Off
    Logstash_Format On
    Logstash_Prefix node
    Retry_Limit False
    Suppress_Type_Name On

```
    
## Usage

### Prerequisites:
- Fluent Bit must be installed in your Kubernetes cluster, and you'll need `Helm` installed locally.
- Elasticsearch should be set up and accessible from Fluent Bit.

### Steps:

1. **Clone this repository:**
    ```bash
    git clone https://github.com/donfortune/fluentbbit-kubernetes-logs.git
    cd fluent-bit-kubernetes-logs
    ```

2. **Deploy Fluent Bit using Helm:**

    - **Add the Fluent Bit Helm repository**:
        ```bash
        helm repo add fluent https://fluent.github.io/helm-charts
        helm repo update
        ```

    - **Create a ConfigMap** with the configuration files:
        ```bash
        kubectl create configmap fluent-bit-config --from-file=fluent-bit.conf --from-file=setIndex.lua -n kube-system
        ```

    - **Install Fluent Bit** using Helm with the custom configuration:
        ```bash
        helm install fluent-bit fluent/fluent-bit \
            --namespace kube-system \
            --set config.configMap=fluent-bit-config
        ```

3. **Monitor Elasticsearch:** Once Fluent Bit is deployed and logs are being pushed, monitor Elasticsearch for the newly created indices based on the configured prefixes (`logstash-*` for Kubernetes logs and `node-*` for system logs).

4. **Access Fluent Bit Metrics:** Fluent Bit exposes metrics through an HTTP server. You can access them by visiting the HTTP server port (configured in `fluent-bit.conf`).

## Environment Variables

You may want to customize certain variables in the configuration file. Here are some options:

- **`flush`**: Time interval for flushing logs.
- **`logLevel`**: The logging level for Fluent Bit (e.g., info, debug).
- **`metricsPort`**: The port for the HTTP server to expose metrics.

These variables can be set dynamically, depending on your deployment platform.

## License

This project is licensed under the MIT License - see the LICENSE file for details.

---

This `README.md` provides clear instructions for setting up and using the Fluent Bit configuration in a Kubernetes environment with Helm. It includes details about how the Lua script and various Fluent Bit settings work. You can paste this directly into your repository to help others understand how to use your setup.


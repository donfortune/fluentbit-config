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

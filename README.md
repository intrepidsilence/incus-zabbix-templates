# Zabbix Templates for Incus

A comprehensive Zabbix 7.4+ template suite for monitoring Incus container and VM clusters using the REST API and Prometheus metrics.

## Architecture

This template suite uses a **VMware-style host prototype architecture** where discovered entities become separate Zabbix hosts:

```
Manual Host: "Incus Cluster"
  └── Incus Cluster by HTTP (main template)
        ├── Cluster-level monitoring (totals, warnings, storage, networks)
        └── Discovers cluster members → creates member hosts
              └── Each member discovers instances → creates instance hosts
```

### Templates

| Template | Purpose | Created By |
|----------|---------|------------|
| **Incus Cluster by HTTP** | Main entry point, cluster metrics, member discovery | Manual link |
| **Incus Cluster Member** | Per-node daemon health, instance discovery | Host prototype |
| **Incus System Container** | Full Linux container monitoring | Host prototype |
| **Incus OCI Container** | Docker-style app container monitoring | Host prototype |
| **Incus Virtual Machine** | QEMU/KVM VM monitoring | Host prototype |

### Auto-Created Host Groups

- `Incus/Members` - All cluster members
- `Incus/System Containers` - All system containers
- `Incus/OCI Containers` - All OCI/app containers
- `Incus/Virtual Machines` - All VMs
- `Incus/Projects/{project}` - Instances by project
- `Incus/Members/{member}` - Instances on each member

## Features

- **Cluster Monitoring**: Member status, online count, warnings
- **Instance Monitoring**: CPU, memory, disk, network metrics per container/VM
- **Three Instance Types**: System containers, OCI containers, and VMs with type-specific monitoring
- **Storage Pool Monitoring**: Usage and status
- **Network Monitoring**: Managed network status
- **Snapshot Tracking**: Count and age (system containers and VMs)
- **Daemon Health**: Prometheus metrics per cluster member

## Requirements

- Zabbix Server 7.4 or later (required for nested host prototypes)
- Incus server 6.0+ with REST API enabled
- TLS client certificate for API authentication

## Installation

### 1. Import Templates

Import all template files in order:

1. `templates/template_incus_cluster.yaml` (main template)
2. `templates/template_incus_member.yaml`
3. `templates/template_incus_system_container.yaml`
4. `templates/template_incus_oci_container.yaml`
5. `templates/template_incus_virtual_machine.yaml`

### 2. Generate Client Certificates

```bash
# Generate client certificate
openssl genrsa -out client.key 4096
openssl req -new -key client.key -out client.csr -subj "/CN=zabbix-monitoring"
openssl x509 -req -days 3650 -in client.csr -signkey client.key -out client.crt
rm client.csr

# Add to Incus trust store
incus config trust add-certificate client.crt
```

### 3. Install Certificates on Zabbix Server

```bash
sudo mkdir -p /usr/share/zabbix/ssl/certs
sudo mkdir -p /usr/share/zabbix/ssl/keys
sudo cp client.crt /usr/share/zabbix/ssl/certs/incus-client.crt
sudo cp client.key /usr/share/zabbix/ssl/keys/incus-client.key
sudo chown zabbix:zabbix /usr/share/zabbix/ssl/certs/incus-client.crt
sudo chown zabbix:zabbix /usr/share/zabbix/ssl/keys/incus-client.key
sudo chmod 644 /usr/share/zabbix/ssl/certs/incus-client.crt
sudo chmod 600 /usr/share/zabbix/ssl/keys/incus-client.key
```

### 4. Create Host

1. Go to **Data collection** > **Hosts** > **Create host**
2. Set hostname (e.g., "Incus Cluster")
3. Link the **Incus Cluster by HTTP** template
4. Configure host macros:

| Macro | Value |
|-------|-------|
| `{$INCUS.API.HOST}` | Your Incus server hostname |
| `{$INCUS.API.PORT}` | `8443` (default) |
| `{$INCUS.TLS.CERT}` | `incus-client.crt` |
| `{$INCUS.TLS.KEY}` | `incus-client.key` |

### 5. Wait for Discovery

- Cluster members are discovered and hosts created automatically
- Instances on each member are then discovered
- Default discovery interval is 1 hour (configurable via `{$INCUS.DISCOVERY.INTERVAL}`)

## Configuration Macros

| Macro | Default | Description |
|-------|---------|-------------|
| `{$INCUS.API.HOST}` | `localhost` | Incus server hostname |
| `{$INCUS.API.PORT}` | `8443` | API port |
| `{$INCUS.TLS.CERT}` | `incus-client.crt` | Client certificate filename |
| `{$INCUS.TLS.KEY}` | `incus-client.key` | Client key filename |
| `{$INCUS.DATA.INTERVAL}` | `5m` | Master data collection interval |
| `{$INCUS.STATE.INTERVAL}` | `1m` | Instance state polling interval |
| `{$INCUS.METRICS.INTERVAL}` | `1m` | Prometheus metrics interval |
| `{$INCUS.DISCOVERY.INTERVAL}` | `1h` | Discovery interval |
| `{$INCUS.INSTANCE.IGNORE}` | `^$` | Regex to ignore instances |
| `{$INCUS.STORAGE.PUSED.WARN}` | `80` | Storage warning threshold % |
| `{$INCUS.STORAGE.PUSED.CRIT}` | `95` | Storage critical threshold % |
| `{$INCUS.SNAPSHOT.MAXAGE}` | `7d` | Snapshot age warning |
| `{$INCUS.MAINTENANCE}` | `0` | Set to 1 to suppress triggers |

## Metrics Collected

### Cluster Level
- Cluster enabled status, member count, online members
- Total instances by type and state
- Warning count, storage pool usage, network status

### Per Member
- Member status (Online/Offline/Evacuated/Blocked)
- Daemon uptime, goroutines, heap memory, operations

### Per Instance (Container/VM)
- Status, CPU usage, memory usage/peak/limit
- Disk usage, process count
- Network I/O per interface
- Snapshot count and age (system containers/VMs only)

### OCI Container Specific
- OCI entrypoint, source image
- No snapshot monitoring (typically rebuilt from images)

## Triggers

- API connectivity issues (Disaster)
- Cluster member offline (High)
- Instance error state (High)
- Storage pool space warnings (Warning/High)
- Old snapshots (Warning)
- Network interface errors (Warning)

## Upgrading from v1.x

The v2.0 architecture is significantly different from v1.x:

1. **Backup** your existing template and host configuration
2. **Remove** the old "Incus Cluster by HTTP" template link
3. **Import** all 5 new templates
4. **Re-link** the new "Incus Cluster by HTTP" template
5. **Wait** for discovery to create member and instance hosts

Note: Historical data will not be migrated automatically.

## License

MIT License

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

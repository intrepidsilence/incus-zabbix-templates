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

Import templates in dependency order (child templates first):

```
1. templates/template_incus_system_container.yaml
2. templates/template_incus_oci_container.yaml
3. templates/template_incus_virtual_machine.yaml
4. templates/template_incus_member.yaml
5. templates/template_incus_cluster.yaml
```

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
3. Add to a host group (e.g., "Incus")
4. Add an Agent interface (required placeholder, IP can be 127.0.0.1)
5. Link the **Incus Cluster by HTTP** template
6. Configure host macros:

| Macro | Value |
|-------|-------|
| `{$INCUS.API.HOST}` | Your Incus server hostname |
| `{$INCUS.TLS.CERT}` | `incus-client.crt` |
| `{$INCUS.TLS.KEY}` | `incus-client.key` |

### 5. Wait for Discovery

- Cluster members are discovered and hosts created automatically
- Instances on each member are then discovered
- Default discovery interval is 1 hour (configurable via `{$INCUS.DISCOVERY.INTERVAL}`)

## Configuration Macros

### Required Macros (set on cluster host)

| Macro | Default | Description |
|-------|---------|-------------|
| `{$INCUS.API.HOST}` | - | Incus server hostname (required) |
| `{$INCUS.TLS.CERT}` | `incus-client.crt` | Client certificate filename |
| `{$INCUS.TLS.KEY}` | `incus-client.key` | Client key filename |

### Optional Macros

| Macro | Default | Description |
|-------|---------|-------------|
| `{$INCUS.API.PORT}` | `8443` | API port |
| `{$INCUS.API.SCHEME}` | `https` | Protocol scheme |
| `{$INCUS.DATA.INTERVAL}` | `5m` | Master data collection interval |
| `{$INCUS.STATE.INTERVAL}` | `1m` | Instance state polling interval |
| `{$INCUS.METRICS.INTERVAL}` | `1m` | Prometheus metrics interval |
| `{$INCUS.DISCOVERY.INTERVAL}` | `1h` | Discovery interval |
| `{$INCUS.MAINTENANCE}` | `0` | Set to 1 to suppress triggers |

### Filter Macros

| Macro | Default | Description |
|-------|---------|-------------|
| `{$INCUS.INSTANCE.IGNORE}` | `^$` | Regex to ignore instances |
| `{$INCUS.POOL.IGNORE}` | `^$` | Regex to ignore storage pools |
| `{$INCUS.NETWORK.IGNORE}` | `^(lo\|docker[0-9]*\|veth.*\|br-[a-f0-9]+)$` | Regex to ignore networks |
| `{$INCUS.IMAGE.IGNORE}` | `^$` | Regex to ignore images |
| `{$INCUS.INTERFACE.IGNORE}` | `^lo$` | Regex to ignore network interfaces |

### Threshold Macros

| Macro | Default | Description |
|-------|---------|-------------|
| `{$INCUS.STORAGE.PUSED.WARN}` | `80` | Storage warning threshold % |
| `{$INCUS.STORAGE.PUSED.CRIT}` | `95` | Storage critical threshold % |
| `{$INCUS.MEMORY.PUSED.WARN}` | `90` | Memory warning threshold % |
| `{$INCUS.MEMORY.PUSED.CRIT}` | `95` | Memory critical threshold % |
| `{$INCUS.SNAPSHOT.MAXAGE}` | `7d` | Snapshot age warning |

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
- Limited metrics (OCI containers typically rebuilt from images)
- No snapshot monitoring

## Host Tags

Discovered hosts automatically receive tags for filtering:

| Host Type | Tags |
|-----------|------|
| Members | `incus: member`, `member: {name}` |
| System Containers | `incus: instance`, `incus.type: system-container`, `instance: {name}`, `project: {project}` |
| OCI Containers | `incus: instance`, `incus.type: oci-container`, `instance: {name}`, `project: {project}` |
| Virtual Machines | `incus: instance`, `incus.type: virtual-machine`, `instance: {name}`, `project: {project}` |

## Triggers

- API connectivity issues (Disaster)
- Cluster member offline (High)
- Instance error state (High)
- Storage pool space warnings (Warning/High)
- Memory usage warnings (Warning/High)
- Old snapshots (Warning)
- Network interface errors (Warning)

## Troubleshooting

### Items show "Not supported"

Check the error message on the item. Common issues:
- Certificate not found: Verify cert files exist in `/usr/share/zabbix/ssl/certs/` and `/usr/share/zabbix/ssl/keys/`
- Connection refused: Ensure Incus is listening on port 8443 (`incus config set core.https_address :8443`)
- Certificate not trusted: Add cert to Incus trust store (`incus config trust add-certificate`)

### Discovery not creating hosts

- Check that master items are collecting data (Latest data > show raw data)
- Verify the cluster host has the correct template linked
- Check Zabbix server logs for discovery errors

### Test connectivity manually

```bash
curl -v --cert /usr/share/zabbix/ssl/certs/incus-client.crt \
        --key /usr/share/zabbix/ssl/keys/incus-client.key \
        -k https://<incus-host>:8443/1.0
```

## License

MIT License

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

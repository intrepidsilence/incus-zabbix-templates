# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Zabbix 7.4+ monitoring template suite for Incus container and VM clusters. The templates use a VMware-style host prototype architecture where discovered entities (cluster members, containers, VMs) become separate Zabbix hosts.

## Template Architecture

### Template Hierarchy

```
Manual Host: "Incus Cluster"
  └── Template: Incus Cluster by HTTP
        ├── Cluster-level items (totals, warnings, storage, networks)
        └── Discovery: Cluster Members
              └── Host Prototype → "Incus Cluster Member" template
                    └── Nested Discovery: Instances
                          ├── Host Prototype [system container] → "Incus System Container"
                          ├── Host Prototype [OCI container] → "Incus OCI Container"
                          └── Host Prototype [VM] → "Incus Virtual Machine"
```

### Template Files

| Template | File | Purpose |
|----------|------|---------|
| Incus Cluster by HTTP | `templates/template_incus_cluster.yaml` | Main template, cluster-level monitoring, member discovery |
| Incus Cluster Member | `templates/template_incus_member.yaml` | Per-member metrics, instance discovery |
| Incus System Container | `templates/template_incus_system_container.yaml` | Full system container monitoring |
| Incus OCI Container | `templates/template_incus_oci_container.yaml` | OCI/Docker-style container monitoring |
| Incus Virtual Machine | `templates/template_incus_virtual_machine.yaml` | VM monitoring |

### Instance Type Detection

Instances are differentiated using:
- `type` field: `container` or `virtual-machine`
- `config.volatile.container.oci`: `"true"` for OCI containers

### Auto-Created Host Groups

- `Incus/Members` - All cluster members
- `Incus/Members/{member}` - Instances on each member
- `Incus/System Containers` - All system containers
- `Incus/OCI Containers` - All OCI/app containers
- `Incus/Virtual Machines` - All VMs
- `Incus/Projects/{project}` - Instances by project

## Working with the Templates

### Zabbix YAML Format

Templates use Zabbix 7.4 YAML export format. Key structures:
- `host_prototypes[]` in discovery rules create new Zabbix hosts
- `group_prototypes[]` define dynamic host groups
- `templates[]` in host prototypes link child templates
- `macros[]` in host prototypes pass LLD values to child hosts

### Macro Inheritance

```
Cluster Host (manual)
  {$INCUS.API.HOST}, {$INCUS.TLS.CERT}, {$INCUS.TLS.KEY}
      ↓ inherited
Member Host (discovered)
  {$INCUS.MEMBER.NAME}, {$INCUS.MEMBER.ADDRESS} ← from LLD
      ↓ inherited
Instance Host (discovered)
  {$INCUS.INSTANCE.NAME}, {$INCUS.INSTANCE.PROJECT} ← from LLD
```

### Adding New Items

1. Identify which template the item belongs to (cluster, member, or instance)
2. Add master item (HTTP_AGENT) if new API endpoint needed
3. Add dependent items with JSONPath preprocessing
4. Follow key naming: `incus.<category>.<metric>`

### Host Prototype Configuration

When modifying host prototypes:
- `host` field must contain at least one LLD macro for uniqueness
- `name` field is the visible name (can be more readable)
- `templates` links child templates
- `macros` converts LLD macros to host macros for child template use

## Incus API Reference

The templates interact with these endpoints:
- `/1.0` - Server info, health check
- `/1.0/cluster` - Cluster status
- `/1.0/cluster/members` - Member discovery
- `/1.0/cluster/members/{name}` - Per-member status
- `/1.0/instances?recursion=2&all-projects=true` - All instances
- `/1.0/instances?target={member}` - Instances on specific member
- `/1.0/instances/{name}/state?project={project}` - Instance metrics
- `/1.0/storage-pools?recursion=2` - Storage pool usage
- `/1.0/networks?recursion=1` - Network configuration
- `/1.0/images?recursion=1` - Cached images
- `/1.0/metrics` - Prometheus-format daemon metrics

## Key Design Decisions

1. **Host prototypes vs item prototypes**: Using host prototypes (VMware-style) for better per-entity dashboards and permissions
2. **Nested discovery**: Members discover their own instances for proper hierarchy
3. **Three instance templates**: System containers, OCI containers, and VMs have different monitoring needs
4. **OCI detection**: Uses `volatile.container.oci` config field to identify OCI containers

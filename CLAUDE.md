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

### Import Order

Templates must be imported in dependency order (children first):

1. `template_incus_system_container.yaml`
2. `template_incus_oci_container.yaml`
3. `template_incus_virtual_machine.yaml`
4. `template_incus_member.yaml`
5. `template_incus_cluster.yaml`

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
- `group_prototypes[]` define dynamic host groups (use LLD macros only)
- `templates[]` in host prototypes link child templates
- `macros[]` in host prototypes pass LLD values to child hosts as user macros

### Macro Types and Usage

**LLD Macros** (`{#MACRO}`):
- Created by discovery from JSON data via `lld_macro_paths`
- Used in host prototype names, group prototypes, and filter conditions
- Resolved at discovery time

**User Macros** (`{$MACRO}`):
- Defined on hosts/templates, inherited down the hierarchy
- Used in item URLs, intervals, thresholds, trigger expressions
- **NOT resolved in tag values** on template items (use host-level tags instead)

### Macro Inheritance

```
Cluster Host (manual)
  {$INCUS.API.HOST}, {$INCUS.TLS.CERT}, {$INCUS.TLS.KEY}
      ↓ inherited
Member Host (discovered)
  {$INCUS.MEMBER.NAME}, {$INCUS.MEMBER.ADDRESS} ← from LLD via host prototype
      ↓ inherited
Instance Host (discovered)
  {$INCUS.INSTANCE.NAME}, {$INCUS.INSTANCE.PROJECT} ← from LLD via host prototype
```

### Host-Level Tags (Resolved Correctly)

Tags set on host prototypes using LLD macros resolve correctly:
```yaml
host_prototypes:
  - host: '{#INSTANCE.PROJECT}-{#INSTANCE.NAME}'
    name: '{#INSTANCE.NAME}'
    tags:
      - tag: instance
        value: '{#INSTANCE.NAME}'    # ✓ Resolves correctly
      - tag: project
        value: '{#INSTANCE.PROJECT}' # ✓ Resolves correctly
```

### Item-Level Tags (Use Static Values Only)

User macros in template item tags do NOT resolve:
```yaml
items:
  - name: 'CPU usage'
    tags:
      - tag: component
        value: cpu                    # ✓ Static value works
      - tag: instance
        value: '{$INCUS.INSTANCE.NAME}' # ✗ Shows literal macro text
```

### Adding New Items

1. Identify which template the item belongs to (cluster, member, or instance)
2. Add master item (HTTP_AGENT) if new API endpoint needed
3. Add dependent items with JSONPath preprocessing
4. Follow key naming: `incus.<category>.<metric>`
5. Use only static values or `component:` tags on items
6. Instance/member identification comes from host-level tags

### Host Prototype Configuration

When modifying host prototypes:
- `host` field must contain at least one LLD macro for uniqueness
- `name` field is the visible name (can be more readable)
- `templates` links child templates
- `macros` converts LLD macros to user macros for child template use
- `tags` set host-level tags (use LLD macros here)

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
5. **Tags at host level**: Instance/member identification via host tags (not item tags) because user macros don't resolve in template item tag values
6. **Simplified instance names**: Visible name shows just instance name; project tracked via tag

## Common Issues

### Unresolved Macros in Tags
If you see `{$INCUS.INSTANCE.NAME}` literally in the UI, the tag is defined on a template item using a user macro. Move such tags to the host prototype level using LLD macros instead.

### Host Prototype Interfaces
User macros (`{$MACRO}`) do NOT resolve directly in host prototype interface definitions - only LLD macros (`{#MACRO}`) work there.

- **Member hosts**: Use `{#MEMBER.ADDRESS}` (LLD macro from cluster member discovery) → resolves to actual IP
- **Instance hosts**: Use `{#INSTANCE.ADDRESS}` (extracted from instance state.network)

The instance discovery extracts the instance's own IPv4 address:
1. JavaScript iterates `state.network` interfaces, finds first global IPv4 address
2. If no IP found (stopped instance, no agent), falls back to `__MEMBER_ADDR__` placeholder
3. STR_REPLACE substitutes `__MEMBER_ADDR__` with `{$INCUS.MEMBER.ADDRESS}` (fallback to member IP)
4. LLD macro `{#INSTANCE.ADDRESS}` extracts the resolved IP
5. Host prototype interface uses `{#INSTANCE.ADDRESS}`

### Naming Convention
Discovered instance hosts use visible name `Incus: {#INSTANCE.NAME}` to avoid conflicts with pre-existing manual hosts that may share the same container names.

### UUIDs
Zabbix 7.4 requires valid UUIDv4 format. Generate with:
```python
import uuid
print(uuid.uuid4().hex)
```

### Template Import Errors
- Import in dependency order (children first)
- Ensure `template_groups` and `host_groups` are separate sections
- Group prototypes can only use LLD macros, not user macros
- always remember to keep all markdown files updated as changes dictate
- always update the github project as changes are made
- remember as we make changes to the templates to go ahead and clean up hosts in zabbix and reconfigure them and test to be sure everything is working in the templates
- remember to use the config.yaml file for the api credentials
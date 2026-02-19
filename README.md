# PostgreSQL High Availability Playbook (Patroni + etcd + HAProxy + keepalived)

This Ansible playbook automates deployment of a highly available PostgreSQL cluster using:
- **Patroni**: PostgreSQL cluster orchestration
- **etcd**: Distributed configuration store (mTLS)
- **HAProxy**: Load balancing with health checks
- **keepalived**: Virtual IP failover

## Topology
- PostgreSQL nodes (3): postgres-01, postgres-02, postgres-03 (192.168.60.103/104/105)
- HAProxy nodes (3): haproxy-01, haproxy-02, haproxy-03 (192.168.60.100/101/102)
- VIP: 192.168.60.110

## Quick Start

```bash
# Update inventory
vim inventory/hosts.yml

# Generate TLS certificates
ansible-playbook playbooks/generate_certificates.yml

# Deploy infrastructure
ansible-playbook playbooks/deploy_postgresql_ha.yml
```

## Roles
- `common`: System updates, packages, users
- `etcd`: Distributed key-value store with mTLS
- `postgresql`: PostgreSQL installation
- `patroni`: Patroni orchestration
- `haproxy`: Load balancer
- `keepalived`: Virtual IP management
- `certificates`: TLS certificate generation and distribution

## Requirements
- Ansible 2.9+
- Ubuntu/Debian family hosts
- SSH access to all nodes
- No existing PostgreSQL installations

2. Update the `inventory/hosts.yml` file to include your target hosts.

3. Customize the variables in the `roles/common/vars/main.yml`, `roles/webserver/vars/main.yml`, and `roles/database/vars/main.yml` files as needed.

4. Run the playbook using the following command:
   ```
   ansible-playbook playbook.yml -i inventory/hosts.yml
   ```

## Roles Overview

- **Common**: Contains tasks and configurations that are shared across all servers.
- **Webserver**: Manages the installation and configuration of web server software and applications.
- **Database**: Handles the installation and configuration of database software and user management.

## License

This project is licensed under the MIT License - see the LICENSE file for details.
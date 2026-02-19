# PostgreSQL High Availability Playbook (Patroni + etcd + HAProxy + keepalived)

This Ansible playbook automates the deployment of a highly available PostgreSQL cluster using:
- **Patroni**: PostgreSQL cluster orchestration and automatic failover management.
- **etcd**: Distributed configuration store (mTLS) to maintain cluster state and consensus.
- **HAProxy**: Load balancing with health checks to route traffic to the active primary node.
- **keepalived**: Virtual IP failover to ensure the routing layer itself is highly available.

## Topology & Architecture

The following diagram illustrates the exact traffic flow and node architecture for this deployment:

```text
      +-------------------------------------------------------+
      |                   Client Applications                 |
      +---------------------------+---------------------------+
                                  |
                                  v
                      +-----------------------+
                      |   Virtual IP (VIP)    |
                      |    192.168.60.110     |
                      |     (keepalived)      |
                      +-----------+-----------+
                                  |
            +---------------------+---------------------+
            |                     |                     |
            v                     v                     v
 +-------------------+ +-------------------+ +-------------------+
 |    haproxy-01     | |    haproxy-02     | |    haproxy-03     |
 |  192.168.60.100   | |  192.168.60.101   | |  192.168.60.102   |
 +---------+---------+ +---------+---------+ +---------+---------+
           |                     |                     |
           +---------------------+---------------------+
                                 |
                         (Read/Write Routing)
                                 |
            +--------------------+----------------------+
            |                    |                      |
            v                    v                      v
 +-------------------+ +-------------------+ +-------------------+
 |    postgres-01    | |    postgres-02    | |    postgres-03    |
 |  192.168.60.103   | |  192.168.60.104   | |  192.168.60.105   |
 |   (PG + Patroni)  | |   (PG + Patroni)  | |   (PG + Patroni)  |
 +---------+---------+ +---------+---------+ +---------+---------+
           |                     |                      |
           +---------------------+----------------------+
                                 |
                                 v
                      +-----------------------+
                      |      etcd Cluster     |
                      | (Distributed Consensus|
                      |       Store)          |
                      +-----------------------+
```

- **PostgreSQL nodes (3)**: postgres-01, postgres-02, postgres-03 (192.168.60.103/104/105)
- **HAProxy nodes (3)**: haproxy-01, haproxy-02, haproxy-03 (192.168.60.100/101/102)
- **VIP**: 192.168.60.110

## Requirements
- Ansible 2.9+
- Ubuntu/Debian family hosts
- SSH access to all nodes
- No existing PostgreSQL installations

## Quick Start

1. Update the `inventory/hosts.yml` file to include your target hosts:
   ```bash
   vim inventory/hosts.yml
   ```

2. Customize the variables in the `roles/common/vars/main.yml`, `roles/webserver/vars/main.yml`, and `roles/database/vars/main.yml` files as needed.

3. Generate TLS certificates:
   ```bash
   ansible-playbook playbooks/generate_certificates.yml
   ```

4. Run the playbook to deploy the infrastructure:
   ```bash
   ansible-playbook playbooks/deploy_postgresql_ha.yml -i inventory/hosts.yml
   ```

## Roles Overview

- **`common`**: Contains tasks and configurations that are shared across all servers (System updates, packages, users).
- **`etcd`**: Distributed key-value store with mTLS.
- **`postgresql`**: PostgreSQL installation.
- **`patroni`**: Patroni orchestration.
- **`haproxy`**: Load balancer.
- **`keepalived`**: Virtual IP management.
- **`certificates`**: TLS certificate generation and distribution.
- **`webserver`**: Manages the installation and configuration of web server software and applications.
- **`database`**: Handles the installation and configuration of database software and user management.

---

## Patroni Procedure (TSE Server)

> **NOTICE:** All maintenance and database operations must be performed **outside of market hours** (the market time is over or has not started 9:00 AM - 3:00 PM).

### Pro-Tip
Instead of using `-c /etc/patroni/patroni.yaml` with `patronictl` you can set an alias in your `.profile` or `.bashrc` file:
```bash
alias patronictl='patronictl -c /etc/patroni/patroni.yaml'
```

### `patronictl list`
List all nodes and their roles/status. You can use it for checking the status of all nodes, identifying the master, and viewing slaves/replicas.
- **When to use:** To check the list and status of all nodes in the cluster, including if any restart is required for any node.
- **How to use:**
  ```bash
  patronictl -c /etc/patroni/patroni.yaml list
  ```

### `patronictl edit-config`
To edit PostgreSQL configuration parameters. It will open the configuration file in an editor, make the required changes, and Patroni will validate all parameters before saving. You can also add `pg_hba` entries to the configuration so these will be reflected all over the cluster.
*💡 If you directly update parameter values in Postgres configuration files, they will be overwritten via Patroni configuration if the same parameter is explicitly defined in the patroni config.*
- **When to use:** If you want to change a postgres parameter or add/remove `pg_hba` entries safely.
- **How to use:**
  ```bash
  # /etc/patroni/patroni.yaml is the configuration file for patroni
  patronictl -c /etc/patroni/patroni.yaml edit-config
  ```

### `patronictl reload`
This command will reload parameters from the configuration file and take required actions like a restart on cluster nodes if needed.
- **When to use:** If you have changed parameters in the configuration file using `edit-config`, you can use the reload command for parameters to take effect.
- **How to use:**
  ```bash
  # patroni_cluster is the name of your cluster
  patronictl -c /etc/patroni/patroni.yaml reload patroni_cluster
  ```

### `patronictl switchover`
It will make the selected replica the master node, essentially switching all traffic to the newly selected node. You can also schedule a planned switchover at a particular time.
- **When to use:** If you have maintenance for the master node, you can safely switch over the master to another node in the cluster.
- **How to use:**
  ```bash
  # It will ask for the node to switch over to and also the time for switchover
  patronictl -c /etc/patroni/patroni.yaml switchover
  ```

### `patronictl restart`
It will restart a single node in the postgres cluster or all nodes (complete cluster). Patroni will execute a rolling restart for postgres on all nodes.
- **When to use:** Sometimes you need to restart all nodes in the cluster without downtime; you can use this command for a safe rolling restart.
- **How to use:**
  ```bash
  # Restart a particular node in the cluster
  patronictl -c /etc/patroni/patroni.yaml restart <CLUSTER_NAME> <NODE_NAME>

  # Restart the whole cluster (all nodes in the cluster)
  patronictl -c /etc/patroni/patroni.yaml restart <CLUSTER_NAME>
  ```

### `patronictl pause`
Patroni will stop managing the postgres cluster and will turn on maintenance mode. If you want to perform manual activities for maintenance, you need to stop Patroni from auto-managing the cluster to prevent unwanted failovers.
- **When to use:** If you want to put the cluster in maintenance mode and manage the Postgres database manually for a period of time.
- **How to use:**
  ```bash
  patronictl -c /etc/patroni/patroni.yaml pause
  ```

### `patronictl resume`
It will start the paused cluster management and remove the cluster from maintenance mode.
- **When to use:** If you want to turn off maintenance mode and allow Patroni to start managing the cluster automatically again.
- **How to use:**
  ```bash
  patronictl -c /etc/patroni/patroni.yaml resume
  ```

### `patronictl reinit`
Reinitialization in Patroni refers to the process of rebuilding a replica (standby) node from the primary.
> **NOTICE:** If there is a high lag on a node, we should use the reinit command to empty the WAL file and reinitialize the replica node. **(Do not do this during market hours.)**
- **When to use:** This command allows you to reinitialize a specific node and can be utilized when a cluster node experiences a failure in starting or displays an unknown status. It is often useful in cases where a node has corrupt data or falls too far behind the master.
- **How to use:**
  ```bash
  patronictl -c /etc/patroni/patroni.yaml reinit <CLUSTER_NAME> <NODE_NAME>
  ```

---

## License
This project is licensed under the MIT License - see the LICENSE file for details.

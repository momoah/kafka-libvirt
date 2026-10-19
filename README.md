# kafkalibvirt

Single-node Apache Kafka 4.3.1 (KRaft mode) on a RHEL 9.8 libvirt VM, as a demo endpoint for OpenShift Logging 6.x (or any Kafka-capable log forwarder).
Size-capped by design: a dedicated 100 GiB data disk plus topic retention limits, so the disk never fills.

## DNS records

```
; forward (local.labmesh.org)
kafka   IN A    192.168.1.237
; reverse (192.168.1.db)
237     IN PTR  kafka.local.labmesh.org.
```

## Run

```
ansible-playbook playbooks/01_build_vm.yml        # VM: 50 GiB root + 100 GiB data disk (virt-install + cloud-init seed ISO)
ansible-playbook playbooks/02_rhel_updates.yml    # Register, pin 9.8, update, reboot if needed
ansible-playbook playbooks/03_install_kafka.yml   # Java 21, mount /dev/vdb at /var/lib/kafka, Kafka tarball (sha512-verified), systemd unit
ansible-playbook playbooks/04_configure_kafka.yml # KRaft single node, TLS :9093, size-capped topic, TLS round-trip test
ansible-playbook playbooks/99_destroy_vm.yml      # Tear down the VM (root + data disks)
```

## Endpoint

| Item | Value |
|---|---|
| Forwarder URL | `tls://kafka.local.labmesh.org:9093` |
| Topic | `openshift-logs` |
| TLS | Wildcard `*.local.labmesh.org` cert (public CA chain), no client auth |
| Local admin listener | `127.0.0.1:9092` (plaintext, on the VM only) |

## Size limits (nothing accumulates)

| Layer | Limit |
|---|---|
| `openshift-logs` | 3 partitions x 20 GiB (`retention.bytes`), 24h (`retention.ms`), 256 MiB / 1h segments |
| Any other topic | Broker default 5 GiB per partition, 24h |
| Topic auto-creation | Disabled (`auto.create.topics.enable=false`) |
| Kafka app logs (`/var/log/kafka`) | Deleted after 3 days by systemd-tmpfiles |
| Worst case on disk | ~61 GiB of 100 GiB |

## Demo cheat sheet

Run on the VM (`ssh admin@kafka.local.labmesh.org`):

| What to show | Command |
|---|---|
| Live log stream | `/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic openshift-logs` |
| Only app logs, nicely formatted | `/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic openshift-logs \| jq -c 'select(.log_type=="application") \| {ns: .kubernetes.namespace_name, msg: .message}'` |
| Last 2 messages then exit | `/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic openshift-logs --max-messages 2 --timeout-ms 20000` |
| How much is stored (messages per partition) | `/opt/kafka/bin/kafka-get-offsets.sh --bootstrap-server 127.0.0.1:9092 --topic openshift-logs` |
| Size limits in effect | `/opt/kafka/bin/kafka-configs.sh --bootstrap-server 127.0.0.1:9092 --describe --entity-type topics --entity-name openshift-logs` |
| Disk can't fill | `df -h /var/lib/kafka` |
| Broker health | `sudo systemctl status kafka --no-pager` |
| Broker errors | `sudo grep -nE "ERROR\|FATAL\|Caused by" /var/log/kafka/server.log \| tail -20` |

## OpenShift Logging 6.x forwarder

Managed in the `general-apps` GitOps repo (`apps/openshift-logging/`): Namespace, OperatorGroup, Subscription (`stable-6.6`),
collector ServiceAccount + ClusterRoleBindings, trusted-CA ConfigMap, and the `ClusterLogForwarder` (`observability.openshift.io/v1`).

Key points:
- The Kafka output needs a `tls:` block with real content (we use `tls.ca` -> ConfigMap `kafka-trusted-ca`, key `ca-bundle.crt`), otherwise the collector sends plaintext to the TLS port.
- Collector resources on sno2: `requests.cpu: 100m` (node CPU requests are ~97% allocated), `limits.memory: 4Gi` (initial backlog catch-up).

## Gotchas

| Symptom | Cause | Fix |
|---|---|---|
| Kafka exits: `Certificate chain is not valid` | `fullchain.pem` contains the leaf cert twice | Keystore task stops at the repeated leaf |
| Collector: `Disconnected while requesting ApiVersion ... security.protocol` | Output had no (or an empty) `tls:` block | Add `tls.ca` |
| ArgoCD `Unknown` / repo-server `Pending` | Collector pod preempted ArgoCD on a full-CPU SNO | Lower collector CPU request |
| Collector `OOMKilled` | Backlog of existing node logs at first start | Raise `collector.resources.limits.memory` |

## Author(s)

- Mohammad Ahmad

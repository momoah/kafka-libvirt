# kafkalibvirt

Single-node Apache Kafka (KRaft mode) on a RHEL 9 libvirt VM, as a demo endpoint for OpenShift Logging (or any Kafka-capable log forwarder).
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

Endpoint for forwarders: `tls://kafka.local.labmesh.org:9093`, topic `openshift-logs`.
Disk use is capped at ~61 GiB of 100 GiB (3 partitions x 20 GiB + open segments); Kafka app logs older than 3 days are deleted by systemd-tmpfiles.

## Author(s)

- Mohammad Ahmad

# Proxmox Nodes

Vor der PVE-Installation mussten der storgae de rServer konfiguriert werden.
Dazu musste auf jeder physischen Festplatte eine virtuelle Festplatte mit RAID 0 erstellt werden.

Die installation de rNodes erfolgte mit einer ISO mit answerfile, die die konfiguration enthält.

## Namesnauflösung

Der `/etc/hosts` folgende Einträge anfügen:
```
192.168.88.231 pveha1.internal pveha1
192.168.88.232 pveha2.internal pveha2
192.168.88.240 pveha-sto.internal pveha-sto
```

## Cluster

### Cluster erstellen
Auf pveha1.internal:
```bash
pvecm create pveha
```

Auf pveha2.internal:
```bash
pvecm add pveha1.internal
```

Kontrolle:
```bash
pvecm status
pvecm nodes
```

### NFS-Storage einbinden
```bash
ansbible-playbook pve_storage.yml
```

### Qdevice einrichten
Auf pveha-sto `corosync-qnetd` und auf pveha1 & pveha2 `corosync-qdevice` installieren
```bash
apt install corosync-qdevice

sudo apt install corosync-qnetd
```

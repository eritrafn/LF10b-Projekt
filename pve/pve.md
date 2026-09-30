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
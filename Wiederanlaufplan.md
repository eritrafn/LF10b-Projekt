# Wiederanlaufplan

## Systemübersicht

| Komponente | Funktion | Kritikalität |
|---|---|---|
| Proxmox-Node | Virtualisierungshost (VMs/CTs) | Hoch |
| CheckMK | Monitoring aller Systeme | Mittel |
| Storage-Server | Storage-Storage (NFS/ZFS) | Hoch |
| Switch | Netzwerkverbindung | Hoch |

## Notfallszenarien
### Ausfall einer VM/Container

### Ausfall eines Proxmox-Nodes

### Ausfall mehrerer Proxmox-Nodes

### Ausfall des Storage-Server

## Wiederanlaufpriorität

1. Netzwerk
2. Proxmox-Nodes
3. Storage-Server
4. CheckMK

## Vorbeugende Maßnahmen
* Tägliche Backups
* Monitoring
* Restoretests
* Dokumentation und Wiederanlaufplan
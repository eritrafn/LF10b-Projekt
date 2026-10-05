# Reflexion

## Welcher Stand an Ausfallsicherheit konnte erreicht werden?
- Durch das HA-Clustering der beiden Proxmox-Nodes können LXC-Container mit minimaler Downtime (unter 1 Minute) migriert werden, VMs können live migriert werden. Das geht nur bei geplanter Migration. Bei einem ungeplanten Ausfall eines Nodes dauert der Wiederanlauf der CT/VM ggf etwas länger, da der RAM nicht übertragen werden kann und zuerst sichergestellt werden muss, dass der ausgefallene Host wirklich weg ist und nicht nir kurz unerreichbar ist, damit CT/VM nicht doppelt entsteht.
- Der Storage-Server selbst hat keine Ausfallsicherheit. (architekturbedingt)

### Was sind die größten verbleibenden Risiken?
- Der Storage-Server verfügt über keine Form der Redundanz, es gubt nur eine Festplatte, nur ein Netzteil, etc. Wenn dieser ausfällt, sind die CT/VMs alle down, da die Dateien auf dem NFS-Share liegen.
- Backups liegen ebenfalls auf dem Storage-Server und sind bei Ausfall von diesem ebenfalls nicht mehr verfügbar.
    - Das wurde so gewählt, um den Hardwareaufwand für das Projekt geringer zu halten.

## In welchem Umfang konnten Sie Automatisierung umsetzen?
- Nutzung von Answerfiles für die Proxmoxinstallation
- Ansible

## Welche Lernerfolge haben Sie erzielt?
- Sehr viel über Proxmox und die Funktionsweise eines PVE-Clusters
- Ansible

## In welchem Umfang konnten Sie die eigenen Projektziele erreichen?
- Installation und Einrichtung von ProxmoxVE
- Einrichtung eines PVE-Clusters
- Storage-Server mit ZFS und NFS-Shares eingerichtet
- Installation und Einrichtung von CheckMK in einem Container

### Nicht erreichte Ziele
- Erstellen einer VM auf dem Cluster. Bei jedem Versuch des Erstellens hat der entsprechende Node die Verbindung zu den NFS-Shares verloren.

## Anforderungen

### Backup

- [ ] Inkrementelle oder Differenzielle Backups
    - Aufgrund der Archtiektur mit NFS-Shares als VM-Storage sind nur Vollbackups mit vzdump möglich
- [x] Konfiguration des Monitoring wird gebackupt
    - Der gesamte Container inklusive der Konfiguration wird gesichert
- [x] Ein Backup wurde erfolgreich zurückgespielt
- [x] Backups werden automatisch erstellt
- [x] Benachrichtigung, wenn automatische Erstellung von Backups fehlschlägt (optional mittels Monitoring)
    - Wenn Service auf WARN oder CRIT geht

### Monitoring

- [x] läuft
    - CheckMK läuft in einem Container
- [x] Ram-Auslastung wird überwacht
- [x] verbleibende freie Festplatenkapazität wird überwacht
- [x] Erfolgreiche automatische Erstellung von Backups wird überwacht
    - Mit Proxmox Special Agent für CheckMK wird auch die Ausführung der Backups überwacht 
- [x] Im Fehlerfall werden Benachrichtigungen „versendet“
    - nicht funktional aber eingerichtet

### Automatisierung

- [x] Regelmäßige vollständige Updates gewährleisten
    - CheckMK Container mit unattended-upgrade
- [x] Automatische Wiederherstellung möglich
    - Answerfiles für die automatisierte Installation von Proxmox auf dem Server
    - Ansible Playbooks für die änderung des Update-Repos und der Einbindung der NFS-Shares
- [x] Lösung ermöglicht Review von Veränderungen (Change Management)
- [x] Idempotente Anwendung der Konfigurationsverwaltung funktioniert fehlerfrei
    - Ansible playbooks sind indepotent geschrieben
- [x] Mehrere Konfigurationen lassen sich miteinander kombinieren
    - Playbooks sind unabhängig voneinander
- [x] Ein einzelner Konfigurationsschritt kann einfach und sauber rückgängig gemacht werden

### Bonus

- [x] Virtualisierung, Containerisierung
    - ProxmoxVE
- [x] Dienste
    - CheckMK
- [ ] Skalierung
- [ ] Weitere Maßnahmen zur Steigerung der Verfügbarkeit
  - [] z.B. RAID, Netzwerkredundanz

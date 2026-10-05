# Wiederanlaufplan

Stand: 05.10.2026 · Verantwortlich: <Name> · Vertretung: <Name>

## Systemübersicht

| Komponente | Hostname / IP | Funktion | Kritikalität |
|---|---|---|---|
| Proxmox-Node 1 | pveha1 / 192.168.88.231 | Virtualisierungshost (VMs/CTs), Cluster `pveha` | Hoch |
| Proxmox-Node 2 | pveha2 / 192.168.88.232 | Virtualisierungshost (VMs/CTs), Cluster `pveha` | Hoch |
| Storage-Server | pveha-sto / 192.168.88.240 | Zentraler Shared Storage (Ubuntu, ZFS-Pool `pveha-sto-pool`, NFS-Exports `backup`, `vm-storage`, `iso`) und QDevice (`corosync-qnetd`) für das Cluster-Quorum | Hoch |
| CheckMK | CT 100 / 192.168.88.233 | Monitoring aller Systeme, als HA-Ressource registriert | Mittel |
| Switch | <Modell> | Netzwerkverbindung. Management-, Cluster- und VM-Traffic laufen über ein gemeinsames Interface (`vmbr0`/`nic0`) | Hoch |

### Abhängigkeiten
- Ohne Switch ist nichts erreichbar. Außerdem verlieren die Nodes den Corosync-Kontakt.
- Ohne Storage-Server sind alle VM/CT-Disks und Backups weg. Außerdem fehlt die dritte Quorum-Stimme (QDevice).
- Cluster-Quorum: 3 Stimmen (2 Nodes + QDevice). Quorum besteht bei mindestens 2 Stimmen. Ein Node plus QDevice reicht also.
- Ohne CheckMK laufen alle anderen Systeme weiter, nur das Monitoring fehlt.

### Wiederanlaufziele (Vorschlag, anpassen)

| Szenario | RTO (Wiederanlaufzeit) | RPO (max. Datenverlust) |
|---|---|---|
| VM/CT mit HA | ca. 5 Min. (automatisch) | 0 (Daten liegen auf Shared Storage) |
| VM/CT aus Backup | 1 Std. | 24 Std. (tägliches Backup) |
| Kompletter Storage-Server | 1 Arbeitstag | 24 Std. |

## Notfallszenarien

### Ausfall einer VM/Container

**Erkennung:** CheckMK meldet DOWN oder CRIT, oder der Dienst ist nicht erreichbar.

1. Zustand prüfen:
```bash
   qm list; pct list
   ha-manager status
```
2. Gast läuft nicht und ist eine HA-Ressource: HA startet ihn neu. Wenn nicht, den Task-Log in der GUI prüfen und manuell starten (`qm start <ID>` / `pct start <ID>`).
3. Gast defekt oder Daten beschädigt: aus dem letzten Backup wiederherstellen.
```bash
   # Container
   pct restore <CTID> /mnt/pve/nfs-backup/dump/<vzdump-lxc-...tar.zst> --storage nfs-vm-storage --force
   # VM
   qmrestore /mnt/pve/nfs-backup/dump/<vzdump-qemu-...vma.zst> <VMID> --storage nfs-vm-storage --force
```
4. Danach HA-Status und Funktion des Dienstes prüfen.

### Ausfall eines Proxmox-Nodes

**Erkennung:** CheckMK-Host DOWN, in der GUI ein Node rot, `pvecm status` zeigt nur noch einen Node.

**Verhalten:** Der verbleibende Node behält das Quorum, weil das QDevice die dritte Stimme liefert. Der HA-Manager fenced den ausgefallenen Node und startet die HA-Ressourcen auf dem überlebenden Node. Beobachtet am 02.10.2026: ca. 4 Minuten vom Link-Ausfall bis zum Neustart von CT 100.

1. Lage prüfen:
```bash
   pvecm status          # Quorum? QDevice sichtbar?
   ha-manager status     # Ressourcen auf dem überlebenden Node?
```
2. Gäste ohne HA starten nicht automatisch. Ist der ausgefallene Node sicher aus, die Konfigurationen auf den überlebenden Node verschieben und starten:
```bash
   mv /etc/pve/nodes/<ausgefallen>/qemu-server/<ID>.conf /etc/pve/nodes/<überlebend>/qemu-server/
   mv /etc/pve/nodes/<ausgefallen>/lxc/<ID>.conf /etc/pve/nodes/<überlebend>/lxc/
```
   **Achtung:** Das nur tun, wenn der ausgefallene Node wirklich aus ist. Läuft er noch, würde der Gast zweimal auf demselben Storage starten.
3. Ursache am ausgefallenen Node klären: Strom, Netzwerk-Link, Konsole, danach `journalctl -b -1` auf dem Node.
4. Node wieder starten. Er tritt dem Cluster automatisch wieder bei. Danach `pvecm status` und `ha-manager status` prüfen.
5. **Node muss neu installiert werden:**
   - Installation mit `answer.toml` (Auto-Installer).
   - Repos und Updates: `ansible-playbook proxmox_repos.yml`.
   - Alten Node aus dem Cluster entfernen: auf dem überlebenden Node `pvecm delnode <name>`.
   - Neuen Node beitreten lassen: `pvecm add 192.168.88.231`.
   - QDevice auf dem neuen Node prüfen (`corosync-qdevice` installiert, `pvecm status` zeigt das Qdevice).
   - Storage-Konfiguration ist clusterweit und kommt automatisch mit.
   - Monitoring: `ansible-playbook checkmk_agent.yml`.

### Ausfall mehrerer Proxmox-Nodes

**Erkennung:** Beide Nodes nicht erreichbar, alle Gäste down (der Ausfall von Switch oder Strom sieht genauso aus, daher zuerst die Ursache prüfen).

1. Ursache klären: Strom (USV, Steckdosenleiste), Switch, Kabel. Wenn die Ursache der Switch ist, dem Szenario "Ausfall des Switches" folgen.
2. Reihenfolge einhalten: Netzwerk, Storage-Server, dann die Nodes (siehe Wiederanlaufpriorität).
3. Nodes starten. Ein Node plus QDevice genügt für das Quorum. Danach `pvecm status` prüfen.
4. HA-Ressourcen starten automatisch. Gäste ohne HA manuell starten, CheckMK zuletzt.
5. **Nur im Notfall,** wenn kein Quorum zustande kommt (z. B. Storage-Server mit QDevice ebenfalls aus und nur ein Node verfügbar):
```bash
   pvecm expected 1
```
   Das setzt das Quorum künstlich herab. Sobald der zweite Node oder das QDevice wieder da ist, prüfen, dass das Cluster wieder konsistent ist.
6. **Totalverlust beider Nodes:**
   - Beide Nodes neu installieren (`answer.toml`).
   - Cluster neu aufbauen und QDevice einrichten (`pvecm qdevice setup 192.168.88.240`).
   - Storages einbinden: `ansible-playbook proxmox_storage.yml`.
   - Gäste aus `nfs-backup` wiederherstellen.

### Ausfall des Storage-Servers

**Erkennung:** Alle Storages in der GUI mit Fragezeichen, `pvesm status` hängt. Gäste auf `nfs-vm-storage` bekommen I/O-Fehler oder frieren ein. Zusätzlich fällt das QDevice weg. Das Cluster bleibt mit beiden Nodes quorate, ist aber nicht mehr gegen einen weiteren Ausfall geschützt.

1. **Nodes nicht blind neu starten.** NFS-Hänger lösen sich meist, sobald der Server wieder erreichbar ist.
2. Storage-Server prüfen: Strom, Netzwerk, Konsole.
```bash
   zpool status
   zpool import pveha-sto-pool    # falls der Pool nicht eingebunden ist
   zfs list
   systemctl status nfs-server
   exportfs -v                    # Exports mit no_root_squash für 192.168.88.0/24?
   systemctl status corosync-qnetd
```
3. Auf den Nodes prüfen, ob die Mounts wieder da sind:
```bash
   timeout 5 ls /mnt/pve/nfs-vm-storage /mnt/pve/nfs-backup /mnt/pve/nfs-iso
   pvesm status
```
4. Mount hängt weiter: `umount -f -l /mnt/pve/<storage>`, danach `systemctl restart pvestatd`. Hilft das nicht, den betroffenen Node kontrolliert neu starten (vorher `qm list`/`pct list` prüfen, welche Gäste darauf laufen).
5. Abgestürzte Gäste neu starten. Danach mit `pvecm status` prüfen, ob das QDevice wieder verbunden ist.
6. **Pool oder Server unwiederbringlich verloren:**
   - Ubuntu Server neu installieren, ZFS-Pool `pveha-sto-pool` anlegen.
   - Datasets `backup`, `vm-storage`, `iso` anlegen: `zfs set compression=lz4 atime=off xattr=sa <dataset>`.
   - NFS-Exports setzen (Netz 192.168.88.0/24, `no_root_squash`).
   - QDevice neu einrichten (`corosync-qnetd` installieren, `pvecm qdevice setup 192.168.88.240`).
   - Storages prüfen, bei Bedarf `proxmox_storage.yml`.
   - Gäste aus dem Backup wiederherstellen.
   - **Achtung:** Die Backups liegen auf demselben Server. Ohne externe Kopie (siehe Vorbeugende Maßnahmen) sind sie bei Totalverlust ebenfalls weg.

### Ausfall des Switches / Netzwerks

**Erkennung:** Alle Hosts und Gäste gleichzeitig nicht erreichbar. In den Logs der Nodes stehen `NIC Copper Link is Down` und Corosync-Token-Timeouts.

1. Link-LEDs und Kabel am Switch und an den Nodes prüfen, Switch-Logs und Port-Counter ansehen.
2. Kabel oder Switch-Port tauschen, bei Bedarf den Switch neu starten.
3. Nach der Wiederherstellung prüfen: `pvecm status` (Quorum, QDevice), `ha-manager status`, Mounts mit `pvesm status`.
4. Dauert der Ausfall länger, kann ein isolierter Node per Watchdog neu starten oder gefenced werden. Danach wie beim Ausfall eines Proxmox-Nodes vorgehen.
5. Dokumentieren: Zeitpunkt (Switch-Log, `journalctl -k`, CheckMK) und Ursache. Der Link-Flap vom 02.10.2026 (ca. 09:23 und 09:53 Uhr) ist dafür ein Beispiel.

## Wiederanlaufpriorität

1. **Netzwerk (Switch).** Ohne Netz ist nichts erreichbar und das Cluster kann kein Quorum bilden.
2. **Storage-Server.** Alle VM/CT-Disks und Backups liegen auf dem NFS-Storage. Der Server stellt außerdem das QDevice.
3. **Proxmox-Nodes.** Erst wenn der Storage verfügbar ist, können HA-Gäste sauber starten.
4. **Gäste:** HA-Ressourcen automatisch, übrige manuell.
5. **CheckMK.** Monitoring ist wichtig, aber alles andere läuft auch ohne.

### Checkliste nach dem Wiederanlauf
- [ ] `pvecm status`: Quorum vorhanden, QDevice verbunden
- [ ] `pvesm status`: alle drei Storages aktiv
- [ ] `ha-manager status`: alle Ressourcen `started`
- [ ] Alle Gäste erreichbar (`qm list`, `pct list`)
- [ ] CheckMK: alle Hosts UP, keine offenen CRIT-Meldungen
- [ ] Letzte Backups vorhanden und Backup-Jobs wieder aktiv
- [ ] Vorfall dokumentiert (Zeitpunkt, Ursache, Maßnahmen)

## Vorbeugende Maßnahmen
* Tägliche Backups auf `nfs-backup` (vzdump). Für Container lokales `tmpdir` in `/etc/vzdump.conf` setzen, sonst schlägt das Backup unprivilegierter LXC über NFS fehl.
* Zusätzliche Sicherung der CheckMK-Site per `omd backup`.
* Externe Kopie der Backups (anderer Standort oder zweites Medium) <anpassen>.
* Monitoring mit CheckMK: Agents auf allen Hosts, Proxmox-VE-Special-Agent (Cluster, VMs, HA, Backup-Status), Alarmierung per Mail <anpassen>.
* HA: Alle wichtigen Gäste als HA-Ressource registrieren (`ha-manager add`), Failover regelmäßig testen.
* Updates kontrolliert: Proxmox-Nodes nacheinander (Rolling-Update, Gäste vorher migrieren), nie beide gleichzeitig.
* Restoretests: mindestens vierteljährlich je einen Container und eine VM aus dem Backup wiederherstellen und die Funktion prüfen.
* Infrastruktur als Code: Ansible-Playbooks (`proxmox_repos.yml`, `proxmox_storage.yml`, `checkmk_agent.yml`) und `answer.toml` versioniert aufbewahren.
* Zugangsdaten (Root, Ansible-Vault-Passwort, API-Token) getrennt und offline hinterlegen, nicht in diesem Dokument.
* Dokumentation und Wiederanlaufplan aktuell halten und nach jedem Vorfall überprüfen.

## Kontakte und Zuständigkeiten <anpassen>

| Rolle | Name | Erreichbarkeit |
|---|---|---|
| Verantwortlich | | |
| Vertretung | | |
| Netzwerk/Switch | | |

# Storage-Server Konfiguration

## Namesnauflösung

Der `/etc/hosts` folgende Einträge anfügen:
```
192.168.88.231 pveha1.internal pveha1
192.168.88.232 pveha2.internal pveha2
192.168.88.240 pveha-sto.internal pveha-sto
```

## ZFS installieren
```bash
sudo apt install zfsutils-linux
```

## HDD bereinigen
```bash
lsblk
sudo wipefs -a /dev/sda
```

## ZFS konfigurieren

### Pool
```bash
sudo zpool create pveha-sto-pool /dev/sda
```
### Datasets
```bash
sudo zfs create pveha-sto-pool/backup

sudo zfs set compression=lz4 atime=off xattr=sa pveha-sto-pool/backup
```

```bash
sudo zfs create pveha-sto-pool/pve-storage

sudo zfs set compression=lz4 atime=off xattr=sa recordsize=64K pveha-sto-pool/pve-storage
```

```bash
sudo zfs create pveha-sto-pool/iso

sudo zfs set compression=lz4 atime=off xattr=sa pveha-sto-pool/iso
```

## NFS Shares
In `/etc/exports` schreiben:
```
/pveha-sto-pool/backup      192.168.88.0/24(rw,sync,no_subtree_check,no_root_squash)
/pveha-sto-pool/pve-storage  192.168.88.0/24(rw,sync,no_subtree_check,no_root_squash)
/pveha-sto-pool/iso         192.168.88.0/24(rw,sync,no_subtree_check,no_root_squash)
```
```bash
sudo exportfs -ra
```

## Kontrolle
```bash
zpool status
zfs list
sudo exportfs -v
```
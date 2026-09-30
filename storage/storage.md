# Storage-Server konfiguration

## ZFS installieren
```bash
sudo apt install zfsutils-linux
```

## ZFS konfigurieren

### Pool
```bash
sudo zpool create pveha-sto-pool /dev/sda
```
### Datasets
```bash
sudo zfs create pveha-sto-pool/backup

sudo zfs set compression=lz4 pveha-sto-pool/backup
sudo zfs set atime=off pveha-sto-pool/backup
sudo zfs set xattr=sa pveha-sto-pool/backup
```

```bash
sudo zfs create pveha-sto-pool/pve-storage

sudo zfs set compression=lz4 pveha-sto-pool/pve-storage
sudo zfs set atime=off pveha-sto-pool/pve-storage
sudo zfs set xattr=sa pveha-sto-pool/pve-storage
sudo zfs set recordsize=64K pveha-sto-pool/pve-storage
```

```bash
sudo zfs create pveha-sto-pool/iso

sudo zfs set compression=lz4 pveha-sto-pool/iso
sudo zfs set atime=off pveha-sto-pool/iso
sudo zfs set recordsize=1M pveha-sto-pool/iso
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
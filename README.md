# Verschieben einer bestehenden Ubuntu 16.04-Installation auf ein ZFS-Root-Dateisystem

Diese Anleitung basiert in sehr großen Teilen auf der folgenden Anleitung, wurde aber an Ubuntu 16.04 und für eine bestehende Installation angepasst:

[HOWTO install Ubuntu to a Native ZFS Root Filesystem](https://github.com/zfsonlinux/pkg-zfs/wiki/HOWTO-install-Ubuntu-to-a-Native-ZFS-Root-Filesystem)

## Backup des Dateisystems

Je nach Aufbau des eigenen Systems (z.B. Untergliederung in root und home), muss man zuerst ein Backup mit Hilfe von Tar durchführen. In meinem Fall habe ich die beiden folgenden Backup-Dateien auf meine externe Festplatte erstellt:

```
tar cvpzf /mnt/external_hardisk/root.tar.gz --one-file-system /
tar cvpzf /mnt/external_hardisk/home.tar.gz --one-file-system /home
```

Wichtig ist die Option "one-file-system", welche verhindert, dass Tar über Mount-Grenzen hinweg arbeitet und z.B. "sys", "proc", usw. mitsichert.

## Live-CD herunterladen

Man benötigt eine aktuelle Version einer Ubuntu 16.04 Live-CD

[xenial-desktop-amd64.iso](http://cdimage.ubuntu.com/daily-live/current/xenial-desktop-amd64.iso)

Die Live-CD wird dann je nach Gusto auf einen USB-Stick oder einen DVD-Rohling transferiert.

## Booten und Anpassen der Live-CD

Da die ZFS-Tools im Moment noch in "Universe" liegen und auch noch nicht in der Standard-Installation enthalten sind, muss diese erst in der Datei "sources.list" aktiviert werden (Ich spare mir hier die Anleitung wie das zu bewerkstelligen ist).

Auf jeden Fall muss man das Paket "zfsutils-linux" nachinstallieren:

```
sudo apt-get install zfsutils-linux
```

## Vorbereiten der Festplatte/SSD und anlegen des ZFS-Pools (kurz "zpool")

Da ZFS eine Mischung aus einem LVM und einem Dateisystem ist, muss zuerst ein sogenannter "zpool" (Quasi ein Container für alles andere) angelegt werden. Dieser Pool kann entweder eine komplette Festplatte oder eine einzelne Partition umfassen. Letzteres ist ist auf PCs die Regel, da zumindest auf modernen PCs zumindest eine EFI-Partition vorhanden ist.

Um auf der Live-CD mit ZFS arbeiten zu können, muss als erstes das ZFS-Kernel-Modul geladen werden. Dank der direkten Integration in Ubuntu 16.04, reicht ein "insmod zfs"

```
sudo insmod zfs
```

Jetzt kommt der erste knifflige Teil:

Wie teile ich die Festplatte mit ZFS auf. Bei mir war vor der Umstellung die SSD in drei Partitionen unterteilt:

> sda1 (250M) -> EFI-Boot-Partition

> sda2 (35G) -> Root-Partition

> sda3 (Rest) -> Home-Partition

Da "root" und "home" am Ende in einem gemeinsamen zpool liegen, habe ich meine SSD folgendermaßen umpartioniert:

> sda1 (250) -> EFI-Boot-Partition

> sda2 (Rest) -> Platz für den ZFS-Pool

___Hinweis:___

_Wer noch den normalen Bios-Boot verwendet, sollte eine separate Boot-Partition für Grub2 einrichten und mit Ext2 oder Ext3 formatieren. Siehe dazu auch die oben verlinkte Anleitung._

Die Id des Dateisystems kann dabei auf "_Linux_" bzw. "_82_" belassen werden. Wer noch eine Swap-Partition einsetzt, sollte diese auf _sda2_ legen und _sda3_ zum "_zpool_" machen.

Jetzt wird der eigentliche "_zpool_" angelegt:
```
sudo zpool create -o ashift=9 rpool /dev/disk/by-id/scsi-SATA_disk1-part2
```
Hier ein paar Erklärungen zu den einzelnen Optionen:

_ashift=9_ -> Legt die die Sektorgröße des Pools fest. Wird von der tatsächlichen Sektorgröße des Laufwerks bestimmt. Bei meiner SSD sind es 512 Byte bzw. 2^9 Bytes. Moderne Laufwerke benutzen 4096 Bytes bzw. 2^12 Bytes.

_rpool_ -> Der Name des zpools. Hier ist es empfehlenswert den Namen "_rpool_" für den Hauptpool zu übernehmen.

_/dev/disk/by-id/scsi-SATA_disk1-part2_ -> Ich zitiere hier mal den Originalartikel:

> Always use the long /dev/disk/by-id/* aliases with ZFS. Using the /dev/sd* device nodes directly can cause sporadic import failures, especially on systems that have more than one storage pool.

Ich habe mich daran gehalten, auch wenn es dafür an anderer Stelle ein kleines Problem gibt (Siehe weiter unten)

___Hinweis:___

_Es kann sein, dass man noch die Option "-f" beim Anlegen des zpools benutzen muss, da es sein kann, dass zpool sich wegen eines bestehenden vorhandenen Dateisystems weigert den Pool anzulegen._

## Anlegen der Datasets im zpool

Jetzt werden die eigentlichen Datasets bzw. Dateisysteme im zpool angelegt. Dabei ist es sinnvoll auch eine Hierarchie einzuhalten und nicht direkt das erste angelegte Dataset einzubinden. Ich habe mich hierbei ebenfalls an der obigen Anleitung orientiert:

```
sudo zfs create rpool/ROOT/
sudo zfs create rpool/ROOT/ubuntu-1
```

Wer seine Homeverzeichnisse in ein extra Dataset legen will (Zwecks besserer Trennung), legt diese entsprechend an:

```
sudo zfs create rpool/HOME
sudo zfs create rpool/HOME/home-1
```
ZFS mountet nach dem Anlegen der Datasets diese direkt unter "/mnt/ROOT" und "/mnt/ROOT/ubuntu-1", usw. Das ist natürlich unschön, für ein normales Produktivsystem. Man muss also den Mountpoint der einzelnen Datasets ändern.

Zuerst werden alle eingebundenen Dateisysteme wieder ausgehängt:

```
sudo zfs umount -a
```

Danach setzt man für jedes Dataset gesondert seinen Mountpoint:

```
sudo zfs set mountpoint=none rpool/ROOT
sudo zfs set mountpoint=/ rpool/ROOT/ubuntu-1
sudo zfs set mountpoint=none rpool/HOME
sudo zfs set mountpoint=/home rpool/HOME/home-1
```

Wichtig sind vor allem die beiden Befehle mit "mountpoint=none". Mit diesen wird verhindert, dass die beiden untersten Datasets gemountet werden.

__Hinweis:__

_Ich habe bewusst keine Größe der einzelnen Datasets angegeben. Der Platz wird automatisch zwischen allen Datasets aufgeteilt. Wer hier feinere Einstellungen vornehmen will, sei an die offizielle Dokumentation von ZFS verwiesen:_

[Setting ZFS Quotas and Reservations](https://docs.oracle.com/cd/E23823_01/html/819-5461/gazvb.html)

## Fertigstellen der ZFS-Umgebung

Als letzten Schritt vor dem Einspielen des Backups müssen noch drei Dinge getan werden:

1. Dem Bootloader mitteilen wo das root-Dateisystem liegt:
```
zpool set bootfs=rpool/ROOT/ubuntu-1 rpool
```

Die Angaben müssen dabei exakt den Namen des angelegten Pools und des Datasets entsprechen.

2. Den Pool exportieren

Ich zitiere hier wieder die Anleitung:

> Don't skip this step. The system is put into an inconsistent state if this command fails or if you reboot at this point.

```
sudo zpool export rpool
```

## Wiedereinspielen des Systems

Nachdem ZFS nach unseren Wünschen eingerichtet ist, müssen wir es wieder in den Verzeichnisbaum einbinden.

Dazu muss der Pool erst importiert werden:

```
sudo zpool import -R /mnt rpool
```

Der Verzeichnisbaum sollte beim Aufruf von "_mount_" in etwa dann so ausschauen (Falls man die Anleitung exakt ausgeführt hat):

```
rpool/ROOT/ubuntu-1 on /mnt type zfs (rw,relatime,xattr,noacl)
rpool/HOME/home-1 on /mnt/home type zfs (rw,relatime,xattr,noacl)
```

Als nächstes wechselt man nach "/mnt" und spielt sein Backup mit Hilfe von "_tar_" wieder ein:

```
cd /mnt
sudo tar zxvpf /Mountpoint_zur_externen_Platte/root.tar.gz
sudo tar zxvpf /Mountpoint_zur_externen_Platte/home.tar.gz
```

Als nächstes erstellt man eine chroot-Umgebung und wechselt dann in diese:

```
mount --bind /dev  /mnt/dev
mount --bind /dev/pts  /mnt/dev/pts
mount --bind /proc /mnt/proc
mount --bind /sys  /mnt/sys
chroot /mnt /bin/bash --login
```

Wer das ältere BIOS-Booten verwendet, sollte an dieser Stelle die Grub2-Boot-Partition in den Verzeichnisbaum einbinden:

```
sudo mount /dev/sda1 /boot/grub
```


In der _chroot_-Umgebung installiert man als allererstes die beiden benötigten ZFS-Pakete

```
sudo apt-get install zfsutils-linux zfs-initramfs
```

Danach geht es an die fstab-Datei. Da ZFS selbst seine Mountpoints verwaltet ist die Datei nur noch Nicht-ZFS-Dateisysteme notwendig. In meinem Fall ist diese minimal, da sie nur noch den Mountpoint für die EFI-Partition enthält:

```
UUID=9924-89D8  /boot/efi       vfat    umask=0077      0       1
```

Analog verfährt man bei dem BIOS-Boot.

Alle anderen Mountpoints sollte man zumindest auskommentieren und erst komplett entfernen, wenn das System auch wirklich bootet.

## Einrichten von Grub2

Der letzte Schritt um das System bootfähig zu machen, ist Grub2 neu einzurichten:

Dazu ruft man als erstes "grub-probe" auf:

```
grub-probe /
```

Hier kann es zu einer Fehlermeldung ähnlich der folgenden kommen:

```
grub-probe: error: failed to get canonical path of `/dev/ata-OCZ_VERTEX-PLUS_8DT5JS4Z24GS69R9D00O-part2'.
```

Wer genauer hinschaut, erkennt gleich den Fehler. Normalerweise liegen die Gerätedateien mit dem vollständigen Namen unter "/dev/disk/by-id", Grub2 sucht aber direkt unter "/dev" danach. Ein schneller Workaround um den Fehler zu beseitigen, sieht dann in etwa so aus:

```
sudo ln -s /dev/sdXY /dev/Name_des_Geräts_aus_der_Fehlermeldung
```

"_/dev/sdaXY_" wird dabei gegen die Gerätedatei ersetzt, die dem Partitionseintrag des Pools entspricht (Bei mir "_/dev/sda2_").

Nach Anlegen des Symlinks sollte "_grub-probe /_" das Wort "_ZFS_" ausspucken.

Als nächstes aktualisiert man seine Initrd und die Grub-Konfiguration:

```
sudo update-initramfs -c -k all
sudo update-grub
```

Als nächstes sollte man überprüfen ob in Grub2 die passenden ZFS-Parameter gesetzt wurden:

```
grep root=ZFS /boot/grub/grub.cfg
```

Gibt es hier eine Ausgabe ähnlich der folgenden, passt alles:

```
linux	/ROOT/ubuntu-1@/boot/vmlinuz-4.4.0-4-generic root=ZFS=rpool/ROOT/ubuntu-1 ro  quiet splash $vt_handoff
```

Im Grunde ist man jetzt schon fertig mit dem Installieren und einrichten. Es fehlen jetzt noch zwei Dinge:

1. Kopieren der angelegten "zpool.cache"-Datei
2. Sauberes unmounten aller Dateisysteme und exportieren des angelegten zpools.

zu 1.: Ich weiß nicht ob der Schritt wirklich notwendig ist, geschadet hat er bei mir nicht. Gibt folgende Befehle auf der Konsole ein:

```
exit
sudo cp /etc/zfs/zpool.cache /mnt/etc/zfs/
```

Das "_exit_" verlässt dabei die chroot-Umgebung.

zu 2.:

```
sudo umount /dev/pts
sudo umount /dev/
sudo umount /proc
sudo umount /sys
sudo umount /mnt/home
sudo umount /mnt/boot/efi
sudo zpool export -a
```

Danach kann man das System versuchen das erste Mal zu starten.

## Probleme beim Booten (Bei UEFI-Installation)

### Grub lädt keine Konfiguration

Es kann beim ersten Booten vorkommen, dass Grub seine Konfiguration nicht laden kann und nur seine Kommandozeile präsentiert. Das ist aber kein Grund zur Panik, da man ohne Probleme das System mit Hilfe dieser starten kann. Dank _Tab-Completion_ spart man auch eine Menge Tipparbeit

```
insmod zfs
root=(hdX,Y) # Hier muss die passende Partition eingesetzt werden. Bei mir ist es z.B. (hd6,gpt2)
linux /ROOT/.../.../.../boot/vmlinux... root=ZFS=rpool/ROOT/ubuntu-1 ro # Hier müssen hinter "ZFS=" die richtigen Angaben stehen
initrd /ROOT/.../.../.../boot/initrd...
boot
```

Danach sollte Grub das System problemlos starten. Wenn das System hochgefahren ist, sollte man sofort seine Grub-Konfiguration mittels "_update-grub_" aktualisieren. Wahrscheinlich muss der Symlink von weiter oben nochmal gesetzt werden.

### System bleibt mit einer Busybox-Shell stehen

Es kann sein, dass es beim ersten Start aus irgendwelchen Gründen in einer Busybox-Shell stehen bleibt. Das liegt wahrscheinlich daran, dass man vergessen hat den Pool am Ende zu exportieren oder die Grub-Konfiguration wurde nicht richtig aktualisiert. Jedenfalls bleibt einem nichts anderes übrig als nochmals die Live-CD zu benutzen und die chroot-Umgebung nochmals zu erstellen. Daten sind dabei aber nicht verloren gegangen.


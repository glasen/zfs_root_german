# Verschieben einer bestehenden Ubuntu 16.04-Installation auf ein ZFS-Root-Dateisystem

Diese Anleitung basiert in sehr großen Teilen auf der folgenden Anleitung, wurde aber an Ubuntu 16.04 und für eine bestehende Installation angepasst:

[HOWTO install Ubuntu to a Native ZFS Root Filesystem](https://github.com/zfsonlinux/pkg-zfs/wiki/HOWTO-install-Ubuntu-to-a-Native-ZFS-Root-Filesystem)

## Backup des Dateisystems

Je nach Aufbau des eigenen Systems (z.B. Untergliederung in root und home), muss man zuerst ein Backup mit Hilfe von Tar durchführen. In meinem Fall habe ich die beiden folgenden Backup-Dateien auf meiner externen Festplatte erstellt:

```
tar cvpzf /mnt/external_hardisk/root.tar.gz --one-file-system /
tar cvpzf /mnt/external_hardisk/home.tar.gz --one-file-system /home
```

Wichtig ist die Option "one-file-system", welche verhindert, dass Tar über Dateisystemgrenzen hinweg arbeitet und z.B. "sys", "proc", usw. mitsichert.

## Live-CD herunterladen

Zusätzlich benötigt man eine aktuelle Version der _Ubuntu 16.04_-Live-CD

[xenial-desktop-amd64.iso](http://cdimage.ubuntu.com/daily-live/current/xenial-desktop-amd64.iso)

Die Live-CD wird dann je nach Gusto auf einen USB-Stick oder einen DVD-Rohling transferiert.

## Booten und Anpassen der Live-CD

Da die ZFS-Tools im Moment noch in "Universe" liegen und auch noch nicht in der Standard-Installation enthalten sind, muss diese Paketquelle erst in der Datei "sources.list" aktiviert werden (Ich spare mir hier die Anleitung wie das zu bewerkstelligen ist).

Auf jeden Fall muss man das Paket "zfsutils-linux" nachinstallieren:

```
sudo apt-get install zfsutils-linux
```

## Vorbereiten der Festplatte/SSD und anlegen des ZFS-Pools (kurz "zpool")

Da ZFS eine Mischung aus einem LVM und einem Dateisystem ist, muss zuerst ein sogenannter "zpool" (Quasi ein Container für alles andere) angelegt werden. Dieser Pool kann entweder eine komplette Festplatte oder eine einzelne Partition umfassen. Letzteres ist auf PCs die Regel, da zumindest auf modernen PCs zumindest eine EFI-Partition vorhanden ist.

Um auf der Live-CD mit ZFS arbeiten zu können, muss als erstes das ZFS-Kernel-Modul geladen werden. Dank der direkten Integration in Ubuntu 16.04, reicht ein "insmod zfs"

```
sudo insmod zfs
```

Jetzt kommt der erste knifflige Teil:

Wie teile ich die Festplatte mit ZFS auf? Bei mir war vor der Umstellung die SSD in drei Partitionen unterteilt:

> sda1 (250M) -> EFI-Boot-Partition

> sda2 (35G) -> Root-Partition

> sda3 (Rest) -> Home-Partition

Da "root" und "home" am Ende in einem gemeinsamen zpool liegen, habe ich meine SSD folgendermaßen umpartioniert:

> sda1 (250MB) -> EFI-Boot-Partition

> sda2 (Rest) -> Platz für den ZFS-Pool

___Hinweis:___

_Wer noch den normalen Bios-Boot verwendet, sollte eine separate Boot-Partition für Grub2 einrichten und mit Ext2 oder Ext3 formatieren. Siehe dazu auch die oben verlinkte Anleitung._

Die Id des Dateisystems kann dabei auf "_Linux_" bzw. "_82_" belassen werden. Wer noch eine Swap-Partition benötigt, kann diese in ein sogenanntes "_zvol_" legen (Siehe weiter unten).

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

### Normale Laufwerke

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

### Swap-Partition in einem ZVOL

ZFS bietet auch die Möglichkeit anstatt eines ZFS-Dateisystems eine Gerätedatei bereitzustellen, welche dann ein beliebiges Dateisystem beinhalten kann. Ein Beispiel ist z.B. eine SWAP-Partition.

```
sudo zfs create -V 16G rpool/ROOT/swap
```

Dieses Dateisystem taucht nach dem Anlegen als Gerätedatei unter "_/dev_" auf:

```
lrwxrwxrwx 1 root root 12 Feb 20 19:23 /dev/zvol/rpool/ROOT/swap -> ../../../zd0
```
Diese Gerätedatei kann man jetzt wie gewohnt mit "_mkswap_" on "_swapon_" behandeln:
```
sudo mkswap /dev/zvol/rpool/ROOT/swap
sudo swapon /dev/zvol/rpool/ROOT/swap
```
Die Swap-Partition __muss__ in die _fstab_-Datei aufgenommen werden:

```
UUID=9a328c8d-b915-4ef7-8224-5b05a66a6d79       none    swap    sw      0       0
```
Ich habe die UUID benutzt. Diese wird von "_mkswap_" beim Erzeugen des Swap-Bereichs angezeigt.

### Optional: Einschalten der Kompression und deaktivieren der "_Access Time_"

Wer möchte kann die transparente Kompression in ZFS aktivieren. Dies kostet auf modernen Rechnern fast keine Rechenzeit und spart je nach Inhalt des Dateisystems eine Menge Platz:

```
sudo zfs set compression=lz4 rpool
```

Dieser Befehl aktiviert für alle Datasets innerhalb von "rpool" die Kompression. Wer das nicht möchte, kann sie auch für die einzelnen Datasets ändern (Dabei greift das Prinzip der Vererbung).

Um Schreibzugriffe zu beschleunigen, empfiehlt es sich die "Access Time"-Option des Dateisystems zu deaktivieren. Diese Option entspricht in etwa der Option "relatime" bei Ext2/3/4:

```
sudo zfs set atime=off rpool
```

__Fortsetzung:__

ZFS mountet nach dem Anlegen der Datasets diese direkt unter "/mnt/ROOT" und "/mnt/ROOT/ubuntu-1", usw. Das ist natürlich für ein normales Produktivsystem nicht zu gebrauchen. Man muss also den Mountpoint der einzelnen Datasets ändern.

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

Wichtig sind vor allem die beiden Befehle mit "mountpoint=none". Mit diesen wird verhindert, dass die beiden untersten Datasets gemountet werden. "_zfs_" wird beim Setzen der Mountpoints "_/_" mit ziemlicher Sicherheit eine Warnung ausgeben, dass die gesetzten Mountpoints schon existieren bzw. nicht leer sind. Das ist in Ordnung und kann an dieser Stelle ignoriert werden.

__Hinweis:__

_Ich habe bewusst keine Größe der einzelnen Datasets angegeben. Der Platz wird automatisch zwischen allen Datasets aufgeteilt. Wer hier feinere Einstellungen vornehmen will, sei an die offizielle Dokumentation von ZFS verwiesen:_

[Setting ZFS Quotas and Reservations](https://docs.oracle.com/cd/E23823_01/html/819-5461/gazvb.html)

## Fertigstellen der ZFS-Umgebung

Als letzten Schritt vor dem Einspielen des Backups müssen noch drei Dinge getan werden:

1. Dem Bootloader mitteilen wo das root-Dateisystem liegt:
```
zpool set bootfs=rpool/ROOT/ubuntu-1 rpool
```

_Das scheint nicht unbedingt notwendig zu sein, schaden tut es aber auch nicht._

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

Die Option "_-R_" legt dabei den untersten Mountpoint des gesamten Pools fest. Hätten wir weiter oben die Mountpoints der Datasets nicht geändert, würde jetzt "_rpool_" unter "_/mnt_" liegen, "_rpool/ROOT_" unter "_/mnt/rpool/ROOT_", usw. Dadurch das wir die Mountpoints verändert haben, wird nur noch "_rpool/ROOT/ubuntu-1_" in "_/mnt_" eingebunden. Andere Datasets werden dabei ebenfalls eingebunden (z.B. "_rpool/HOME/home-1_" in "_/mnt/home_").

Der Verzeichnisbaum sollte beim Aufruf von "_mount_" dann in etwa so ausschauen (Falls man die Anleitung exakt ausgeführt hat):

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
for i in /dev /dev/pts /proc /sys; do sudo mount -B $i /mnt$i; done
sudo chroot /mnt /bin/bash --login
```
__Hinweis__:

_Ab jetzt arbeitet man als "_root_" in der chroot-Umgebung (Zu sehen am Prompt)!_

Als nächstes bindet man jetzt die EFI-Partition in den Verzeichnisbaum ein. Falls das Verzeichnis "_efi_" noch nicht existiert, muss man es vor dem Einbinden anlegen:
```
mkdir /boot/efi
mount /dev/sda1 /boot/efi
```

Wer das ältere BIOS-Boot verwendet, sollte an dieser Stelle die Grub2-Boot-Partition in den Verzeichnisbaum einbinden:

```
mount /dev/sda1 /boot/grub
```
Falls Dateien unter "_boot/grub_" vorhanden sind (Weil man vorher keine eigene Boot-Partition benutzt hat), sollte man diese Befehle benutzen:
```
mv /boot/grub /boot/grub.bak
mount /dev/sda1 /boot/grub
mv /boot/grub.bak/* /boot/grub/
rm -r /boot/grub.bak
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

Wer weiter oben eine Swap-Partition angelegt hat, muss diese ebenfalls in der _fstab_-Datei aufnehmen.

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
Normalerweise liegen die Gerätedateien mit dem vollständigen Namen unter "_/dev/disk/by-id_", Grub2 sucht aber direkt unter "_/dev_" danach. Ein schneller Workaround um den Fehler zu beseitigen, sieht dann in etwa so aus:

```
sudo ln -s /dev/sdXY /dev/Name_des_Geräts_aus_der_Fehlermeldung
```

"_/dev/sdXY_" wird dabei gegen die Gerätedatei ersetzt, die dem Partitionseintrag des Pools entspricht (Bei mir "_/dev/sda2_").

Nach Anlegen des Symlinks sollte "_grub-probe /_" das Wort "_ZFS_" ausspucken.

Um den Fehler dauerhaft im installierten System zu beseitigen, empfiehlt es sich die folgende Datei nach "_/etc/udev/rules.d_" bzw. "_/mnt/etc/udev/rules.d_" zu kopieren:
[60-zpool.rules](https://launchpadlibrarian.net/237062391/60-zpool.rules)

```
# This creates symlinks directly in /dev with the same names as those in
# /dev/disk/by-id, but only for ZFS partitions.  This is necessary for GRUB to
# locate the partitions using the output of `zpool status`.
KERNEL=="sd*[0-9]", IMPORT{parent}=="ID_*", ENV{ID_PART_ENTRY_SCHEME}=="gpt", ENV{ID_PART_ENTRY_TYPE}=="6a898cc3-1dd2-11b2-99a6-080020736631", SYMLINK+="$env{ID_BUS}-$env{ID_SERIAL}-part%n"
```


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

Im Grunde ist man jetzt schon fertig mit dem Installieren und einrichten. Es fehlt jetzt nur noch eine wichtige Sache:

Sauberes unmounten aller Dateisysteme und exportieren des angelegten zpools.

```
sudo umount /dev/pts
sudo umount /dev/
sudo umount /proc
sudo umount /sys
sudo umount /mnt/home
sudo umount /mnt/boot/efi
sudo zpool export -a
```

Wichtig ist vor allem das Exportieren des/der Pool(s) damit diese nicht in einem inkonsistenten Zustand sind.

Danach kann man das System versuchen das erste Mal zu starten.

## Probleme beim Booten (Bei UEFI-Installation)

### Grub lädt keine Konfiguration

Es kann beim ersten Booten vorkommen, dass Grub seine Konfiguration nicht laden kann und nur seine Kommandozeile präsentiert. Gerade bei der Installation zum Testen unter VirtualBox passiert mir das immer, da "_grub-probe_" grundsätzlich nicht funktioniert (Er findet in der chroot-Umgebung die Gerätedatei "_/dev/zfs_" nicht). Das ist aber kein Grund zur Panik, da man ohne Probleme das System mit Hilfe dieser starten kann. Dank _Tab-Completion_ spart man auch eine Menge Tipparbeit (Die "_...._" stehen hier für die TAB-Taste):

```
insmod zfs
root=(hdX,Y) # Hier muss die passende Partition eingesetzt werden. Bei mir ist es z.B. (hd6,gpt2)
linux /ROOT/.../.../.../boot/vmlinux... root=ZFS=rpool/ROOT/ubuntu-1 ro # Hier müssen hinter "ZFS=" die richtigen Angaben stehen
initrd /ROOT/.../.../.../boot/initrd...
boot
```

Danach sollte Grub das System problemlos starten. Wenn das System hochgefahren ist, sollte man sofort seine Grub-Konfiguration mittels "_update-grub_" aktualisieren.

### System bleibt mit einer Busybox-Shell stehen

Es kann sein, dass der Boot beim ersten Start aus irgendwelchen Gründen in einer Busybox-Shell stehen bleibt. Das liegt wahrscheinlich daran, dass man vergessen hat den Pool am Ende zu exportieren oder die Grub-Konfiguration wurde nicht richtig aktualisiert. Jedenfalls bleibt einem nichts anderes übrig als nochmals die Live-CD zu benutzen und die chroot-Umgebung nochmals zu erstellen und die Anleitung ab dem Importieren des Pools nochmals abzuarbeiten. Dieser Fehler ist bei mir nur bei den ersten Versuchen mit ZFS aufgetreten.
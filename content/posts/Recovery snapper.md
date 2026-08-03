---
title: Restaurar snapshot en Arch Linux
description: Recuperar snapshot creada con snapper en Arch Linux
date: 2026-07-28 08:00:00 +0100
categories:
   - Arch
tags:
   - Admin
weight: 1
---

### Snapshots con Arch linux

Durante la instalación de Arch el sistema preguanta si deseamos hacer snapshots. En mi caso elegí hacer snapshots con snapper.  


### Instalación y configuración de snapper 

``` bash
sudo pacman -S snapper snap-pac
```

- snapper: la herramienta en sí. (ya debería estar instalada)
- snap-pac: hook de pacman que crea automáticamente un snapshot antes y después de cada pacman -Syu (esto es lo que responde a tu pregunta de "que se hagan las snapshots con cada update").   

Mi sistema ya tiene un layout de subvolúmenes tipo el recomendado por el wiki de Arch (@, @home, @log, @pkg, @snapshots), con @snapshots como subvolumen independiente en vez de dejar que snapper cree .snapshots como subvolumen anidado dentro de @.   

Y además @home/.snapshots indica que ya tienes (o tenías) un config home de snapper también activo.  

#### Prerarar el subvolumen @snapshots

Vamos a dejar el subvolumen @snapshots aparte del subovolumen @.

```bash
# Crear el config (snapper generará /.snapshots como subvolumen anidado)
sudo snapper -c root create-config /

# Borrar ese subvolumen anidado que acaba de crear
sudo btrfs subvolume delete /.snapshots

# Recrear el punto de montaje como directorio normal
sudo mkdir /.snapshots
```

Añadimos la entrada a /etc/fstab:

```bash
# /.snapshots
UUID=21edca7e-d04b-4672-8f0d-45663cb2c90d  	/.snapshots  	btrfs  		subvol=@snapshots,compress=zstd,defaults  			0  0
```

Montamos:
```bash
sudo mount /.snapshots
```

Verificamos:
```bash
findmnt /.snapshots

# salida:
TARGET      SOURCE                        FSTYPE OPTIONS
/.snapshots /dev/mapper/root[/@snapshots] btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,subvolid=559,subvol=/@snapshots
```

Permisos:
```bash
sudo chmod 750 /.snapshots
sudo chown :wheel /.snapshots
```

En /etc/snapper/configs/ tenemos dos ficheros de configuración: home y root. Estos ficheros tiene la configuración por defecto:

```bash
[noe@envy ~]% cat /etc/snapper/configs/root 

# subvolume to snapshot
SUBVOLUME="/"

# filesystem type
FSTYPE="btrfs"


# btrfs qgroup for space aware cleanup algorithms
QGROUP=""


# fraction or absolute size of the filesystems space the snapshots may use
SPACE_LIMIT="0.5"

# fraction or absolute size of the filesystems space that should be free
FREE_LIMIT="0.2"


# users and groups allowed to work with config
ALLOW_USERS=""
ALLOW_GROUPS=""

# sync users and groups from ALLOW_USERS and ALLOW_GROUPS to .snapshots
# directory
SYNC_ACL="no"


# start comparing pre- and post-snapshot in background after creating
# post-snapshot
BACKGROUND_COMPARISON="yes"


# run daily number cleanup
NUMBER_CLEANUP="yes"

# limit for number cleanup
NUMBER_MIN_AGE="3600"
NUMBER_LIMIT="50"
NUMBER_LIMIT_IMPORTANT="10"


# create hourly snapshots
TIMELINE_CREATE="yes"

# cleanup hourly snapshots after some time
TIMELINE_CLEANUP="yes"

# limits for timeline cleanup
TIMELINE_MIN_AGE="3600"
TIMELINE_LIMIT_HOURLY="10"
TIMELINE_LIMIT_DAILY="10"
TIMELINE_LIMIT_WEEKLY="0"
TIMELINE_LIMIT_MONTHLY="10"
TIMELINE_LIMIT_QUARTERLY="0"
TIMELINE_LIMIT_YEARLY="10"


# cleanup empty pre-post-pairs
EMPTY_PRE_POST_CLEANUP="yes"

# limits for empty pre-post-pair cleanup
EMPTY_PRE_POST_MIN_AGE="3600"
```

#### Timers de limpieza automatica:
Los timers realizan la limpieza según la política establecida en el fichero de configuración: **TIMELINE_LIMIT_**.

```bash
sudo systemctl enable --now snapper-timeline.timer
sudo systemctl enable --now snapper-cleanup.timer
```

### Recuperar snapshot

Arrancamos desde nuestro usb de Arch.

```bash
# Cargamos teclado en español
loadkeys es

# Aumentamos la fuente porque la resolución de nuestra pantalla nos deja ciegos
setfont ter-132n
```

Identificamos particiones del sistema:
```bash
lsblk -f
# Debería verse algo como nvme0n1p2 con TYPE=crypto_LUKS

# Desbloquear el contenedor LUKS
sudo cryptsetup open /dev/nvme0n1p2 root
# Nos pedirá nuestra contraseña de desencriptado de disco

# Esto crea /dev/mapper/root — montar el nivel superior de Btrfs
sudo mkdir -p /mnt/btrfs-root
sudo mount -o subvolid=5 /dev/mapper/root /mnt/btrfs-root
```

Identificamos nuestra snapshot:

```bash
cd /mnt/btrfs-root
ls @snapshots/
cat @snapshots/N/info.xml   # identificar la snapshot buena


sudo mv @ @broken_$(date +%Y%m%d)
sudo btrfs subvolume snapshot @snapshots/N/snapshot @
```

Montamos el nuevo @ y preparamos el chroot:
```bash
sudo umount /mnt/btrfs-root
sudo mount -o subvol=@ /dev/mapper/root /mnt

sudo mount -o subvol=@home /dev/mapper/root /mnt/home
sudo mount -o subvol=@log  /dev/mapper/root /mnt/var/log
sudo mount -o subvol=@pkg  /dev/mapper/root /mnt/var/cache/pacman/pkg
sudo mount -o subvol=@snapshots /dev/mapper/root /mnt/.snapshots

# /boot NO está en el contenedor LUKS, es una partición aparte sin cifrar
sudo mount /dev/nvme0n1p1 /mnt/boot

sudo arch-chroot /mnt
```

#### Configuración de LUKS dentro del chroot

En mi sistema uso UKI (Unified Kernel Image). No hay archivos .conf en boot/loader/entries/ porque no el modelo clásico de systemd-boot (kernel + initramfs + entrada separada), sino que arch arranca mediante una UKI (Unified Kernel Image): un único archivo .efi que lleva empaquetados el kernel, el initramfs y la línea de comandos (cmdline) todo junto. 
Por eso /proc/cmdline tiene contenido pero no hay ningún .conf que lo defina — viene incrustado dentro de la propia imagen UKI.

```bash
cat /etc/kernel/cmdline

# salida: 
cryptdevice=PARTUUID=f20486ea-cc6d-4645-b109-ef06ff0d0cd9:root root=/dev/mapper/root zswap.enabled=0 rootflags=subvol=@ rw rootfstype=btrfs quiet splash
```

El preset de mkinitcpio que genera la UKI, para confirmar la ruta de salida en la ESP:
```bash
cat /etc/mkinitcpio.d/linux.preset

# vemos el fichero por defecto de UKI:

default_uki="/boot/EFI/Linux/arch-linux.efi"
```

El hook encrypt/sd-encrypt:

```bash
grep -E 'HOOKS|encrypt|sd-encrypt' /etc/mkinitcpio.conf
```


Regenerar la UKI: Esto reconstruye la UKI completa (kernel + initramfs + cmdline) y la coloca directamente en la ruta de la ESP definida en el preset — no hay paso intermedio de "actualizar bootloader" porque systemd-boot simplemente detecta el .efi presente.
```bash
mkinitcpio -P
```

Salimos del chroot y reiniciamos
```bash
exit
sudo umount -R /mnt
sudo cryptsetup close root
reboot
```

### Verificaciones post-restauración

Una vez arranca el sistema tras la restauración, conviene comprobar que arrancó desde el snapshoot que debería, que los subvolúmenes están bien montados, que snapper sigue operativo, y que no quedó nada a medias del proceso de rescate.  

- Confirmar que arranca desde el @ restaurado
```bash
findmnt /
```
Debería mostrar /dev/mapper/root[/@]. Si por error arranca con el @broken_*, aquí debería salir.  

- Verificar que todos los subvolúmenes están montados donde toca
```bash
[noe@envy ~]% findmnt -t btrfs
TARGET SOURCE              FSTYPE OPTIONS
/      /dev/mapper/root[/@]
│                          btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,sub
├─/.snapshots
│      /dev/mapper/root[/@snapshots]
│                          btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,sub
├─/var/log
│      /dev/mapper/root[/@log]
│                          btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,sub
├─/var/cache/pacman/pkg
│      /dev/mapper/root[/@pkg]
│                          btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,sub
└─/home
       /dev/mapper/root[/@home]
                           btrfs  rw,relatime,compress=zstd:3,ssd,space_cache=v2,sub
```

Comprobar que aparecen /, /home, /var/log, /var/cache/pacman/pkg y /.snapshots, cada uno con el subvol=@... correcto.  
Un fallo típico tras una restauración manual es olvidar una línea de /etc/fstab o que apunte al subvolumen equivocado.  

- Comprobar que snapper reconoce el histórico correctamente
```bash
[noe@envy ~]% sudo snapper -c root list
[sudo] password for noe: 
# │ Type   │ Pre # │ Date                             │ User │ Cleanup  │ Description                                                              │ Userdata
──┼────────┼───────┼──────────────────────────────────┼──────┼──────────┼──────────────────────────────────────────────────────────────────────────┼─────────
0 │ single │       │                                  │ root │          │ current                                                                  │
1 │ single │       │ Mon 27 Jul 2026 11:04:42 AM CEST │ root │          │ post-reorg baseline                                                      │
2 │ pre    │       │ Mon 27 Jul 2026 11:16:56 AM CEST │ root │ number   │ pacman -Syu                                                              │
3 │ post   │     2 │ Mon 27 Jul 2026 11:16:58 AM CEST │ root │ number   │ imagemagick                                                              │
4 │ pre    │       │ Mon 27 Jul 2026 01:37:05 PM CEST │ root │ number   │ pacman -S systemd-ukify                                                  │
5 │ post   │     4 │ Mon 27 Jul 2026 01:37:06 PM CEST │ root │ number   │ python-cffi python-cryptography python-pefile python-pycparser systemd-u │
6 │ single │       │ Mon 27 Jul 2026 02:00:10 PM CEST │ root │ timeline │ timeline                                                                 │
7 │ single │       │ Mon 27 Jul 2026 10:34:28 PM CEST │ root │ timeline │ timeline                                                                 │
8 │ single │       │ Tue 28 Jul 2026 05:00:02 PM CEST │ root │ timeline │ timeline                                                                 │
9 │ single │       │ Wed 29 Jul 2026 10:29:38 PM CEST │ root │ timeline │ timeline                     
```
Debería el listado completo de snapshots (incluida la que acabamos de restaurar, y las posteriores a ella. Si sale vacío o da error, es señal de que /.snapshots no está bien montado sobre @snapshots.  

- Confirmar que el hook de pacman sigue funcionando
```bash
sudo pacman -S --needed algún-paquete-de-prueba 
# Luego lo podemos desinstalar
sudo snapper -c root list | tail -n 4

# Ejemplo:
sudo pacman -S --needed firefox
sudo snapper -c root list | tail -n 4
warning: firefox-153.0-1 is up to date -- skipping
 there is nothing to do
6 │ single │       │ Mon 27 Jul 2026 02:00:10 PM CEST │ root │ timeline │ timeline                                                                 │
7 │ single │       │ Mon 27 Jul 2026 10:34:28 PM CEST │ root │ timeline │ timeline                                                                 │
8 │ single │       │ Tue 28 Jul 2026 05:00:02 PM CEST │ root │ timeline │ timeline                                                                 │
9 │ single │       │ Wed 29 Jul 2026 10:29:38 PM CEST │ root │ timeline │ timeline              
```

- Revisar que los timers siguen habilitados
```bash
systemctl status snapper-timeline.timer snapper-cleanup.timer
● snapper-timeline.timer - Timeline of Snapper Snapshots
     Loaded: loaded (/usr/lib/systemd/system/snapper-timeline.timer; enabled; prese>
     Active: active (waiting) since Tue 2026-07-28 16:20:36 CEST; 1 day 6h ago
 Invocation: 5248951024d44024a2791f3d4dce9751
    Trigger: Wed 2026-07-29 23:00:00 CEST; 20min left
   Triggers: ● snapper-timeline.service
       Docs: man:snapper(8)
             man:snapper-configs(5)

Jul 28 16:20:36 envy systemd[1]: Started Timeline of Snapper Snapshots.

● snapper-cleanup.timer - Hourly Cleanup of Snapper Snapshots
     Loaded: loaded (/usr/lib/systemd/system/snapper-cleanup.timer; enabled; preset>
     Active: active (waiting) since Tue 2026-07-28 16:20:36 CEST; 1 day 6h ago
 Invocation: 6bf7364c0968447e97034bd73279ac5e
    Trigger: Wed 2026-07-29 22:48:30 CEST; 9min left
   Triggers: ● snapper-cleanup.service
       Docs: man:snapper(8)
             man:snapper-configs(5)

Jul 28 16:20:36 envy systemd[1]: Started Hourly Cleanup of Snapper Snapshots.
```

- Comprobar journal y arranque limpio  
Busca errores de montaje, del hook encrypt/sd-encrypt, o de servicios que fallen por dependencias rotas de una restauración a un punto temporal distinto del resto del sistema.  
```bash
journalctl -b -p err
systemctl --failed
```
- Verificar la UKI activa  
Confirma qué entrada UKI se usó para arrancar y que systemd-boot no está mostrando ninguna entrada obsoleta o duplicada.  
```bash
[noe@envy ~]% bootctl status
System:
      Firmware: UEFI 2.50 (INSYDE Corp. 3890.00)
 Firmware Arch: x64
   Secure Boot: disabled
  TPM2 Support: yes
  Measured UKI: yes
   Measured OS: yes
  Boot into FW: supported
 Platform Lang: en_US.UTF-8

Current Boot Loader:
        Product: systemd-boot 261.2-1-arch
       Features: ✓ Boot counting
                 ✓ Menu timeout control
                 ✓ One-shot menu timeout control
                 ✓ Default entry control
                 ✓ One-shot entry control
                 ✓ Support for XBOOTLDR partition
                 ✗ Support for passing random seed to OS
                 ✗ Load drop-in drivers
                 ✗ Support Type #1 sort-key field
                 ✗ Support @saved pseudo-entry
                 ✗ Support Type #1 devicetree field
                 ✗ Enroll SecureBoot keys
                 ✗ Retain SHIM protocols
                 ✓ Menu can be disabled
                 ✗ Multi-Profile UKIs are supported
                 ✗ Loader reports network boot URL
                 ✗ Support Type #1 uki field
                 ✗ Support Type #1 uki-url field
                 ✗ Loader reports active TPM2 PCR banks
                 ✗ Loader reports firmware keyboard layout
                 ✗ Loader measures SMBIOS information
      Partition: /dev/disk/by-partuuid/152e48bb-6760-4c52-afa9-cc4e23956633
         Loader: └─/boot//EFI/systemd/systemd-bootx64.efi
  Current Entry: arch-linux.efi
  Default Entry: system-opensuse-tumbleweed-6.18.9-1-default-1.conf
[....]
```

- **Limpieza final**

Una vez confirmado que todo funciona bien y ha pasado un margen de tiempo prudencial (**recomendable esperar al menos hasta el siguiente reinicio exitoso**):

```bash
sudo btrfs subvolume delete /@broken_YYYYMMDD
```

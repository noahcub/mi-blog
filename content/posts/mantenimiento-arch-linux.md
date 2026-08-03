---
title: Mantenimiento en Arch Linux
description: Guía de mantenimiento para Arch Linux
date: 2026-07-31 08:00:00 +0100
categories:
   - Arch
tags:
   - Admin
weight: 1
---


# Política de mantenimiento para Arch Linux

Arch es una distro *rolling release*: no hay "versiones", solo un flujo continuo de paquetes nuevos. Eso da ventajas (siempre software reciente) pero exige disciplina, porque los problemas no se acumulan en un "salto de versión" sino que pueden aparecer en cualquier actualización. La política se organiza en tres bloques: **antes de actualizar**, **actualización**, y **limpieza periódica**.

---

## 1. Reglas de oro antes de actualizar

1. **Lee el Arch Linux Homepage / news feed antes de cada actualización grande.**
   Muchas roturas (cambios de firma de claves PGP, migraciones de `/usr`, cambios de configuración manual requeridos) se anuncian ahí. Comando rápido:
   ```bash
   curl -s https://archlinux.org/feeds/news/ | grep -oP '(?<=<title>)[^<]+' | head -20
   ```
   O usa `informant` / `arch-audit` (ver sección de herramientas).

   En mi caso, estoy suscrito al [Feed de Arch Linux News](https://archlinux.org/feeds/news/) a través de mi cliente de correo Thunderbird y ahí veo todas las noticias imporantes antes de un update.

2. **Nunca actualices parcialmente.** No hagas `pacman -S paquete` sin sincronizar antes, y no dejes el sistema a medias (`pacman -Sy` sin el `-u` inmediato). Esto rompe dependencias porque la base de datos local queda desincronizada del repo. Siempre:
   ```bash
   sudo pacman -Syu
   ```
   nunca `-Sy` solo.

3. **No actualices en caliente si vas a apagar el equipo pronto o si es un sistema crítico**, sobre todo si hay actualización de kernel + reinicio pendiente + trabajo importante en curso.

4. **Frecuencia recomendada:** actualizar cada 1–2 semanas es un buen equilibrio (evita acumular demasiados cambios de golpe, que es cuando más conflictos aparecen). Evita dejar pasar meses.

---

## 2. Proceso de actualización (SEMANAL y MENSUAL)

### Actualización general (SEMANAL)
```bash
# 1. Revisar noticias de Arch (romper antes de romper)
informant check          # si tienes 'informant' instalado

# 2. Sincronizar y actualizar
sudo pacman -Syu

# 3. Reiniciar si hubo actualización de kernel, systemd, o mesa/nvidia
# 3. Reiniciar si hubo actualización de kernel, systemd, o mesa/nvidia
# 3. Reiniciar si hubo actualización de kernel, systemd, o mesa/nvidia
# IMPORTANTÍSIMO: Reiniciar en estos casos
# 3. Reiniciar si hubo actualización de kernel, systemd, o mesa/nvidia
# 3. Reiniciar si hubo actualización de kernel, systemd, o mesa/nvidia

# Creo que esta fue la causa de un ruptura de mi sistema hace unos días
```

### AUR (si usas un helper como `yay` o `paru`) (SEMANAL)
- Actualiza AUR por separado y revisa el `PKGBUILD` de paquetes nuevos o sospechosos antes de compilar (nunca confíes ciegamente).
- **IMPORTANTÍSIMO: Revisar el `PKGBUILD`.** Si es complejo, se puede pasar a una IA para que lo revise y nos de sus conclusiones.
- Los paquetes AUR pueden romperse cuando cambia una dependencia oficial; mantenlos al día también, no solo el repo oficial, para evitar conflictos de versiones.  
- En mi caso uso paru.
```bash
paru -Syu
```

### Manejo de archivos `.pacnew` / `.pacsave` (MENSUAL)
Este es el punto que más gente descuida y es clave en Arch. Cuando un paquete actualiza un archivo de configuración que tú modificaste, pacman NO lo sobreescribe: deja un `archivo.pacnew` al lado. Si no los revisas, tu sistema queda con configuración desactualizada silenciosamente.

```bash
# Buscar todos los .pacnew/.pacsave pendientes
sudo pacman -Qii | grep -E '^Backup files' # (poco práctico)
# mejor:
find /etc -name '*.pacnew'

# Herramienta dedicada (recomendado instalar):
sudo pacman -S pacdiff
sudo DIFFPROG=vimdiff pacdiff   # o meld, kdiff3, etc.
```
Revisa esto **después de cada actualización relevante**.


---

## 3. Limpieza periódica del sistema

### a) Caché de paquetes de pacman
Pacman guarda **todas** las versiones descargadas en `/var/cache/pacman/pkg/`, y esto crece indefinidamente.

```bash
# Ver tamaño actual
du -sh /var/cache/pacman/pkg/

# Opción simple: dejar solo las 2 versiones más recientes de cada paquete instalado
sudo pacman -Sc

# Opción agresiva: borrar TODO lo que no esté instalado actualmente
sudo pacman -Scc
```

En mi caso he instalado `pacman-contrib` y usando `paccache`, tenemos las siguientes opciones:
```bash
sudo pacman -S pacman-contrib
sudo paccache -r          # deja 3 versiones por paquete (default)
sudo paccache -rk1        # deja solo 1 versión
sudo paccache -ruk0       # borra cachés de paquetes YA desinstalados (huérfanos), sin dejar ninguna
```
Recomendación: automatizar `paccache -rk2` semanalmente con un timer de systemd (ver sección 4).

### b) Paquetes huérfanos (dependencias que ya nadie usa) (MENSUAL)
```bash
# Listar huérfanos
pacman -Qtdq

# Eliminarlos (revisa la lista antes de confirmar)
sudo pacman -Rns $(pacman -Qtdq)
```

### c) Journal de systemd
El log binario de systemd puede crecer mucho si no está limitado.
```bash
journalctl --disk-usage
sudo journalctl --vacuum-time=2weeks
# o por tamaño:
sudo journalctl --vacuum-size=300M
```
Mejor: configúralo de forma permanente en `/etc/systemd/journald.conf`:
```ini
[Journal]
SystemMaxUse=300M
```

### d) Kernels antiguos / módulos huérfanos
Si usas `linux-lts` como respaldo, revisa que no se acumulen kernels sin usar. Si un kernel se desinstala, ejecuta:
```bash
sudo mkinitcpio -P     # regenerar todas las initramfs por si acaso
```

### e) Cachés de usuario y "cosas sueltas"
```bash
# Cachés de flatpak (si usas), thumbnails, etc.
flatpak uninstall --unused
rm -rf ~/.cache/thumbnails/*
```

---

## 4. Automatización recomendada (systemd timers)

En vez de acordarte manualmente, crea timers. Ejemplo para limpiar caché de pacman cada semana:

`/etc/systemd/system/paccache.timer` (ya viene con `pacman-contrib`, solo actívalo):
```bash
sudo systemctl enable --now paccache.timer
```

Para revisar `.pacnew` automáticamente tras cada actualización, puedes engancharlo a un *pacman hook* en `/etc/pacman.d/hooks/`:
```ini
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = *

[Action]
Description = Comprobando archivos .pacnew...
When = PostTransaction
Exec = /usr/bin/find /etc -name '*.pacnew'
```

---

## 5. Herramientas útiles a instalar

| Paquete | Para qué sirve |
|---|---|
| `pacman-contrib` | `paccache`, `pacdiff`, `checkupdates` |
| `informant` | avisa de noticias importantes de Arch antes de actualizar |
| `arch-audit` | detecta paquetes instalados con vulnerabilidades conocidas (CVE) |
| `reflector` | mantiene actualizada y ordenada tu lista de mirrors por velocidad/ubicación |
| `rebuild-detector` (AUR) | detecta paquetes que necesitan recompilarse tras actualizar una librería |

Ejemplo de uso de `reflector` (mantener mirrors rápidos, muy recomendable en Arch):
```bash
sudo pacman -S reflector
sudo reflector --country Spain --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

---

## 6. Checklist resumido para no leer el hilo comleto

**Cada actualización (semanal/quincenal):**
- [ ] `informant check` o revisar arch news
- [ ] `sudo pacman -Syu`
- [ ] Revisar `.pacnew` con `pacdiff`
- [ ] Reiniciar si hubo kernel/systemd/drivers gráficos
- [ ] `paru -Syu`

**Mensual:**
- [ ] `pacman -Qtdq` → eliminar huérfanos
- [ ] `paccache -rk2` (o dejar que el timer lo haga)
- [ ] `arch-audit` para ver CVEs pendientes
- [ ] Revisar `journalctl --disk-usage`

**Puntual / cuando algo se ve raro:**
- [ ] `rebuild-detector` tras actualizar una librería base (glibc, openssl, etc.)
- [ ] Backup de `/etc` y lista de paquetes (`pacman -Qqe > pkglist.txt`) antes de actualizaciones grandes o cambios de kernel/init

---

## Nota final sobre resiliencia
En Arch, el mejor "seguro" no es evitar actualizar (eso empeora las cosas, porque acumulas más cambios de golpe), sino:
1. Mantener un **snapshot del sistema** (Timeshift con Btrfs, o Snapper) antes de actualizaciones grandes. Esta opción está perfectamente explicada en el manual [Restaurar un snapshot en Arch](https://blog.lasnotasdenoah.com/posts/recovery-snapper/).  
2. Guardar siempre `pacman -Qqe > ~/pkglist-$(date +%F).txt` periódicamente, para poder reconstruir el sistema rápido si algo se rompe. Esta opción se merece un nuevo manual que actualizaremos en proximos días.  
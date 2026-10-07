---
title: Cambiar la hora y la zona horaria en Linux
tags:
  - linux
  - systemd
---

En systemd todo el reloj se administra con `timedatectl`: hora del sistema,
zona horaria, reloj de hardware y sincronización NTP.

## 1. Estado actual

```bash
timedatectl status
```

| Campo                       | Qué significa                                     |
| --------------------------- | ------------------------------------------------- |
| `Local time`                | la hora que ves, con la zona ya aplicada          |
| `Universal time`            | la hora real del sistema en UTC                   |
| `RTC time`                  | el reloj de la placa, sobrevive al apagado        |
| `System clock synchronized` | si NTP ya corrigió el reloj                       |

Si `Universal time` está bien y `Local time` no, el problema es la zona
horaria. Basta el paso 2.

## 2. Zona horaria

```bash
timedatectl list-timezones | grep -i lima
sudo timedatectl set-timezone America/Lima
```

No mueve el reloj, solo cómo se muestra. Linux guarda siempre la hora en UTC.

## 3. Sincronización por NTP

```bash
sudo timedatectl set-ntp true
```

Para ver el servidor en uso y la desviación medida:

```bash
timedatectl show-timesync --all
systemctl status systemd-timesyncd
```

Con NTP activo el sistema rechaza fijar la hora a mano. Ese es el error
`Failed to set time: Automatic time synchronization is enabled`.

## 4. Hora manual

Solo para máquinas sin internet. Hay que apagar NTP antes.

```bash
sudo timedatectl set-ntp false
sudo timedatectl set-time "2026-09-09 22:00:00"
sudo timedatectl set-ntp true
```

El formato es `YYYY-MM-DD HH:MM:SS` en hora local, no UTC. También acepta solo
`"22:00:00"`.

## 5. Reloj de hardware

```bash
sudo hwclock --systohc   # sistema -> RTC
sudo hwclock --hctosys   # RTC -> sistema, si el arranque trajo hora absurda
```

El RTC debe guardar UTC. Solo se pone en hora local si la máquina arranca
también Windows.

```bash
sudo timedatectl set-local-rtc 0
```

## 6. Ejemplo

En esta máquina: `Local time: Wed 2026-09-09 22:00 -05`, `Universal time: Thu
2026-09-10 03:00 UTC`, zona `America/Lima`, `RTC in local TZ: no`,
`System clock synchronized: yes`.

Las cinco horas de diferencia son el desfase de Lima, el RTC guarda UTC y NTP
mantiene el reloj solo. Configuración correcta.

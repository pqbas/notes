---
title: Cambiar la hora y la zona horaria en Linux
tags:
  - linux
  - systemd
---

En un sistema con systemd la hora se administra con `timedatectl`, que agrupa
en un solo comando el reloj del sistema, la zona horaria, el reloj de hardware
y la sincronización por NTP. Los pasos siguientes cubren el caso normal, que es
corregir la zona horaria, y el caso de una máquina sin red, donde hay que poner
la hora a mano.

## 1. Estado actual del reloj

El estado se consulta primero, porque distingue el problema real entre una zona
horaria mal puesta y un reloj que se fue de hora.

```bash
timedatectl status
```

La salida describe cuatro cosas distintas que conviene no confundir.

| Campo                        | Qué significa                                          |
| ---------------------------- | ------------------------------------------------------ |
| `Local time`                 | la hora que ves, ya con la zona horaria aplicada       |
| `Universal time`             | la hora real del sistema en UTC, sin zona              |
| `RTC time`                   | el reloj de hardware, el que sobrevive al apagado      |
| `System clock synchronized`  | si NTP ya corrigió el reloj contra un servidor         |

Si `Universal time` es correcta pero `Local time` no, el problema es la zona
horaria y no el reloj, de modo que basta el paso 2 y no hace falta tocar la
hora.

## 2. Zona horaria

Las zonas disponibles se listan con el nombre que espera `timedatectl`, en
formato `Región/Ciudad`.

```bash
timedatectl list-timezones | grep -i lima
```

El cambio se aplica nombrando la zona, y requiere privilegios porque afecta a
todo el sistema.

```bash
sudo timedatectl set-timezone America/Lima
```

Cambiar la zona no mueve el reloj del sistema, solo la forma de mostrarlo, ya
que internamente Linux siempre guarda la hora en UTC.

## 3. Sincronización automática por NTP

Lo habitual es no poner la hora a mano, sino dejar que el sistema la corrija
contra un servidor de tiempo.

```bash
sudo timedatectl set-ntp true
```

El detalle de la sincronización, con el servidor en uso y la desviación
medida, se ve con el cliente de systemd.

```bash
timedatectl show-timesync --all
systemctl status systemd-timesyncd
```

Mientras NTP está activo el sistema rechaza cualquier intento de fijar la hora
a mano, con el error `Failed to set time: Automatic time synchronization is
enabled`, que es la causa habitual de que `set-time` no funcione.

## 4. Hora manual

La hora manual solo hace falta en una máquina sin salida a internet, y exige
apagar antes la sincronización automática.

```bash
sudo timedatectl set-ntp false
sudo timedatectl set-time "2026-09-09 22:00:00"
```

El formato es `YYYY-MM-DD HH:MM:SS` y se interpreta en la zona horaria local
configurada, no en UTC.

Solo la hora, sin la fecha, se acepta igual.

```bash
sudo timedatectl set-time "22:00:00"
```

Terminado el ajuste conviene volver a habilitar NTP, porque el reloj de una
máquina se desvía por su cuenta con el paso de los días.

```bash
sudo timedatectl set-ntp true
```

## 5. Reloj de hardware

El RTC es el reloj de la placa, el que mantiene la hora con la máquina
apagada, y se escribe desde el reloj del sistema.

```bash
sudo hwclock --systohc
```

La operación inversa, útil cuando el sistema arrancó con una hora absurda pero
el RTC está bien, copia el hardware al sistema.

```bash
sudo hwclock --hctosys
```

El RTC debe guardarse en UTC, que es lo que espera Linux, y solo se pone en
hora local cuando la máquina arranca también Windows.

```bash
sudo timedatectl set-local-rtc 0
```

## 6. Ejemplo de ejecución

En esta máquina `timedatectl status` devuelve `Local time: Wed 2026-09-09
22:00 -05` contra `Universal time: Thu 2026-09-10 03:00 UTC`, con zona
`America/Lima`, `RTC in local TZ: no` y `System clock synchronized: yes`.

Esa combinación es la correcta: la diferencia de cinco horas entre local y UTC
es exactamente el desfase de Lima, el RTC guarda UTC como corresponde, y NTP
mantiene el reloj en hora sin intervención.

---
title: Cambiar la hora y la zona horaria en Linux
tags:
  - linux
  - systemd
---

En un sistema con systemd la hora se administra con `timedatectl`, que agrupa en
un solo comando el reloj del sistema, la zona horaria, el reloj de hardware y la
sincronización por NTP.

Los pasos siguientes cubren el caso normal, que es corregir la zona horaria, y
el caso de una máquina sin red, donde hay que poner la hora a mano.

## 1. Estado actual del reloj

El estado se consulta primero, porque distingue el problema real entre una zona
horaria mal puesta y un reloj que se fue de hora.

```bash
timedatectl status
```

```text
               Local time: Wed 2026-09-16 16:18:06 -05
           Universal time: Wed 2026-09-16 21:18:06 UTC
                 RTC time: Wed 2026-09-16 21:18:06
                Time zone: America/Lima (-05, -0500)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

La salida describe lo siguiente:

| Campo                       | Descripción                                      |
| --------------------------- | ------------------------------------------------ |
| `Local time`                | La hora que ves, ya con la zona horaria aplicada |
| `Universal time`            | La hora del sistema en UTC, sin zona             |
| `RTC time`                  | El reloj de hardware                             |
| `System clock synchronized` | Si NTP ya corrigió el reloj contra un servidor   |

Si `Universal time` es correcta pero `Local time` no, el problema es la zona
horaria y no el reloj, de modo que basta el paso 2 y no hace falta tocar la
hora.

## 2. Zona horaria

Las zonas disponibles se listan con el nombre que espera `timedatectl`, en
formato `Región/Ciudad`.

```bash
timedatectl list-timezones | grep -i <ciudad>
```

```text
America/Lima
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

El detalle de la sincronización, con el servidor en uso y la desviación medida,
se ve con el cliente de systemd.

```bash
timedatectl show-timesync --all
```

```text
LinkNTPServers=
SystemNTPServers=
RuntimeNTPServers=
FallbackNTPServers=ntp.ubuntu.com
ServerName=ntp.ubuntu.com
ServerAddress=185.125.190.57
RootDistanceMaxUSec=5s
PollIntervalMinUSec=32s
PollIntervalMaxUSec=34min 8s
PollIntervalUSec=34min 8s
NTPMessage={ Leap=0, Version=4, Mode=4, Stratum=2, ... Jitter=24.414ms }
Frequency=-1332573
```

```bash
systemctl status systemd-timesyncd
```

```text
● systemd-timesyncd.service - Network Time Synchronization
     Loaded: loaded (/usr/lib/systemd/system/systemd-timesyncd.service; enabled)
     Active: active (running) since Wed 2026-09-16 07:52:00 -05; 8h ago
   Main PID: 1097 (systemd-timesyn)
     Status: "Contacted time server 185.125.190.57:123 (ntp.ubuntu.com)."
```

Mientras NTP está activo el sistema rechaza cualquier intento de fijar la hora a
mano, que es la causa habitual de que `set-time` no funcione.

```text
Failed to set time: Automatic time synchronization is enabled
```

## 4. Hora manual

La hora manual solo hace falta en una máquina sin salida a internet, y exige
apagar antes la sincronización automática. El formato es `YYYY-MM-DD HH:MM:SS` y
se interpreta en la zona horaria local configurada, no en UTC.

```bash
# se apaga la sincronizacion automatica
sudo timedatectl set-ntp false

# configura con hora y fecha
sudo timedatectl set-time "2026-09-09 22:00:00"

# configura solo la hora
sudo timedatectl set-time "22:00:00"
```

Terminado el ajuste conviene volver a habilitar NTP, porque el reloj de una
máquina se desvía por su cuenta con el paso de los días.

```bash
sudo timedatectl set-ntp true
```

## 5. Reloj de hardware

El RTC es el reloj de la placa, el que mantiene la hora con la máquina apagada,
y se escribe desde el reloj del sistema.

Con NTP activo casi nunca hace falta tocarlo, porque `systemd-timesyncd` baja la
hora del sistema al RTC por su cuenta cada once minutos. El ajuste manual queda
para estos casos:

- En una máquina sin red el RTC es la única fuente de hora al arrancar, de modo
  que después de corregir la hora a mano hay que escribirla en el hardware.
- En un arranque dual con Windows uno de los dos sistemas muestra la hora
  corrida, porque Windows lee el RTC como hora local y Linux como UTC.
- Un RTC con deriva grande o con la pila agotada hace que cada arranque aparezca
  con una hora absurda, aunque la configuración esté bien.
- Los equipos sin RTC con batería, como una Raspberry Pi, no conservan la hora
  al apagarse y la recuperan con NTP o con `fake-hwclock`.

```bash
sudo hwclock --systohc
```

La operación inversa, útil cuando el sistema arrancó con una hora absurda pero
el RTC está bien, copia el hardware al sistema.

```bash
sudo hwclock --hctosys
```

El RTC debe guardarse en UTC, que es lo que espera Linux, y solo se pone en hora
local cuando la máquina arranca también Windows.

```bash
sudo timedatectl set-local-rtc 0
```

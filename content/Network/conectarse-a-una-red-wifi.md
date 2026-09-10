---
title: Conectarse a una red WiFi desde la terminal en Linux
tags:
  - network
  - linux
  - wifi
---

Los pasos siguientes conectan una máquina Linux a una red WiFi sin usar el
entorno gráfico, primero con `nmcli`, que es el cliente de línea de comandos de
NetworkManager y viene instalado en casi toda distribución de escritorio, y
después con `iw` y `wpa_supplicant`, que sirven en un sistema mínimo donde
NetworkManager no está presente.

## 1. Interfaz inalámbrica disponible

La interfaz que se va a usar se identifica listando los dispositivos de red,
donde el que aparece con tipo `wifi`, por ejemplo `wlo1` o `wlan0`, es el
adaptador inalámbrico.

```bash
nmcli device status
```

| Estado          | Qué significa                                     |
| --------------- | ------------------------------------------------- |
| `connected`     | la interfaz ya está asociada a una red            |
| `disconnected`  | la interfaz está lista pero sin red               |
| `unavailable`   | la radio está apagada o falta el driver           |
| `unmanaged`     | NetworkManager no controla esa interfaz           |

Si la interfaz aparece como `unavailable`, la causa habitual es que la radio
está bloqueada, y el bloqueo se consulta y se levanta con `rfkill`.

```bash
rfkill list
rfkill unblock wifi
```

## 2. Redes al alcance

El escaneo pide al adaptador que recorra los canales y devuelva las redes que
anuncian su presencia, con el nombre, la potencia de la señal y el tipo de
cifrado de cada una.

```bash
nmcli device wifi list
```

El listado se puede forzar a repetir el escaneo en vez de mostrar el resultado
en caché, que es lo que conviene cuando una red recién encendida todavía no
aparece.

```bash
nmcli device wifi rescan
nmcli device wifi list
```

## 3. Conexión a la red

La conexión se hace nombrando el SSID y la contraseña, y NetworkManager se
encarga de la asociación, la autenticación WPA y la petición DHCP en un solo
paso.

```bash
nmcli device wifi connect "<SSID>" password "<contraseña>"
```

Escribir la contraseña en el comando la deja guardada en el historial de la
shell, así que en una máquina compartida conviene pedirla de forma interactiva
con `--ask`, que la lee sin mostrarla.

```bash
nmcli --ask device wifi connect "<SSID>"
```

Una red oculta no aparece en el escaneo, por lo que hay que declarar
explícitamente que el SSID no se anuncia.

```bash
nmcli device wifi connect "<SSID>" password "<contraseña>" hidden yes
```

## 4. Verificación de la conexión

La comprobación tiene tres niveles, y separarlos indica en qué capa está la
falla cuando algo no funciona.

```bash
nmcli connection show --active   # la red está asociada
ip -4 addr show wlo1             # el DHCP entregó una dirección
ping -c3 8.8.8.8                 # hay salida a internet
```

Si hay dirección IP y responde `8.8.8.8` pero no los nombres de dominio, el
problema es de resolución DNS y no de la conexión WiFi.

```bash
resolvectl status
```

## 5. Gestión de redes guardadas

Cada conexión exitosa queda guardada como un perfil que se reutiliza de forma
automática la próxima vez que la red esté al alcance.

```bash
nmcli connection show
```

Los perfiles se activan, se desactivan y se borran por nombre, donde borrar es
lo que hace falta cuando la contraseña de la red cambió y el perfil viejo
sigue intentando autenticarse con la anterior.

```bash
nmcli connection up "<SSID>"
nmcli connection down "<SSID>"
nmcli connection delete "<SSID>"
```

La contraseña guardada de un perfil se puede recuperar, lo que sirve para
leer la clave de una red a la que la máquina ya se conectó antes.

```bash
nmcli --show-secrets connection show "<SSID>" | grep psk
```

## 6. Sin NetworkManager

En un sistema mínimo, por ejemplo un servidor o un live USB, los tres pasos
que NetworkManager agrupa hay que darlos por separado, empezando por levantar
la interfaz.

```bash
ip link set wlan0 up
iw dev wlan0 scan | grep SSID
```

La autenticación WPA la hace `wpa_supplicant`, que lee las credenciales de un
archivo de configuración generado con `wpa_passphrase`.

```bash
wpa_passphrase "<SSID>" "<contraseña>" > /etc/wpa_supplicant/wifi.conf
wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wifi.conf
```

| Flag | Qué aporta                                    |
| ---- | --------------------------------------------- |
| `-B` | deja el proceso corriendo en segundo plano    |
| `-i` | indica la interfaz sobre la que se autentica  |
| `-c` | apunta al archivo con el SSID y la clave      |

La asociación no entrega dirección IP por sí sola, así que el último paso es
pedirla al servidor DHCP de la red.

```bash
dhclient wlan0
```

Una red abierta, sin cifrado, no necesita `wpa_supplicant` y se asocia
directamente con `iw`.

```bash
iw dev wlan0 connect "<SSID>"
dhclient wlan0
```

Una vez conectada la máquina, el rango de la red se explora con los pasos de
[[Network/descubrimiento-de-dispositivos|Descubrimiento de dispositivos por IP en una red local]].

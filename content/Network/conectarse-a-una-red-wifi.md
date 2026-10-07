---
title: Conectarse a una red WiFi desde la terminal en Linux
tags:
  - network
  - linux
  - wifi
---

Con `nmcli`, el cliente de NetworkManager que ya viene en toda distro de
escritorio. Al final, la alternativa con `iw` y `wpa_supplicant` para sistemas
mínimos sin NetworkManager.

## 1. Interfaz inalámbrica

```bash
nmcli device status
```

La que aparece con tipo `wifi` es el adaptador, por ejemplo `wlo1` o `wlan0`.

| Estado         | Qué significa                            |
| -------------- | ---------------------------------------- |
| `connected`    | ya está asociada a una red               |
| `disconnected` | lista pero sin red                       |
| `unavailable`  | radio apagada o falta el driver          |
| `unmanaged`    | NetworkManager no la controla            |

Si sale `unavailable`, casi siempre es la radio bloqueada:

```bash
rfkill list
rfkill unblock wifi
```

## 2. Redes al alcance

```bash
nmcli device wifi list
```

Muestra el resultado en caché. Para forzar un escaneo nuevo, útil cuando una
red recién encendida no aparece:

```bash
nmcli device wifi rescan && nmcli device wifi list
```

## 3. Conectar

```bash
nmcli device wifi connect "<SSID>" password "<contraseña>"
```

NetworkManager hace asociación, autenticación WPA y DHCP en un solo paso.

La contraseña ahí queda en el historial de la shell. En máquina compartida,
`--ask` la pide sin mostrarla:

```bash
nmcli --ask device wifi connect "<SSID>"
```

Una red oculta no sale en el escaneo y hay que declararlo:

```bash
nmcli device wifi connect "<SSID>" password "<contraseña>" hidden yes
```

## 4. Verificar

Tres niveles separados, para saber en qué capa está la falla:

```bash
nmcli connection show --active   # la red está asociada
ip -4 addr show wlo1             # el DHCP entregó dirección
ping -c3 8.8.8.8                 # hay salida a internet
```

Si hay IP y responde `8.8.8.8` pero no los dominios, el problema es DNS:

```bash
resolvectl status
```

## 5. Redes guardadas

Cada conexión exitosa queda como perfil y se reutiliza sola.

```bash
nmcli connection show
nmcli connection up "<SSID>"
nmcli connection down "<SSID>"
nmcli connection delete "<SSID>"
```

Borrar el perfil es lo que hace falta cuando la red cambió de contraseña y el
viejo sigue intentando con la anterior.

Para leer la clave guardada de una red a la que ya te conectaste:

```bash
nmcli --show-secrets connection show "<SSID>" | grep psk
```

## 6. Sin NetworkManager

En un servidor o un live USB hay que dar por separado los tres pasos que
NetworkManager agrupa.

```bash
ip link set wlan0 up
iw dev wlan0 scan | grep SSID
```

La autenticación WPA la hace `wpa_supplicant`:

```bash
wpa_passphrase "<SSID>" "<contraseña>" > /etc/wpa_supplicant/wifi.conf
wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wifi.conf
```

| Flag | Para qué                            |
| ---- | ----------------------------------- |
| `-B` | deja el proceso en segundo plano    |
| `-i` | la interfaz                         |
| `-c` | el archivo con el SSID y la clave   |

Asociarse no entrega IP. Falta pedirla:

```bash
dhclient wlan0
```

Una red abierta no necesita `wpa_supplicant`:

```bash
iw dev wlan0 connect "<SSID>"
dhclient wlan0
```

Ya conectado, el rango se explora con
[[Network/descubrimiento-de-dispositivos|Descubrimiento de dispositivos por IP en una red local]].

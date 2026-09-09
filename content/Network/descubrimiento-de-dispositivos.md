---
title: Descubrimiento de dispositivos por IP en una red local
tags:
  - network
  - linux
---

Los cuatro pasos siguientes identifican qué hosts están vivos en una LAN, por
ejemplo `192.168.50.0/24`, usando solo herramientas que ya trae el sistema,
por lo que sirven en una máquina donde no están instalados `nmap` ni
`arp-scan`.

## 1. Subred propia de la máquina

La subred que hay que barrer se lee de la interfaz activa, donde la línea
`inet x.x.x.x/24` de la interfaz en uso, por ejemplo `wlo1`, indica el rango
al que pertenece la máquina.

```bash
ip -4 addr show
```

## 2. Hosts ya conocidos por el kernel

La tabla de vecinos guarda solo los hosts con los que la máquina conversó
recientemente, de modo que es una consulta pasiva que no genera tráfico.

```bash
ip neigh show
```

| Estado      | Qué significa                                     |
| ----------- | ------------------------------------------------- |
| `REACHABLE` | el host respondió y está confirmado como vivo     |
| `STALE`     | se vio antes, sin reconfirmación reciente         |
| `FAILED`    | no hubo respuesta                                 |

## 3. Barrido de la subred con ping

El descubrimiento activo se hace lanzando un `ping` por cada dirección del
rango en segundo plano, donde `-c1` envía un solo paquete y `-W1` corta la
espera al segundo.

```bash
# Barremos las 254 direcciones del rango en paralelo
for i in $(seq 1 254); do
  (ping -c1 -W1 192.168.50.$i >/dev/null 2>&1 && echo "192.168.50.$i UP") &
done
wait
```

El prefijo del `ping` y el del `echo` deben ser el mismo, porque si difieren
los hosts que responden se reportan bajo una subred que no es la suya.

Mientras corre, la shell imprime `exit 1` por cada ping sin respuesta, que es
el código normal de "sin respuesta" y no un error, dado que la mayoría de las
254 direcciones no contesta.

Al terminar conviene repetir `ip neigh show`, ya que el barrido deja en la
tabla ARP la dirección MAC de cada host que respondió.

## 4. Puerto SSH abierto en un host candidato

La prueba de puerto se hace con `/dev/tcp`, un archivo especial de bash que al
leerse o escribirse abre un socket TCP hacia el host y puerto indicados, por lo
que no hace falta `nc` ni `nmap`.

```bash
timeout 1.5 bash -c "cat < /dev/null > /dev/tcp/<ip>/22" 2>/dev/null && echo "SSH OPEN" || echo "closed/filtered"
```

Cada pieza del comando cumple una función dentro de la prueba.

| Elemento                        | Qué aporta                                             |
| ------------------------------- | ------------------------------------------------------ |
| `cat < /dev/null > /dev/tcp/...`| abre la conexión sin enviar datos, solo para probarla  |
| `timeout 1.5`                   | corta el intento para que un host mudo no cuelgue todo |
| `2>/dev/null`                   | oculta el error de conexión rechazada                  |

El banner confirma qué servicio está escuchando detrás del puerto.

```bash
timeout 3 bash -c 'exec 3<>/dev/tcp/<ip>/22; timeout 2 cat <&3'
```

La misma prueba se aplica sobre una lista de candidatos con un bucle.

```bash
for ip in 192.168.50.1 192.168.50.51 192.168.50.103; do
  if timeout 1.5 bash -c "cat < /dev/null > /dev/tcp/$ip/22" 2>/dev/null; then
    echo "$ip -> SSH OPEN"
  else
    echo "$ip -> closed/filtered"
  fi
done
```

## 5. Ejemplo de ejecución

Sobre la subred `192.168.50.0/24`, desde el host `192.168.50.188` en `wlo1`, el
barrido devolvió diez direcciones vivas, de las cuales solo `192.168.50.1`
tenía el puerto 22 abierto, correspondiente a la interfaz de administración del
router y no al equipo buscado.

El equipo esperado en `192.168.50.103` no respondió, por lo que estaba apagado
o había cambiado de dirección, que es el caso donde la tabla de vecinos del
paso 2 ayuda a reconocer la dirección anterior.

---
title: Recuperar una sesión de Hyprland con el bloqueo colgado
tags:
  - hyprland
  - wayland
  - linux
  - hyprlock
---

Cuando el programa de bloqueo muere sin desbloquear, Hyprland deja la sesión
bloqueada y muestra una pantalla de emergencia con instrucciones en inglés. La
máquina sigue funcionando y las ventanas siguen abiertas, pero no hay forma de
escribir la contraseña porque el programa que la pedía ya no existe. Los pasos
siguientes recuperan la sesión desde otro TTY, y el último paso es el que de
verdad resuelve el problema aunque no aparezca en el mensaje de Hyprland.

## 1. Por qué queda bloqueada

El bloqueo de pantalla en Wayland no lo dibuja el compositor sino un cliente
aparte, `hyprlock`, a través del protocolo `ext-session-lock-v1`. El compositor
tapa todo lo demás y solo levanta el bloqueo cuando ese cliente le avisa que la
autenticación fue correcta.

| Situación                        | Qué hace el compositor                       |
| -------------------------------- | -------------------------------------------- |
| el cliente autentica y avisa     | libera el bloqueo y devuelve el escritorio   |
| el cliente muere sin avisar      | mantiene el bloqueo y dibuja la emergencia   |

Mantener el bloqueo es una decisión de seguridad: si matar el bloqueador
desbloqueara la pantalla, bastaría un `pkill` para saltarse la contraseña. Por
eso `kill`, `pkill` o un crash de `hyprlock` dejan la sesión inaccesible.

## 2. Entrar por otro TTY

La recuperación se hace desde una consola de texto distinta a la que ocupa
Hyprland, que se alcanza con `Ctrl + Alt + F<N>`. En una instalación normal
quedan libres la `F3` y siguientes.

Ahí se inicia sesión con el usuario y la contraseña de siempre, y se averigua en
qué TTY vive cada sesión, porque ese número hace falta al final.

```bash
loginctl list-sessions
```

La columna `TTY` distingue las dos: la de tipo `wayland` es Hyprland, y la de
tipo `tty` es la consola recién abierta.

```bash
loginctl show-session <id> -p Type -p TTY -p Active
```

## 3. Permitir que otro bloqueador tome el relevo

Hyprland solo deja que un cliente nuevo se haga cargo de un bloqueo existente si
la opción está habilitada, que es lo primero que pide la pantalla de emergencia.

```bash
hyprctl keyword misc:allow_session_lock_restore 1
```

El valor actual se consulta antes, y si ya dice `true` este paso no cambia nada.

```bash
hyprctl getoption misc:allow_session_lock_restore
```

## 4. Relanzar el bloqueador desde el compositor

Un `hyprlock` lanzado a mano desde la consola de texto no hereda el entorno de
la sesión gráfica, así que conviene pedirle al propio Hyprland que lo ejecute.

```bash
killall -9 hyprlock
hyprctl dispatch exec hyprlock
```

En una configuración de Hyprland escrita en Lua, como la de KooL Dots, el
despachador recibe una expresión Lua y la forma anterior falla con
`')' expected near 'hyprlock'`. La equivalente es la siguiente.

```bash
hyprctl dispatch 'hl.dsp.exec_cmd("hyprlock")'
```

El proceso se comprueba antes de seguir, porque si no arrancó el resto no tiene
sentido.

```bash
pgrep -ax hyprlock
```

## 5. Activar la sesión gráfica

Este es el paso que falta en el mensaje de Hyprland y sin el cual todo lo
anterior parece no funcionar. Mientras el TTY de la sesión gráfica no esté
activo, `hyprlock` arranca pero nunca llega a dibujar nada, de modo que la
pantalla sigue mostrando el mensaje de emergencia y parece que el relanzamiento
falló.

```bash
loginctl activate 2
```

El número es el de la sesión de tipo `wayland` obtenida en el paso 2. El
resultado se verifica leyendo el estado, que debe pasar a `yes`.

```bash
loginctl show-session 2 -p Active
```

En ese momento la pantalla muestra el campo de contraseña de `hyprlock`, se
escribe la clave y la sesión vuelve con todas las ventanas donde estaban. Si
algo sale mal, `Ctrl + Alt + F3` devuelve a la consola de texto.

## 6. Evitar que vuelva a pasar

La causa habitual es lanzar `hyprlock` para probar un cambio de configuración y
cerrarlo con `pkill` o `Ctrl + C` en vez de autenticarse. Los cambios en
`~/.config/hypr/hyprlock.conf` se editan y se dan por buenos sin ejecutarlo, y
si hay que verlo se bloquea de verdad y se desbloquea escribiendo la contraseña.

El diagnóstico rápido de un bloqueo colgado es la salida de `hyprctl monitors`,
donde el compositor declara que el bloqueo sigue activo.

```bash
hyprctl monitors | grep solitaryBlockedBy
```

Si la línea incluye `session lock` y `pgrep hyprlock` no devuelve nada, la
sesión está en este estado exacto y aplican los pasos anteriores.

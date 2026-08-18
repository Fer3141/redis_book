# 4. Pub/Sub, transacciones y Lua

## Pub/Sub

Redis funciona como sistema de publicación y suscripción. Un cliente publica mensajes en un canal y otros clientes, suscritos a ese canal, los reciben en tiempo real.

```bash
SUBSCRIBE noticias
```
Este comando bloquea la conexión: el cliente queda "escuchando" el canal `noticias` y cada mensaje publicado le llega automáticamente, sin tener que hacer polling.

```bash
PUBLISH noticias "nuevo artículo publicado"
```
Desde **otra** conexión, publica un mensaje en el canal. `PUBLISH` devuelve un número: cuántos suscriptores lo recibieron en ese instante (0 si nadie estaba escuchando).

```bash
PSUBSCRIBE noticias.*
```
Suscripción por patrón: recibe mensajes de cualquier canal que matchee (`noticias.deportes`, `noticias.tecnología`, etc.), sin tener que suscribirte a cada uno por separado.

**El punto que hay que entender bien**: Pub/Sub en Redis **no guarda historial**. Si publicás un mensaje y en ese momento no hay nadie suscrito, el mensaje se pierde para siempre — no queda encolado para que alguien lo lea después. Es útil para notificaciones en vivo (por ejemplo, avisar a un panel de administración que algo cambió), pero **no** sirve como cola confiable de mensajes. Para eso, Redis ofrece **Streams** (`XADD`/`XREAD`), que sí persisten y permiten leer mensajes pasados.

## Transacciones (MULTI/EXEC)

```bash
MULTI
SET saldo:1 100
INCRBY saldo:1 -20
EXEC
```

- `MULTI` marca el inicio de una transacción: a partir de acá, los comandos que escribas no se ejecutan de inmediato, se **encolan**.
- `EXEC` ejecuta todos los comandos encolados, uno tras otro, sin que ningún otro cliente pueda meter un comando en el medio. Es una garantía de **atomicidad de ejecución**, no de aislamiento tipo SQL: no hay "rollback" si uno de los comandos falla en tiempo de ejecución (por ejemplo, un `WRONGTYPE`), los demás igual se ejecutan.
- `DISCARD` cancela la transacción en curso sin ejecutar nada de lo encolado.

```bash
WATCH saldo:1
MULTI
DECRBY saldo:1 20
EXEC
```

`WATCH` agrega **bloqueo optimista**: le decís a Redis "vigilá esta clave". Si otro cliente modifica `saldo:1` entre el `WATCH` y el `EXEC`, la transacción entera se cancela sola (`EXEC` devuelve `nil`) y no aplica ningún cambio. Esto resuelve un problema real: imaginá que dos procesos leen `saldo:1 = 100` al mismo tiempo para calcular un descuento — sin `WATCH`, ambos podrían escribir un resultado basado en un valor que ya quedó desactualizado. Con `WATCH`, el segundo en llegar se entera de que el dato cambió y puede reintentar con el valor fresco, en vez de pisar el trabajo del primero.

## Scripts Lua

Cuando la lógica es más compleja que unos pocos comandos encolados, Redis puede ejecutar scripts Lua **dentro** del servidor, de forma atómica.

```bash
EVAL "return redis.call('GET', KEYS[1])" 1 saldo:1
```

- El primer argumento es el código Lua.
- El número (`1` en este caso) indica cuántas claves recibe el script.
- `KEYS[1]` dentro del script apunta a `saldo:1`, el primer argumento de tipo clave pasado después del número.

Un ejemplo más útil: descontar saldo solo si alcanza, todo en una sola operación atómica (evitando el problema de "leer, decidir, escribir" en múltiples viajes de red donde otro cliente podría meterse en el medio):

```bash
EVAL "
local saldo = tonumber(redis.call('GET', KEYS[1]))
if saldo >= tonumber(ARGV[1]) then
  return redis.call('DECRBY', KEYS[1], ARGV[1])
else
  return -1
end
" 1 saldo:1 20
```

`ARGV[1]` es el primer argumento "normal" (no-clave) pasado al script, en este caso `20`.

```bash
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
EVALSHA <sha1-devuelto> 1 saldo:1
```

- `SCRIPT LOAD` sube el script al servidor una sola vez y devuelve su hash SHA1.
- `EVALSHA` ejecuta ese script ya cargado usando el hash, en vez de mandar el código Lua completo cada vez — ahorra ancho de banda cuando el mismo script se ejecuta muy seguido.

**Por qué esto importa en la práctica**: mover lógica al servidor con Lua evita el problema clásico de "leer un valor, decidir algo en tu aplicación, y escribir el resultado" en tres pasos separados por la red — porque entre esos pasos, otro cliente puede haber cambiado el dato. Con Lua, todo el ciclo leer-decidir-escribir ocurre como una sola operación atómica dentro de Redis.

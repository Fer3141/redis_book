# Pequeño libro de Redis

Guía rápida de Redis: comandos más usados, tipos de datos, persistencia, replicación y buenas prácticas, explicados en español.

## Índice

- [¿Qué es Redis?](#qué-es-redis)
- [Instalación y arranque](#instalación-y-arranque)
- [Comandos generales sobre claves](#comandos-generales-sobre-claves)
- [Strings](#strings)
- [Listas](#listas)
- [Conjuntos (Sets)](#conjuntos-sets)
- [Hashes](#hashes)
- [Sorted Sets (ZSets)](#sorted-sets-zsets)
- [Persistencia (RDB y AOF)](#persistencia-rdb-y-aof)
- [Replicación](#replicación)
- [Pub/Sub](#pubsub)
- [Transacciones (MULTI/EXEC)](#transacciones-multiexec)
- [Scripts Lua](#scripts-lua)
- [Cluster](#cluster)
- [Buenas prácticas](#buenas-prácticas)
- [Mini proyectos](#mini-proyectos)

---

## ¿Qué es Redis?

Redis es un almacén de datos **en memoria** (in-memory) que se usa como base de datos, caché, cola de trabajos y sistema de mensajería. Es rápido por tres motivos concretos: todo vive en RAM (leer de memoria es ~100.000 veces más rápido que leer de disco), es **single-threaded** para ejecutar comandos (evita locks entre threads, por eso comandos como `INCR` son atómicos sin esfuerzo extra) y sus estructuras están pensadas para acceso O(1) o O(log n).

La contrapartida: un comando muy pesado (por ejemplo `KEYS *` sobre millones de claves) bloquea a todos los demás clientes mientras se ejecuta, porque no hay paralelismo dentro del motor.

Usos típicos: caché de consultas caras, sesiones de usuario con expiración automática, contadores y rankings, colas de trabajo simples, Pub/Sub para notificaciones en tiempo real, y rate limiting.

---

## Instalación y arranque

La forma más rápida de probarlo sin instalar nada en el sistema es con Docker:

```bash
docker run --name redis-book -p 6379:6379 -d redis:7
```
`-p 6379:6379` expone el puerto por defecto de Redis desde el contenedor hacia tu máquina; `-d` lo corre en segundo plano.

```bash
docker exec -it redis-book redis-cli
```
Conecta al cliente interactivo dentro del contenedor.

Si Redis ya está instalado localmente (Linux/macOS, o Windows vía WSL):

```bash
redis-server
redis-cli
```
`redis-server` arranca el servidor; `redis-cli`, en otra terminal, abre el cliente interactivo.

```bash
PING
```
El "hola mundo" de Redis: si responde `PONG`, el servidor está vivo y aceptando comandos.

```bash
INFO server
```
Muestra versión, modo (standalone/cluster/sentinel), uptime y conexiones. Es lo primero que se mira al diagnosticar un problema.

```bash
CONFIG GET maxmemory
CONFIG SET maxmemory 256mb
```
Consulta o cambia un parámetro de configuración en caliente, sin tocar `redis.conf` ni reiniciar el servidor.

```bash
DBSIZE
```
Devuelve cuántas claves hay en la base seleccionada.

---

## Comandos generales sobre claves

```bash
SET saludo "hola"
GET saludo
DEL saludo
EXISTS saludo
```
`SET` crea o sobreescribe una clave. `GET` devuelve el valor, o `nil` si no existe (no es lo mismo que `""`). `DEL` la elimina y devuelve cuántas borró realmente. `EXISTS` devuelve `1`/`0` (o cuántas de varias claves existen).

```bash
TYPE saludo
TTL saludo
EXPIRE saludo 60
```
`TYPE` dice qué estructura es (`string`, `list`, `set`, `hash`, `zset`...), para no usar el comando equivocado sobre una clave. `TTL` muestra los segundos de vida restantes: positivo = segundos, `-1` = sin expiración, `-2` = no existe. `EXPIRE` le pone (o reemplaza) el tiempo de vida a una clave existente.

```bash
PERSIST saludo
RENAME saludo saludo2
COPY saludo2 saludo3
```
`PERSIST` le quita el TTL a una clave. `RENAME` renombra (pisa el destino si ya existía; `RENAMENX` solo renombra si no existe). `COPY` duplica el valor en otra clave sin borrar la original.

> **Evitar en producción**: `KEYS *` recorre todo el keyspace de forma bloqueante. Con muchas claves puede congelar el servidor por segundos. La alternativa es `SCAN` (ver [Buenas prácticas](#buenas-prácticas)).

---

## Strings

El tipo más simple: texto, números o cualquier valor serializado. Un string puede pesar hasta 512 MB.

```bash
SET contador 10
INCR contador
INCRBY contador 5
DECR contador
DECRBY contador 3
```
`INCR`/`DECR` suman o restan de forma **atómica**: si dos procesos incrementan el mismo contador a la vez, nunca se pisan (a diferencia de hacer `GET` + sumar en código + `SET`, que sí tiene condición de carrera). Si la clave no existe, se trata como `0` antes de incrementar.

```bash
SET saldo:1 100 EX 60
SETEX saldo:1 60 100
```
`SET ... EX segundos` crea la clave con expiración en un solo comando. `SETEX clave segundos valor` hace lo mismo con otro orden de argumentos (primero segundos, después valor — error común invertirlo).

```bash
APPEND log "primera línea\n"
STRLEN log
MSET a 1 b 2 c 3
MGET a b c
```
`APPEND` agrega texto al final (o crea la clave). `STRLEN` da el largo en bytes. `MSET`/`MGET` escriben o leen varias claves en un solo viaje de red.

---

## Listas

Secuencias ordenadas, ideales para colas y pilas simples.

```bash
LPUSH tareas "enviar correo"
RPUSH tareas "generar informe"
LRANGE tareas 0 -1
LPOP tareas
RPOP tareas
```
`LPUSH` inserta al principio (izquierda), `RPUSH` al final (derecha). `LRANGE clave 0 -1` lee toda la lista (`-1` = último elemento). Combinando `RPUSH` + `LPOP` desde otro proceso se obtiene una **cola FIFO**.

```bash
LLEN tareas
LINSERT tareas BEFORE "generar informe" "revisar informe"
LSET tareas 0 "enviar correo urgente"
```
`LLEN` cuenta elementos. `LINSERT` inserta antes/después de un valor puntual (recorre la lista, más lento en listas grandes). `LSET` reemplaza el elemento en una posición dada.

---

## Conjuntos (Sets)

Colecciones de valores **únicos**, sin orden garantizado.

```bash
SADD tags "redis" "base-datos" "cache"
SMEMBERS tags
SISMEMBER tags "redis"
SREM tags "cache"
SCARD tags
```
`SADD` agrega elementos (ignora los duplicados en silencio). `SISMEMBER` responde `1`/`0` en O(1) — mucho más eficiente que traer todo con `SMEMBERS` y buscar en código; es el uso más común (¿este usuario ya votó?). `SCARD` cuenta elementos sin traer el contenido.

```bash
SADD equipoA "ana" "luis"
SADD equipoB "luis" "marta"
SINTER equipoA equipoB
SUNION equipoA equipoB
SDIFF equipoA equipoB
```
`SINTER` (intersección) → `luis`. `SUNION` (unión, sin duplicar). `SDIFF` (lo que está en el primero pero no en el segundo) → `ana`.

---

## Hashes

Un hash agrupa varios campos bajo una misma clave — la forma natural de guardar "un registro" sin serializar todo a JSON.

```bash
HSET usuario:1 nombre "Ana" edad 29 ciudad "Madrid"
HGET usuario:1 nombre
HGETALL usuario:1
HINCRBY usuario:1 edad 1
```
`HSET` puede setear varios campos en un comando. `HGET` trae un campo puntual (más eficiente que traer todo). `HGETALL` trae todos los campos — cuidado con hashes muy grandes. `HINCRBY` incrementa un campo numérico de forma atómica.

```bash
HDEL usuario:1 ciudad
HEXISTS usuario:1 nombre
HKEYS usuario:1
HVALS usuario:1
```
`HDEL` borra campos puntuales (no la clave entera). `HEXISTS` comprueba si un campo existe sin traer su valor. `HKEYS`/`HVALS` traen solo nombres o solo valores.

> Guardar `nombre`, `edad` y `ciudad` en un hash (`usuario:1`) es más eficiente en memoria que tres claves sueltas, y permite borrar o expirar todo el registro con una sola clave.

---

## Sorted Sets (ZSets)

Como un set, pero cada miembro tiene una **puntuación (score)** numérica que mantiene el conjunto siempre ordenado. Ideal para rankings y leaderboards.

```bash
ZADD puntuaciones 1200 "ana" 980 "luis" 1500 "marta"
ZRANGE puntuaciones 0 -1 WITHSCORES
ZREVRANGE puntuaciones 0 2 WITHSCORES
```
`ZADD` agrega o actualiza miembros con su score. `ZRANGE` devuelve de menor a mayor (`WITHSCORES` incluye el puntaje). `ZREVRANGE` de mayor a menor — el orden típico de un ranking.

```bash
ZSCORE puntuaciones "ana"
ZRANK puntuaciones "ana"
ZINCRBY puntuaciones 50 "ana"
ZCARD puntuaciones
ZREM puntuaciones "luis"
```
`ZSCORE` da el puntaje de un miembro. `ZRANK` da su posición en orden ascendente (`ZREVRANK` en descendente). `ZINCRBY` suma puntos de forma atómica — el comando típico para actualizar un puntaje sin leer-calcular-escribir. `ZCARD` cuenta, `ZREM` elimina.

```bash
ZRANGEBYSCORE puntuaciones 1000 2000
```
Trae los miembros cuyo score cae en un rango — útil cuando importa el valor del score, no la posición.

---

## Persistencia (RDB y AOF)

Redis trabaja en memoria, pero puede guardar datos en disco para no perderlos ante un reinicio.

**RDB** genera una foto completa del dataset (`dump.rdb`):

```bash
SAVE
BGSAVE
LASTSAVE
```
`SAVE` es **bloqueante** (casi no se usa en producción). `BGSAVE` genera el snapshot en un proceso hijo, sin bloquear. `LASTSAVE` da el timestamp del último snapshot exitoso, útil para monitoreo.

En `redis.conf` se configuran reglas automáticas, por ejemplo `save 900 1` (snapshot si pasaron 900s y hubo ≥1 cambio). **Riesgo de RDB**: si el proceso muere entre dos snapshots, se pierden los cambios posteriores al último.

**AOF** registra cada escritura en un log, en orden:

```bash
CONFIG SET appendonly yes
BGREWRITEAOF
```
`CONFIG SET appendonly yes` activa AOF en caliente. `BGREWRITEAOF` compacta el archivo (que crece con el tiempo) en segundo plano.

| `appendfsync` | Cuándo escribe a disco | Riesgo de pérdida |
|---|---|---|
| `always` | Después de cada escritura | Mínimo, pero más lento |
| `everysec` (default recomendado) | Una vez por segundo | Hasta 1 segundo |
| `no` | Cuando decide el SO | Puede perder varios segundos |

RDB es más compacto y rápido de cargar al arrancar; AOF es más seguro pero pesa más. Muchos sistemas en producción usan **ambos**: RDB para backups portables, AOF como red de seguridad ante un crash.

---

## Replicación

Un primario recibe todas las escrituras; una o más réplicas copian sus datos y pueden atender lecturas.

```bash
REPLICAOF 192.168.1.10 6379
```
Convierte al servidor en réplica del que está en esa IP/puerto; empieza a sincronizar y recibe en tiempo real cada escritura.

```bash
REPLICAOF NO ONE
```
Corta la relación y convierte al nodo en primario independiente (por ejemplo, tras promover manualmente una réplica).

```bash
INFO replication
ROLE
```
`INFO replication` muestra si el nodo es `master`/`slave`, cuántas réplicas tiene y su lag. `ROLE` da una respuesta compacta pensada para scripts.

> La replicación es **asíncrona** por defecto: el primario no espera confirmación de la réplica antes de responder al cliente. Si el primario cae justo después de confirmar un `SET`, esa escritura puede no haber llegado a la réplica. Para forzar confirmación de N réplicas antes de continuar, existe `WAIT numreplicas timeout`.

Sirve para alta disponibilidad (promover una réplica si el primario cae), separar lectura/escritura, y sacar backups (`BGSAVE`) desde una réplica sin afectar al servidor que sirve tráfico real.

---

## Pub/Sub

```bash
SUBSCRIBE noticias
```
Bloquea la conexión: el cliente queda escuchando el canal `noticias` y recibe cada mensaje publicado, sin hacer polling.

```bash
PUBLISH noticias "nuevo artículo publicado"
```
Desde otra conexión, publica en el canal. Devuelve cuántos suscriptores lo recibieron (0 si nadie escuchaba).

```bash
PSUBSCRIBE noticias.*
```
Suscripción por patrón: recibe mensajes de cualquier canal que matchee, sin suscribirse a cada uno por separado.

> Pub/Sub **no guarda historial**: si publicás y no hay nadie suscrito en ese momento, el mensaje se pierde para siempre. Sirve para notificaciones en vivo, pero no como cola confiable. Para eso, Redis ofrece **Streams** (`XADD`/`XREAD`), que sí persisten.

---

## Transacciones (MULTI/EXEC)

```bash
MULTI
SET saldo:1 100
INCRBY saldo:1 -20
EXEC
```
`MULTI` marca el inicio: los comandos siguientes no se ejecutan de inmediato, se **encolan**. `EXEC` los ejecuta todos seguidos, sin que otro cliente pueda meterse en el medio. Es atomicidad de ejecución, no aislamiento tipo SQL: no hay rollback si un comando falla en tiempo de ejecución. `DISCARD` cancela la transacción sin ejecutar nada.

```bash
WATCH saldo:1
MULTI
DECRBY saldo:1 20
EXEC
```
`WATCH` agrega bloqueo optimista: si otro cliente modifica `saldo:1` entre el `WATCH` y el `EXEC`, la transacción se cancela sola (`EXEC` devuelve `nil`). Así, si dos procesos leen el mismo valor para calcular algo, el segundo se entera de que el dato cambió y puede reintentar en vez de pisar el trabajo del primero.

---

## Scripts Lua

Para lógica más compleja que unos pocos comandos encolados, Redis ejecuta scripts Lua **dentro** del servidor, de forma atómica.

```bash
EVAL "return redis.call('GET', KEYS[1])" 1 saldo:1
```
El primer argumento es el código Lua, el número indica cuántas claves recibe el script, y `KEYS[1]` apunta a `saldo:1`.

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
Descuenta saldo solo si alcanza, todo en una operación atómica — evita el problema de "leer, decidir, escribir" en varios viajes de red donde otro cliente podría meterse en el medio. `ARGV[1]` es el primer argumento no-clave (`20`).

```bash
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
EVALSHA <sha1-devuelto> 1 saldo:1
```
`SCRIPT LOAD` sube el script una vez y devuelve su hash SHA1. `EVALSHA` lo ejecuta usando el hash, sin reenviar el código completo cada vez.

---

## Cluster

Cuando una sola instancia no basta, Redis escala horizontalmente con **Redis Cluster**, que reparte las claves entre nodos mediante particionado (sharding) en **16384 slots** (`CRC16(clave) % 16384`).

```bash
CLUSTER INFO
CLUSTER NODES
CLUSTER KEYSLOT saldo:1
```
`CLUSTER INFO` muestra el estado general (`ok` o degradado). `CLUSTER NODES` lista los nodos y qué slots maneja cada uno. `CLUSTER KEYSLOT` dice a qué slot/nodo pertenece una clave — útil para debuggear.

> Comandos que tocan varias claves a la vez (`MGET a b`) solo funcionan en cluster si todas caen en el **mismo slot**. Se fuerza con "hash tags": `{usuario:1}:nombre` y `{usuario:1}:edad` van al mismo slot porque Redis solo hashea la parte entre `{}`.

Antes de elegir cluster, réplicas o una instancia sola, conviene responder: ¿la carga es de lectura o escritura?, ¿qué pérdida de datos es aceptable ante un crash?, ¿se necesita consistencia fuerte o latencia mínima?, ¿las claves se reparten uniformemente o hay unas pocas "calientes"? Cluster no es la respuesta a todo: una instancia bien configurada con réplicas y buena persistencia suele alcanzar, con mucha menos complejidad operativa.

---

## Buenas prácticas

- **Nombres de claves consistentes**: un esquema tipo `entidad:id:campo` (`usuario:42:nombre`) hace predecibles el `SCAN` y el debugging.
- **Evitar `KEYS *` en producción**, usar `SCAN`:
  ```bash
  SCAN 0 MATCH usuario:* COUNT 100
  ```
  Recorre el keyspace de a poco con un cursor (el `0` inicial, que la respuesta actualiza), sin bloquear el servidor. Se llama repetidas veces hasta que el cursor vuelve a ser `0`.
- **TTL a los datos temporales**: cualquier dato de caché debería tener `EXPIRE`; sin TTL, una caché "que nunca se limpia sola" termina llenando la memoria.
- **No serializar objetos enormes en una sola clave**: mejor partirlos o usar un hash.
- **Medir memoria y estructura real de una clave**:
  ```bash
  MEMORY USAGE usuario:1
  OBJECT ENCODING usuario:1
  ```
  `MEMORY USAGE` dice cuántos bytes ocupa una clave puntual. `OBJECT ENCODING` muestra la representación interna elegida (ej. un hash chico como `listpack`, muy compacto; uno grande como `hashtable`).
- **Diagnosticar comandos lentos**:
  ```bash
  SLOWLOG GET 10
  ```
  Muestra los últimos comandos que superaron el umbral de latencia configurado — el primer lugar donde mirar si Redis "se puso lento".
- **Separar caché, sesiones, colas y métricas**, aunque compartan servidor, con prefijos de clave distintos.

Regla general: guardá en Redis lo que necesites leer o actualizar con mucha frecuencia y se beneficie de una estructura simple y rápida. Si el dato es grande, se consulta poco o necesita queries complejas (joins, filtros arbitrarios), probablemente pertenece a una base relacional.

---

## Mini proyectos

**Caché de perfiles**
```bash
SETEX user:42 300 '{"nombre":"Ana","rol":"admin"}'
GET user:42
```
Guarda el perfil serializado con expiración de 5 minutos. Patrón típico de caché: si no está, se recalcula desde la base real y se vuelve a cachear.

**Contador de visitas**
```bash
INCR visitas:home
```
Un solo comando atómico: aunque lo llamen miles de requests concurrentes, nunca se pierde un conteo.

**Cola de trabajos**
```bash
LPUSH queue:emails "email-001"
RPOP queue:emails
```
Productor hace `LPUSH`, worker hace `RPOP` → cola FIFO. Para que el worker no haga polling constante, existe `BRPOP queue:emails 0`, que bloquea hasta que aparezca un elemento nuevo.

**Ranking de jugadores**
```bash
ZINCRBY ranking 25 "jugador:7"
ZRANGE ranking 0 9 WITHSCORES
```
`ZINCRBY` suma puntos de forma atómica; `ZRANGE ranking 0 9` trae el top 10 siempre ordenado, sin que la aplicación tenga que ordenar nada.

**Rate limiting simple**
```bash
INCR ratelimit:usuario42
EXPIRE ratelimit:usuario42 60
```
Cada request incrementa el contador; el primer `INCR` también crea la clave, a la que se le pone `EXPIRE 60`. Si supera el límite permitido antes de que pasen los 60 segundos, la aplicación rechaza la request — la base de casi cualquier rate limiter por ventana fija.

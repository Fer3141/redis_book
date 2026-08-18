# 2. Primeros comandos y tipos de datos

Redis habla en términos de comandos: `COMANDO clave [argumentos...]`. Todo comando opera sobre una **clave** (un string, como `"usuario:42"` o `"saldo:1"`) y casi todos son atómicos: mientras se ejecutan, ningún otro cliente puede interferir a la mitad.

## Comandos generales sobre claves

```bash
SET saludo "hola"
GET saludo
DEL saludo
EXISTS saludo
```

- `SET clave valor` crea o sobreescribe una clave sin preguntar si existía antes.
- `GET clave` devuelve el valor, o `nil` si la clave no existe (esto es importante: `nil` no es lo mismo que un string vacío `""`).
- `DEL clave` la elimina y devuelve cuántas claves borró realmente (0 si no existía). Podés borrar varias a la vez: `DEL a b c`.
- `EXISTS clave` devuelve `1` o `0`. También acepta varias claves y suma cuántas de ellas existen.

```bash
KEYS *
TYPE saludo
TTL saludo
EXPIRE saludo 60
```

- `KEYS patron` busca claves que matchean un patrón (`KEYS user:*`). **Evitalo en producción**: recorre todo el keyspace de forma bloqueante y con muchas claves puede congelar el servidor por segundos (ver capítulo 5, ahí está la alternativa: `SCAN`).
- `TYPE clave` te dice qué estructura de datos es (`string`, `list`, `set`, `hash`, `zset`, `stream`...). Sirve para no operar con el comando equivocado sobre una clave (por ejemplo, hacer `LPUSH` sobre algo que en realidad es un string tira error `WRONGTYPE`).
- `TTL clave` (time to live) muestra los segundos que le quedan de vida a una clave con expiración. Los tres valores posibles son:
  - un número positivo: segundos restantes,
  - `-1`: la clave existe pero **no** tiene expiración configurada,
  - `-2`: la clave no existe (ya expiró o nunca existió).
- `EXPIRE clave segundos` le pone (o reemplaza) un tiempo de vida a una clave ya existente. Pasado ese tiempo, Redis la borra automáticamente.

```bash
PERSIST saludo
RENAME saludo saludo2
COPY saludo2 saludo3
```

- `PERSIST clave` le quita el TTL a una clave, la vuelve permanente otra vez.
- `RENAME origen destino` renombra una clave. Si `destino` ya existía, la pisa sin avisar (por eso existe `RENAMENX`, que solo renombra si el destino no existe).
- `COPY origen destino` duplica el valor de una clave en otra, sin borrar la original.

## Strings

El tipo más simple: texto, números, o cualquier valor serializado (JSON, por ejemplo). Un string en Redis puede pesar hasta 512 MB.

```bash
SET contador 10
INCR contador
INCRBY contador 5
DECR contador
DECRBY contador 3
```

- `INCR`/`DECR` suman o restan 1 de forma **atómica**. Esto es clave: si dos procesos hacen `INCR` al mismo tiempo sobre el mismo contador, nunca se pisan entre sí ni pierden una actualización (a diferencia de hacer `GET` + sumar en tu código + `SET`, que sí tiene una condición de carrera).
- Si la clave no existe, Redis la trata como si valiera `0` antes de incrementar.
- Si el valor no es un número entero, `INCR` devuelve error.

```bash
SET saldo:1 100 EX 60
SETEX saldo:1 60 100
```

- `SET clave valor EX segundos` crea la clave con expiración en un solo comando (evita el viaje extra de hacer `SET` y después `EXPIRE`).
- `SETEX clave segundos valor` hace lo mismo, con otro orden de argumentos: **primero los segundos, después el valor**. Es un error común invertir el orden.

```bash
APPEND log "primera línea\n"
STRLEN log
MSET a 1 b 2 c 3
MGET a b c
```

- `APPEND` agrega texto al final de un string existente (o lo crea si no existía). Útil para logs simples o buffers.
- `STRLEN` devuelve el largo en bytes del valor guardado.
- `MSET`/`MGET` escriben o leen varias claves en un solo viaje de red, más eficiente que hacer varios `SET`/`GET` seguidos.

## Listas

Secuencias ordenadas de elementos. Internamente son listas enlazadas optimizadas, ideales para colas y pilas simples (no para acceso aleatorio por índice a gran escala).

```bash
LPUSH tareas "enviar correo"
RPUSH tareas "generar informe"
LRANGE tareas 0 -1
LPOP tareas
RPOP tareas
```

- `LPUSH` inserta al **principio** de la lista (izquierda); `RPUSH` inserta al **final** (derecha).
- `LRANGE clave inicio fin` lee un rango. `0 -1` es el modismo para "toda la lista" (`-1` significa "el último elemento").
- `LPOP`/`RPOP` sacan y devuelven el primer/último elemento. Combinando `RPUSH` + `LPOP` desde otro proceso, obtenés una **cola FIFO** (primero en entrar, primero en salir): el patrón típico de una cola de trabajos simple.

```bash
LLEN tareas
LINSERT tareas BEFORE "generar informe" "revisar informe"
LSET tareas 0 "enviar correo urgente"
```

- `LLEN` devuelve cuántos elementos tiene la lista.
- `LINSERT` inserta un elemento antes o después de otro valor puntual (recorre la lista para encontrarlo, así que en listas muy grandes es más lento que `LPUSH`/`RPUSH`).
- `LSET` reemplaza el elemento en una posición (índice) específica.

## Conjuntos (Sets)

Colecciones de valores **únicos**, sin orden garantizado. Pensalos como el equivalente a un `Set` de cualquier lenguaje de programación, pero persistido.

```bash
SADD tags "redis" "base-datos" "cache"
SMEMBERS tags
SISMEMBER tags "redis"
SREM tags "cache"
SCARD tags
```

- `SADD` agrega uno o más elementos; si alguno ya estaba, lo ignora en silencio (por eso "conjunto": no hay duplicados).
- `SISMEMBER clave valor` responde `1`/`0` en tiempo **O(1)** — es mucho más eficiente que traer todo el set con `SMEMBERS` y buscar el valor en tu código. Es el uso más común en la práctica: comprobar pertenencia rápido (¿este usuario ya votó?, ¿este email ya está en la lista negra?).
- `SCARD` devuelve la cantidad de elementos (cardinalidad) sin traer el contenido.

```bash
SADD equipoA "ana" "luis"
SADD equipoB "luis" "marta"
SINTER equipoA equipoB
SUNION equipoA equipoB
SDIFF equipoA equipoB
```

- `SINTER` (intersección): elementos que están en ambos sets → `luis`.
- `SUNION` (unión): todos los elementos combinados sin duplicar.
- `SDIFF` (diferencia): lo que está en el primer set pero no en el segundo → `ana`.

## Hashes

Un hash es como un objeto/diccionario: agrupa varios campos bajo una misma clave. Es la forma natural de guardar "un registro", por ejemplo un usuario, sin serializar todo a JSON.

```bash
HSET usuario:1 nombre "Ana" edad 29 ciudad "Madrid"
HGET usuario:1 nombre
HGETALL usuario:1
HINCRBY usuario:1 edad 1
```

- `HSET clave campo valor [campo valor ...]` puede setear varios campos en un solo comando.
- `HGET` trae un campo puntual (más eficiente que traer todo el hash si solo necesitás un dato).
- `HGETALL` trae todos los campos y valores. Cuidado con hashes muy grandes: trae todo de una vez.
- `HINCRBY` incrementa un campo numérico dentro del hash de forma atómica, igual que `INCR` pero a nivel de campo.

```bash
HDEL usuario:1 ciudad
HEXISTS usuario:1 nombre
HKEYS usuario:1
HVALS usuario:1
```

- `HDEL` borra uno o más campos (no la clave entera).
- `HEXISTS` comprueba si un campo puntual existe, sin traer su valor.
- `HKEYS`/`HVALS` traen solo los nombres de los campos o solo los valores, respectivamente.

**Cuándo usar hash vs. strings separados**: si tenés `nombre`, `edad` y `ciudad` de un usuario, guardarlos en un hash (`usuario:1`) es más eficiente en memoria que tres claves sueltas (`usuario:1:nombre`, `usuario:1:edad`, ...), y te deja borrar o expirar todo el registro con una sola clave.

## Sorted Sets (ZSets)

Como un set, pero cada miembro tiene asociada una **puntuación (score)** numérica que Redis usa para mantenerlo siempre ordenado. Es la estructura ideal para rankings, leaderboards y colas con prioridad.

```bash
ZADD puntuaciones 1200 "ana" 980 "luis" 1500 "marta"
ZRANGE puntuaciones 0 -1 WITHSCORES
ZREVRANGE puntuaciones 0 2 WITHSCORES
```

- `ZADD clave score miembro [score miembro ...]` agrega o actualiza miembros con su puntuación.
- `ZRANGE clave 0 -1` devuelve todos los miembros ordenados de menor a mayor score. `WITHSCORES` incluye el puntaje en la salida.
- `ZREVRANGE` hace lo mismo pero de mayor a menor — el orden típico de un ranking ("top jugadores").

```bash
ZSCORE puntuaciones "ana"
ZRANK puntuaciones "ana"
ZINCRBY puntuaciones 50 "ana"
ZCARD puntuaciones
ZREM puntuaciones "luis"
```

- `ZSCORE` devuelve el puntaje de un miembro puntual.
- `ZRANK` devuelve la posición (índice, empezando en 0) de un miembro dentro del orden ascendente. `ZREVRANK` da la posición en el ranking descendente (posición 0 = el primero del top).
- `ZINCRBY clave incremento miembro` suma puntos a un miembro existente de forma atómica — el comando típico para actualizar un puntaje ("sumale 50 puntos a Ana") sin tener que leer, calcular y volver a escribir.
- `ZCARD` cuenta miembros. `ZREM` elimina un miembro del set.

```bash
ZRANGEBYSCORE puntuaciones 1000 2000
```
Trae solo los miembros cuyo score cae en un rango — por ejemplo, "todos los jugadores entre 1000 y 2000 puntos". Es más útil que `ZRANGE` por posición cuando lo que te importa es el valor del score, no en qué lugar del ranking están.

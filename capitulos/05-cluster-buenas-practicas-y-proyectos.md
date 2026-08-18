# 5. Cluster, buenas prácticas y mini proyectos

## Redis Cluster

Cuando una sola instancia no basta (por memoria o por throughput), Redis puede escalarse horizontalmente con **Redis Cluster**, que divide las claves entre varios nodos mediante particionado (sharding).

Redis Cluster reparte el espacio de claves en **16384 slots**. Cada clave se asigna a un slot mediante un hash de su nombre (`CRC16(clave) % 16384`), y cada nodo del cluster es responsable de un rango de slots.

```bash
CLUSTER INFO
CLUSTER NODES
CLUSTER KEYSLOT saldo:1
```

- `CLUSTER INFO` muestra el estado general del cluster: si está `ok` o degradado, cuántos slots están asignados, etc.
- `CLUSTER NODES` lista los nodos del cluster y qué rango de slots maneja cada uno.
- `CLUSTER KEYSLOT clave` te dice a qué slot (y por lo tanto, a qué nodo) pertenece una clave específica — útil para debuggear por qué una operación con varias claves falló.

**Detalle práctico importante**: comandos que tocan varias claves a la vez (como `MGET a b`) solo funcionan en cluster si todas esas claves caen en el **mismo slot**. Para forzar eso, se usa la convención de "hash tags": `{usuario:1}:nombre` y `{usuario:1}:edad` van al mismo slot porque Redis solo hashea la parte entre `{}`. Es la forma de garantizar que datos relacionados queden en el mismo nodo.

## Preguntas antes de diseñar

Antes de elegir cluster, réplicas, o una instancia sola, conviene responder:

- ¿La carga principal es de lectura o de escritura? (lectura → replicas ayudan mucho; escritura → cluster reparte mejor)
- ¿Qué nivel de pérdida de datos es aceptable ante un crash? (define RDB vs AOF vs ambos, ver capítulo 3)
- ¿Se necesita consistencia fuerte o latencia mínima? (replicación asíncrona prioriza latencia, no consistencia estricta)
- ¿Las claves se pueden repartir de forma uniforme, o hay unas pocas claves "calientes" que concentran todo el tráfico? (una clave muy usada no se beneficia de cluster, porque siempre cae en el mismo nodo)

Cluster no es la respuesta a todo: a veces una instancia bien configurada con réplicas y buena persistencia alcanza y sobra, con mucha menos complejidad operativa.

## Buenas prácticas, explicadas

- **Nombres de claves consistentes**: usar un esquema tipo `entidad:id:campo` (`usuario:42:nombre`) hace que `SCAN` por patrón y el debugging sean predecibles.
- **Evitar `KEYS *` en producción**, y usar `SCAN` en su lugar:
  ```bash
  SCAN 0 MATCH usuario:* COUNT 100
  ```
  `SCAN` recorre el keyspace de a poco usando un cursor (el `0` inicial, que la respuesta va actualizando), sin bloquear el servidor como hace `KEYS`. Se llama repetidas veces pasando el cursor devuelto hasta que este vuelve a ser `0`.
- **TTL a los datos temporales**: cualquier dato de caché debería tener `EXPIRE`. Sin TTL, una caché que "nunca se limpia sola" termina llenando la memoria disponible.
- **No serializar objetos enormes sin necesidad**: un JSON de varios MB en una sola clave es lento de traer entero y de deserializar. Mejor partirlo en varias claves o usar un hash.
- **Medir memoria y estructura real de una clave**:
  ```bash
  MEMORY USAGE usuario:1
  OBJECT ENCODING usuario:1
  ```
  `MEMORY USAGE` dice cuántos bytes ocupa una clave puntual — útil para encontrar las claves más pesadas. `OBJECT ENCODING` muestra la representación interna que Redis eligió (por ejemplo, un hash chico se guarda como `listpack`, muy compacto; uno grande pasa a `hashtable`). Sirve para entender por qué una estructura que "debería pesar poco" está consumiendo más memoria de la esperada.
- **Diagnosticar comandos lentos**:
  ```bash
  SLOWLOG GET 10
  ```
  Muestra los últimos comandos que superaron el umbral de latencia configurado (`slowlog-log-slower-than`). Es el primer lugar donde mirar si Redis "se puso lento".
- **Separar claramente caché, sesiones, colas y métricas**, aunque compartan el mismo servidor — por ejemplo, con prefijos de clave distintos o incluso bases lógicas distintas (`SELECT`), para poder medir y limpiar cada cosa por separado.

Una buena regla general: guardá en Redis lo que necesites leer o actualizar con mucha frecuencia y se beneficie de una estructura simple y rápida. Si el dato es grande, se consulta poco, o necesita queries complejas (joins, filtros arbitrarios), probablemente pertenece a una base relacional, no a Redis.

## Mini proyectos

### Caché de perfiles

```bash
SETEX user:42 300 '{"nombre":"Ana","rol":"admin"}'
GET user:42
```
`SETEX clave segundos valor` guarda el perfil serializado con expiración de 300 segundos (5 minutos). Pasado ese tiempo, Redis lo borra solo — típico patrón de caché: si no está, se recalcula desde la base de datos real y se vuelve a cachear.

### Contador de visitas

```bash
INCR visitas:home
```
Un solo comando atómico. Aunque lo llamen miles de requests concurrentes, nunca se pierde un conteo (a diferencia de leer-sumar-guardar en el código de la aplicación).

### Cola de trabajos

```bash
LPUSH queue:emails "email-001"
RPOP queue:emails
```
Un proceso productor hace `LPUSH` (agrega al principio), un proceso worker hace `RPOP` (saca del final) → cola FIFO. Para que el worker no esté haciendo polling constante con `RPOP`, existe `BRPOP queue:emails 0`, que bloquea la conexión hasta que aparezca un elemento nuevo, sin gastar CPU en reintentos.

### Ranking de jugadores

```bash
ZINCRBY ranking 25 "jugador:7"
ZRANGE ranking 0 9 WITHSCORES
```
`ZINCRBY` suma puntos de forma atómica; `ZRANGE ranking 0 9` trae el top 10 (posiciones 0 a 9) siempre ordenado, sin que la aplicación tenga que ordenar nada por su cuenta.

### Rate limiting simple

```bash
INCR ratelimit:usuario42
EXPIRE ratelimit:usuario42 60
```
Cada request de ese usuario incrementa el contador; el primer `INCR` también crea la clave con `EXPIRE 60`. Si el contador supera el límite permitido (por ejemplo, 100) antes de que pasen los 60 segundos, la aplicación rechaza la request. Es la base de casi cualquier rate limiter simple por ventana fija.

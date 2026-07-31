# 2. Primeros comandos y tipos de datos

Redis habla en términos de comandos. La experiencia suele empezar con operaciones de lectura y escritura sobre claves.

```bash
SET saludo "hola"
GET saludo
DEL saludo
EXISTS saludo
```

Algunos comandos útiles al principio:

```bash
KEYS *
TYPE saludo
TTL saludo
EXPIRE saludo 60
```

`TTL` muestra el tiempo restante de vida de una clave. Si el valor es `-1`, la clave no expira. Si es `-2`, ya no existe.

Redis no almacena solo cadenas. Su fortaleza está en sus estructuras nativas.

## Cadenas

Las cadenas son el tipo más simple. Se usan para texto, números y valores serializados.

```bash
SET contador 10
INCR contador
INCRBY contador 5
DECR contador
```

## Listas

Las listas son secuencias ordenadas de elementos. Funcionan bien para colas simples.

```bash
LPUSH tareas "enviar correo"
RPUSH tareas "generar informe"
LRANGE tareas 0 -1
LPOP tareas
RPOP tareas
```

## Conjuntos

Los conjuntos guardan valores únicos sin orden garantizado.

```bash
SADD tags "redis" "base-datos" "cache"
SMEMBERS tags
SISMEMBER tags "redis"
SREM tags "cache"
```

## Hashes

Los hashes permiten agrupar campos relacionados bajo una misma clave.

```bash
HSET usuario:1 nombre "Ana" edad 29 ciudad "Madrid"
HGET usuario:1 nombre
HGETALL usuario:1
HINCRBY usuario:1 edad 1
```

## Sorted Sets

Los conjuntos ordenados asocian cada miembro con una puntuación. Son perfectos para rankings y líderes.

```bash
ZADD puntuaciones 1200 "ana" 980 "luis" 1500 "marta"
ZRANGE puntuaciones 0 -1 WITHSCORES
ZREVRANGE puntuaciones 0 2 WITHSCORES
```

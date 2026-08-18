# 3. Persistencia y replicación

Redis trabaja en memoria, pero también puede guardar datos en disco. Así se consigue equilibrio entre velocidad y durabilidad: si el proceso se reinicia o el servidor se apaga, los datos no desaparecen.

## RDB (snapshots)

`RDB` genera una **foto completa** de todo el dataset en un momento dado, guardada en un archivo binario (`dump.rdb`).

```bash
SAVE
BGSAVE
LASTSAVE
```

- `SAVE` genera el snapshot de forma **bloqueante**: el servidor no atiende otros comandos hasta terminar. No se usa casi nunca en producción por eso.
- `BGSAVE` hace lo mismo pero en un proceso hijo (fork), sin bloquear al servidor principal. Es la forma real de generar un snapshot manual.
- `LASTSAVE` devuelve el timestamp Unix del último `SAVE`/`BGSAVE` exitoso — útil para monitoreo, para confirmar que los backups automáticos realmente están corriendo.

En `redis.conf` (o con `CONFIG SET save "..."`), se configuran reglas automáticas, por ejemplo:
```
save 900 1
save 300 10
```
Esto significa "guardá un snapshot si pasaron 900 segundos y hubo al menos 1 cambio, **o** si pasaron 300 segundos y hubo al menos 10 cambios". Podés tener varias reglas combinadas.

**Riesgo real de RDB**: si el proceso muere entre dos snapshots, perdés todos los cambios que ocurrieron después del último snapshot guardado. Para una caché eso no importa; para datos de negocio, sí.

## AOF (Append Only File)

`AOF` registra **cada operación de escritura** en un log, en el orden en que ocurrió. Al reiniciar, Redis reconstruye el estado reproduciendo ese log.

```bash
CONFIG SET appendonly yes
BGREWRITEAOF
```

- `CONFIG SET appendonly yes` activa AOF en caliente sin reiniciar el servidor.
- `BGREWRITEAOF` compacta el archivo AOF (que puede crecer mucho con el tiempo) generando una versión mínima equivalente, en segundo plano.

La política de cuándo escribir a disco (`appendfsync`) tiene tres modos, con un trade-off directo entre velocidad y seguridad:

| Modo | Cuándo escribe a disco | Riesgo de pérdida |
|---|---|---|
| `always` | Después de cada comando de escritura | Prácticamente ninguna, pero es el más lento |
| `everysec` | Una vez por segundo (el default recomendado) | Hasta 1 segundo de datos en el peor caso |
| `no` | Cuando decide el sistema operativo | Puede perder varios segundos de datos |

En la práctica, `everysec` es el punto de equilibrio que usa casi todo el mundo.

**RDB vs. AOF**: RDB es más compacto y rápido de cargar al arrancar; AOF es más seguro (pierde menos datos ante un corte) pero el archivo pesa más y tarda más en reconstruirse. Muchos sistemas en producción usan **ambos** a la vez: RDB para backups/snapshots portables, AOF como red de seguridad ante un crash.

## Replicación

Un servidor primario recibe todas las escrituras; uno o más réplicas copian sus datos y pueden atender lecturas, repartiendo la carga.

```bash
REPLICAOF 192.168.1.10 6379
```
Ejecutado en un servidor, lo convierte en réplica del servidor en esa IP y puerto. A partir de ahí, empieza a sincronizar y recibe en tiempo real cada escritura que ocurre en el primario.

```bash
REPLICAOF NO ONE
```
Corta la relación de réplica y convierte a ese nodo en un primario independiente (por ejemplo, durante una promoción manual tras la caída del primario original).

```bash
INFO replication
ROLE
```
- `INFO replication` muestra si el nodo es `master` o `slave`, cuántas réplicas tiene conectadas, y cuánto "atraso" (lag) tiene cada una respecto al primario.
- `ROLE` da una respuesta más compacta y pensada para scripts: el rol actual del nodo y el estado de sincronización.

**Punto importante para bajarlo a tierra**: la replicación en Redis es **asíncrona** por defecto. El primario no espera a que la réplica confirme haber recibido una escritura antes de responderle "OK" al cliente. Esto significa que, si el primario se cae justo después de confirmar un `SET`, es posible que esa escritura nunca haya llegado a la réplica — y se pierda si la réplica es promovida a primario. Para casos donde eso es inaceptable, existe `WAIT numreplicas timeout`, que fuerza a esperar confirmación de al menos N réplicas antes de continuar.

La replicación sirve para:
- **Alta disponibilidad**: si el primario cae, una réplica puede promoverse y seguir sirviendo tráfico.
- **Separar lectura y escritura**: las réplicas pueden atender `GET`s mientras el primario se dedica a escrituras, repartiendo carga.
- **Backups sin tocar el primario**: correr `BGSAVE` sobre una réplica no afecta la latencia del servidor que sirve tráfico real.

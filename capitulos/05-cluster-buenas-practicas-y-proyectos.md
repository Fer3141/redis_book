# 5. Cluster, buenas prácticas y mini proyectos

Cuando una sola instancia no basta, Redis puede escalarse con estrategias distintas.

La opción más conocida es Redis Cluster, que divide las claves entre varios nodos mediante particionado. Así se distribuye la carga y se mantiene disponibilidad ante fallos parciales.

En el diseño de un sistema conviene responder primero estas preguntas:

- ¿La carga principal es de lectura o de escritura?
- ¿Qué nivel de pérdida de datos es aceptable?
- ¿Se necesita consistencia fuerte o latencia mínima?
- ¿Las claves se pueden repartir de forma uniforme?

Cluster no es la respuesta a todo. A veces una instancia bien configurada con réplicas y persistencia basta. Otras veces la aplicación necesita particionar por dominio, no por infraestructura.

Redis es rápido, pero también puede ser costoso si se usa sin disciplina.

- Usa nombres de claves consistentes y predecibles.
- Evita `KEYS *` en producción; puede bloquear instancias grandes.
- Pon TTL a los datos temporales.
- No serialices objetos enormes sin necesidad.
- Mide memoria, latencia y cardinalidad de claves.
- Diseña pensando en el patrón de acceso, no solo en el tipo de dato.
- Separa claramente caché, sesiones, colas y métricas.

Una buena regla es esta: guarda en Redis aquello que necesites leer o actualizar con frecuencia y que se beneficie de una estructura simple y rápida.

## Mini proyectos

### Caché de perfiles

```bash
SETEX user:42 '{"nombre":"Ana","rol":"admin"}' 300
GET user:42
```

### Contador de visitas

```bash
INCR visitas:home
```

### Cola de trabajos

```bash
LPUSH queue:emails "email-001"
RPOP queue:emails
```

### Ranking de jugadores

```bash
ZINCRBY ranking 25 "jugador:7"
ZRANGE ranking 0 9 WITHSCORES
```

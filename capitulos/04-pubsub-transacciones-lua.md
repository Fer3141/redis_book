# 4. Pub/Sub, transacciones y Lua

Redis también funciona como sistema de publicación y suscripción. Un cliente publica mensajes en un canal y otros clientes se suscriben para recibirlos.

```bash
SUBSCRIBE noticias
PUBLISH noticias "nuevo artículo publicado"
```

Pub/Sub es útil para eventos en tiempo real, notificaciones y coordinación ligera entre servicios. No es un sistema de mensajes persistentes; si nadie escucha en el momento de la publicación, el mensaje se pierde.

Redis permite agrupar comandos en una transacción con `MULTI` y `EXEC`.

```bash
MULTI
SET saldo:1 100
INCRBY saldo:1 -20
EXEC
```

Las transacciones ayudan a ejecutar varias operaciones de manera ordenada. No sustituyen una base de datos relacional con aislamiento completo, pero sí son muy útiles para secuencias consistentes de comandos.

Cuando necesitas lógica más compleja, Redis puede ejecutar scripts Lua. Eso permite mover la lógica al servidor y reducir viajes de ida y vuelta.

Un caso típico es validar, calcular y actualizar en una sola operación atómica.

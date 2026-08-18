# 1. Qué es Redis y arranque

Redis es un almacén de datos **en memoria** (in-memory) que se usa como base de datos, caché, cola de trabajos y sistema de mensajería. Su nombre suele asociarse a rapidez, simplicidad y flexibilidad.

## Por qué es tan rápido (en términos concretos)

No es magia, son tres decisiones de diseño:

1. **Todo vive en RAM**, no en disco. Leer de memoria es ~100.000 veces más rápido que leer de un disco tradicional.
2. **Es de un solo hilo (single-threaded)** para ejecutar comandos. Suena contraintuitivo, pero evita el costo de locks y cambios de contexto entre threads: cada comando se ejecuta de punta a punta sin que otro lo interrumpa. Por eso comandos como `INCR` son **atómicos** sin que tengas que hacer nada especial.
3. **Sus estructuras de datos están pensadas para acceso O(1) o O(log n)**: un `GET` sobre una clave es prácticamente instantáneo sin importar cuántas claves tengas, porque internamente usa tablas hash.

La contrapartida: un solo comando muy pesado (por ejemplo `KEYS *` en una base con millones de claves) bloquea a todos los demás clientes mientras se ejecuta, porque no hay paralelismo dentro del motor. Esto se explica más en el capítulo 5.

## Para qué se usa en la práctica

El mismo servidor puede servir para varias cosas a la vez:

- **Caché**: guardar el resultado de una consulta cara (a una base SQL, a una API externa) por unos minutos.
- **Sesiones de usuario**: guardar el token/sesión de un login con expiración automática.
- **Contadores y rankings**: visitas a una página, puntuaciones de un juego.
- **Colas de trabajo**: una lista donde un proceso hace `LPUSH` y otro hace `RPOP` (cola FIFO simple).
- **Pub/Sub**: notificaciones en tiempo real entre servicios.
- **Rate limiting**: contar cuántas requests hizo un usuario en los últimos 60 segundos para bloquear abuso de una API.

## Instalación y arranque

La forma más rápida de probarlo, sin instalar nada en el sistema, es con Docker:

```bash
docker run --name redis-book -p 6379:6379 -d redis:7
```

- `-p 6379:6379` expone el puerto por defecto de Redis (6379) desde el contenedor hacia tu máquina.
- `-d` lo corre en segundo plano (detached).

Para conectarte al servidor dentro del contenedor:

```bash
docker exec -it redis-book redis-cli
```

Si Redis ya está instalado localmente (Linux/macOS, o Windows vía WSL), arrancás el servidor con:

```bash
redis-server
```

Y en otra terminal abrís el cliente interactivo:

```bash
redis-cli
```

## Primeros pasos dentro de `redis-cli`

Antes de meterte con comandos de datos, conviene verificar que todo funciona:

```bash
PING
# PONG
```
`PING` es el "hola mundo" de Redis: si responde `PONG`, el servidor está vivo y aceptando comandos.

```bash
INFO server
```
Muestra información del servidor: versión de Redis, modo (standalone/cluster/sentinel), uptime, cantidad de conexiones, etc. Es lo primero que se mira al diagnosticar un problema.

```bash
CONFIG GET maxmemory
```
Consulta el valor de un parámetro de configuración en caliente, sin tener que abrir el archivo `redis.conf`. Se puede combinar con `CONFIG SET maxmemory 256mb` para cambiarlo sin reiniciar el servidor.

```bash
SELECT 1
```
Redis en modo standalone tiene 16 bases lógicas numeradas (0 a 15) dentro del mismo proceso. `SELECT` cambia a cuál te conectás. En la práctica casi nadie usa más de la base 0 salvo para separar entornos de prueba rápido.

```bash
DBSIZE
```
Devuelve cuántas claves hay en la base seleccionada. Útil para saber si estás mirando la base vacía o la que tiene datos.

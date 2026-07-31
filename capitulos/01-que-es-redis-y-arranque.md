# 1. Qué es Redis y arranque

Redis es un almacén de datos en memoria que se usa como base de datos, caché, cola de trabajos y sistema de mensajería. Su nombre suele asociarse a rapidez, simplicidad y flexibilidad.

La idea central es sencilla: guardar datos cerca del proceso que los necesita, con acceso muy rápido. Por eso Redis suele aparecer en aplicaciones web, sistemas de analítica, colas de tareas y servicios que necesitan baja latencia.

Redis no obliga a elegir un único uso. El mismo servidor puede servir para:

- cachear respuestas frecuentes,
- guardar sesiones de usuario,
- coordinar workers,
- publicar eventos,
- mantener contadores y rankings,
- persistir estructuras simples con alta velocidad.

Redis puede instalarse en Linux, macOS, Windows mediante WSL o usando contenedores. En muchos casos, la forma más rápida de probarlo es con Docker:

```bash
docker run --name redis-book -p 6379:6379 redis:7
```

Para conectarte al servidor:

```bash
docker exec -it redis-book redis-cli
```

Si Redis ya está instalado localmente, puedes arrancarlo con:

```bash
redis-server
```

Y abrir otra terminal para usar el cliente:

```bash
redis-cli
```

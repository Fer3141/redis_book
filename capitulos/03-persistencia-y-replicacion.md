# 3. Persistencia y replicación

Redis trabaja en memoria, pero también puede guardar datos en disco. Así se consigue equilibrio entre velocidad y durabilidad.

Los dos mecanismos más conocidos son:

- RDB, que genera instantáneas del estado en momentos concretos.
- AOF, que registra operaciones de escritura para reconstruir la base de datos.

RDB suele ser eficiente para copias periódicas. AOF ofrece mayor detalle histórico de los cambios. En sistemas reales, muchas veces se combinan ambos.

La elección depende del caso de uso. Una caché agresiva puede tolerar cierta pérdida de datos, mientras que una cola de trabajo o un sistema de sesiones puede requerir más garantías.

Redis puede replicarse para mejorar disponibilidad y distribuir lectura. El servidor principal recibe escrituras y uno o más réplicas pueden copiar sus datos.

La replicación ayuda a:

- aumentar la tolerancia a fallos,
- separar lectura y escritura,
- preparar estrategias de recuperación.

En términos prácticos, una réplica suele configurarse para seguir a un nodo primario. Cuando el primario cambia, el sistema puede promover otra instancia según la arquitectura elegida.

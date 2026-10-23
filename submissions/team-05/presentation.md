# Presentación — team-05

**Duración máxima: 10 minutos (7 exposición + 3 preguntas).**

## 1. Incidente y objetivo (1 min)

NexaCloud estaba teniendo problemas porque algunos contenedores consumían demasiados recursos y eso afectaba los servicios.

Nuestro objetivo fue controlar el uso de CPU y memoria y mantener los contenedores funcionando.

## 2. Arquitectura y decisiones (2 min)

Trabajamos con Docker Swarm y usamos la imagen de Nginx.

Configuramos 2 réplicas, un límite de 0.50 de CPU y 128 MB de memoria por contenedor.

También agregamos reservas de recursos, reinicio automático cuando hay fallos y un healthcheck para revisar que el contenedor esté bien.

## 3. Demo: comandos y evidencia (3 min)

Primero usamos `docker stack deploy` para desplegar el servicio.

Después, con `docker stack services team05`, revisamos las réplicas.

Al principio nos aparecía 0/2, pero luego las dos estaban en estado Running.

Con `docker service inspect` podemos revisar los recursos y las configuraciones que aplicamos.

También usamos `docker stats` para mirar el consumo de CPU y memoria.

La prueba de recuperación consiste en detener un contenedor y revisar si Docker Swarm crea otro para reemplazarlo.

## 4. Limitaciones y decisiones (1 min)

La principal limitación es que usamos un solo nodo. Si ese nodo falla completamente, los servicios también pueden dejar de funcionar.

Para una empresa real sería mejor trabajar con varios nodos y tener un monitoreo más completo.

## 5. Preguntas (3 min)

**¿Qué problema resolvimos?**

Controlar el consumo de recursos de los contenedores.

**¿Cómo demostramos que funciona?**

Con los comandos de Docker que muestran el estado de las réplicas y la configuración del servicio.

**¿Qué aprendimos?**

Que Docker Swarm permite administrar varios contenedores, limitar sus recursos y recuperar tareas cuando se presentan fallos.



# Evidencias — team-05

## Diseño propuesto

En esta misión trabajamos con Docker Swarm para controlar los recursos de los contenedores y evitar que consuman demasiado.

Usamos nginx con 2 réplicas, límites de CPU de 0.50 y memoria de 128 MB. También configuramos reservas de recursos, reinicio automático y una prueba para revisar que el contenedor esté funcionando.

## Comandos de despliegue

```bash
docker stack deploy -c submissions/team-05/docker-stack.yml team05 
docker stack services team05
docker service ps team05_worker
docker service inspect team05_worker --pretty
```

## Experimento funcional

**Hipótesis:**

Si un contenedor falla, Docker Swarm debería levantar otro para mantener las 2 réplicas.

**Procedimiento:**

Primero desplegamos el servicio y revisamos si los contenedores estaban funcionando. Luego usamos comandos para mirar su estado y comprobar la configuración.

**Resultado esperado:**

Que las 2 réplicas estén funcionando y que el sistema pueda recuperarse si una falla.

**Resultado observado:**

Al principio aparecía 0/2, pero después revisamos nuevamente y las dos réplicas estaban en estado Running. El nodo también aparecía activo.

Todavía falta confirmar el resultado de la prueba de recuperación.

## Evidencias

- El archivo YAML pasó la validación.
- Docker creó el servicio `team05_worker`.
- Las 2 tareas aparecieron funcionando.
- El nodo estaba en estado Ready.
- Se configuraron límites de CPU, memoria y healthcheck.

## Limitaciones y lecciones

Una limitación es que trabajamos con un solo nodo, entonces si ese nodo falla, el servicio puede dejar de funcionar.

Aprendimos que Docker Swarm permite manejar varios contenedores, controlar sus recursos y facilitar su recuperación cuando presentan fallos.
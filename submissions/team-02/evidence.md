# Evidencias — team-02

## Diseño propuesto
Para resolver el incidente del ticket **REL-202 (NexaCloud)**, donde un release defectuoso provocó una interrupción de servicio, se diseñó una estrategia de despliegue continuo seguro basada en **Rolling Updates** y **Rollback Automático** con Docker Swarm:

- **Topología del Servicio:**
  - Servicio `web` basado en `nginx:1.27-alpine` con `replicas: 2` para garantizar alta disponibilidad básica.
  - Exposición de puerto `8082:80` para verificación de tráfico HTTP.
  - `healthcheck` integrado (`wget -q --spider http://127.0.0.1/`) para validar el estado de salud a nivel de aplicación cada 5 segundos.

- **Estrategia de Actualización (`update_config`):**
  - `parallelism: 1`: Actualización progresiva de una en una réplica. Al menos el 50% de la capacidad de servicio se mantiene activa sirviendo tráfico durante el despliegue.
  - `delay: 5s`: Tiempo de amortiguación entre la actualización exitosa de una tarea y el inicio de la siguiente.
  - `failure_action: rollback`: Si la tarea actualizada entra en fallo o no pasa los chequeos de salud, Swarm aborta la actualización e inicia la reversión de inmediato.
  - `monitor: 10s`: Ventana de tiempo tras el arranque de la tarea para certificar que el contenedor se mantiene estable y saludable.
  - `max_failure_ratio: 0.2`: Umbral de tolerancia a fallos estricto (20%).
  - `order: start-first`: Inicia la nueva réplica antes de dar de baja la antigua para minimizar o anular ventanas de indisponibilidad.

- **Estrategia de Reversión (`rollback_config`):**
  - `parallelism: 1`: Reversión controlada réplica a réplica.
  - `delay: 0s`: Sin retardo innecesario para restaurar el estado estable original lo más rápido posible.
  - `failure_action: pause`: Si la reversión encontrara problemas, pausa para intervención humana.
  - `monitor: 10s`: Supervisión post-rollback.
  - `order: stop-first`: Reemplazo directo deteniendo primero la tarea inestable revertida.

---

## Comandos de despliegue

1. **Inicialización y despliegue del stack:**
```bash
docker stack deploy --resolve-image always -c submissions/team-02/docker-stack.yml swarm02
```

2. **Verificación del estado inicial de los servicios y réplicas:**
```bash
docker service ls --filter "name=swarm02_web"
docker service ps swarm02_web
```

3. **Inspección de las políticas de actualización y rollback registradas en el motor:**
```bash
docker service inspect swarm02_web --format '{{json .Spec.UpdateConfig}}'
docker service inspect swarm02_web --format '{{json .Spec.RollbackConfig}}'
```

4. **Inyección de actualización con imagen defectuosa (simulación de fallo):**
```bash
docker service update --detach=false --update-parallelism 1 --update-failure-action rollback --image nginx:invalid-training-tag swarm02_web
```

5. **Validación post-rollback de la especificación e imagen activa:**
```bash
docker service inspect swarm02_web --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'
docker service ps swarm02_web
```

---

## Experimento funcional

**Hipótesis:**
Al actualizar el servicio `swarm02_web` hacia una imagen rota o inexistente (`nginx:invalid-training-tag`), el orquestador actualizará únicamente la primera réplica (`parallelism: 1`), detectará el fallo en la etapa de despliegue/monitoreo (`monitor: 10s`), abortará la propagación a la segunda réplica y disparará automáticamente la política `failure_action: rollback`. La segunda réplica permanecerá operativa e intacta sirviendo tráfico, y la especificación del servicio retornará automáticamente a `nginx:1.27-alpine`.

**Procedimiento:**
1. Desplegar el stack `swarm02` con 2 réplicas saludables en la versión base `nginx:1.27-alpine`.
2. Verificar que las 2 réplicas están en estado `Running` y responden peticiones.
3. Ejecutar `docker service update` síncrono (`--detach=false`) forzando la actualización con la imagen inválida `nginx:invalid-training-tag`.
4. Observar el flujo de rollback en tiempo real y el comportamiento de las tareas en `docker service ps`.
5. Inspeccionar la imagen resultante en la especificación del servicio mediante `docker service inspect`.

**Resultado esperado:**
- La tarea 1 intenta actualizarse y falla (error de resolución o arranque).
- La tarea 2 nunca se interrumpe y permanece en estado `Running`.
- El comando de actualización informa `rollback: update rolled back due to failure...` y converge en estado estable.
- La especificación de imagen del servicio regresa a `nginx:1.27-alpine`.

**Resultado observado:**
- Docker Swarm aisló el fallo en la tarea de actualización `swarm02_web.1`.
- Tras rechazar la imagen inválida (`Rejected: failed to resolve reference`), Swarm activó la rutina de rollback de forma automática.
- La réplica `swarm02_web.2` nunca se apagó (`Running 3 minutes ago`).
- La especificación del servicio quedó en `nginx:1.27-alpine@sha256:...`, cumpliendo exactamente con la hipótesis y el criterio de aceptación.

---

## Evidencias

### 1. Despliegue exitoso inicial (2/2 réplicas)
```text
$ docker service ls --filter "name=swarm02_web"
ID             NAME          MODE         REPLICAS   IMAGE               PORTS
0sm8o17wgt4j   swarm02_web   replicated   2/2        nginx:1.27-alpine   *:8082->80/tcp
```

### 2. Tareas activas antes del incidente
```text
$ docker service ps swarm02_web --no-trunc
ID                          NAME            IMAGE                                                                                       NODE             DESIRED STATE   CURRENT STATE            ERROR   PORTS
4kd0fu43349firwo5yb8l29jg   swarm02_web.1   nginx:1.27-alpine@sha256:65645c7bb6a0661892a8b03b89d0743208a18dd2f3f17a54ef4b76fb8e2f2a10   docker-desktop   Running         Running 19 seconds ago           
wm1vicp4vyx5jmzuv2xvw7s96   swarm02_web.2   nginx:1.27-alpine@sha256:65645c7bb6a0661892a8b03b89d0743208a18dd2f3f17a54ef4b76fb8e2f2a10   docker-desktop   Running         Running 19 seconds ago           
```

### 3. Registro de UpdateConfig y RollbackConfig en el motor
```json
$ docker service inspect swarm02_web --format '{{json .Spec.UpdateConfig}}'
{"Parallelism":1,"Delay":5000000000,"FailureAction":"rollback","Monitor":10000000000,"MaxFailureRatio":0.2,"Order":"start-first"}

$ docker service inspect swarm02_web --format '{{json .Spec.RollbackConfig}}'
{"Parallelism":1,"FailureAction":"pause","Monitor":10000000000,"MaxFailureRatio":0,"Order":"stop-first"}
```

### 4. Ejecución del update fallido y disparo del rollback
```text
$ docker service update --detach=false --update-parallelism 1 --update-failure-action rollback --image nginx:invalid-training-tag swarm02_web
swarm02_web
overall progress: 0 out of 2 tasks
1/2:  
2/2:  
overall progress: 0 out of 2 tasks
rollback: update rolled back due to failure or early termination of task 7rhpg1n1w2i0gckjrofhxglw2
overall progress: rolling back update: 2 out of 2 tasks
verify: Waiting 10 seconds to verify that tasks are stable...
verify: Service swarm02_web converged
```

### 5. Historial de tareas post-rollback (demostración de aislamiento y continuidad)
```text
$ docker service ps swarm02_web
ID             NAME                IMAGE                        NODE             DESIRED STATE   CURRENT STATE            ERROR                              PORTS
s81a0w740i7o   swarm02_web.1       nginx:invalid-training-tag   docker-desktop   Shutdown        Shutdown 2 minutes ago                                      
7rhpg1n1w2i0    \_ swarm02_web.1   nginx:invalid-training-tag   docker-desktop   Shutdown        Rejected 2 minutes ago   "failed to resolve reference "…"   
4kd0fu43349f    \_ swarm02_web.1   nginx:1.27-alpine            docker-desktop   Running         Running 3 minutes ago                                       
wm1vicp4vyx5   swarm02_web.2       nginx:1.27-alpine            docker-desktop   Running         Running 3 minutes ago                                       
```

### 6. Verificación de la especificación activa conservada
```text
$ docker service inspect swarm02_web --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'
nginx:1.27-alpine@sha256:65645c7bb6a0661892a8b03b89d0743208a18dd2f3f17a54ef4b76fb8e2f2a10
```

---

## Limitaciones y lecciones

1. **Limitación de un nodo único:**
   - En un cluster mono-nodo (Swarm local), tanto las réplicas principales como las nuevas tareas conviven en el mismo host físico/kernel. No es posible validar tolerancia a fallos a nivel de nodo (caída de host, partición de red o balanceo inter-nodo).
   - Los recursos de CPU, RAM y puertos compiten en la misma máquina.
2. **Dependencia del Healthcheck:**
   - La directiva `monitor` solo es verdaderamente efectiva si existe un `healthcheck` adecuado a nivel de servicio. Si un contenedor arranca un proceso pero este queda en bucle infinito o devuelve error HTTP 500, sin un healthcheck activo Swarm lo consideraría erróneamente "saludable" y continuaría actualizando el resto del stack.
3. **Lecciones aprendidas:**
   - La combinación `parallelism: 1` + `order: start-first` + `failure_action: rollback` proporciona un escudo robusto contra errores humanos y tags corruptos en pipelines de CI/CD.
   - En Swarm no aplican directivas de Compose como `build` o `depends_on` para orquestación de despliegue; la resiliencia debe manejarse declarativamente en la sección `deploy`.

# Presentación — team-02

**Misión:** REL-202 — Despliegues seguros  
**Cliente:** NexaCloud  
**Duración máxima:** 10 minutos (7 min exposición + 3 min preguntas)

---

## 1. Incidente y objetivo (1 min)
- **El Incidente:** Un release con defectos en producción tumbó el portal de NexaCloud, causando indisponibilidad total para los clientes y violando los SLAs corporativos.
- **Causa Raíz:** Despliegue en bloque (sin escalonamiento) y ausencia de mecanismos automáticos de supervisión de salud y reversión en fallo.
- **Objetivo Platform Engineering:**
  - Diseñar un esquema de entrega continua resiliente en Docker Swarm con 2 réplicas mínimas.
  - Actualizaciones atómicas y progresivas de una en una (`parallelism: 1`).
  - Monitoreo activo de estabilidad (`monitor: 10s`) junto con `healthcheck`.
  - Reversión automática a la versión previa estable (`failure_action: rollback`) ante cualquier anomalía.

---

## 2. Arquitectura y decisiones (2 min)
- **Topología Declarativa en `docker-stack.yml`:**
  - **Servicio:** `web` con imagen base `nginx:1.27-alpine` y `replicas: 2`.
  - **Healthcheck:** Verificación HTTP activa (`wget -q --spider http://127.0.0.1/`) cada 5s.
- **Estrategia `update_config`:**
  - `parallelism: 1`: Garantiza que el 50% de las réplicas siempre continúe procesando tráfico.
  - `delay: 5s`: Tiempo de enfriamiento entre lotes.
  - `failure_action: rollback`: Aborta el despliegue y revierte si se detecta falla.
  - `monitor: 10s`: Ventana de evaluación tras iniciar la nueva versión antes de declararla sana.
  - `order: start-first`: Arranca la nueva réplica antes de apagar la antigua para lograr zero downtime.
- **Estrategia `rollback_config`:**
  - `parallelism: 1`, `delay: 0s`, `order: stop-first` para retornar al estado seguro con máxima rapidez.

---

## 3. Demo: comandos y evidencia (3 min)
- **Paso 1: Despliegue inicial del stack**
  ```bash
  docker stack deploy --resolve-image always -c submissions/team-02/docker-stack.yml swarm02
  docker service ls --filter "name=swarm02_web"
  ```
  *Evidencia:* 2/2 réplicas saludables en ejecución.

- **Paso 2: Inyección de fallo simulado con tag corrupto**
  ```bash
  docker service update --detach=false --update-parallelism 1 --update-failure-action rollback --image nginx:invalid-training-tag swarm02_web
  ```
  *Evidencia observada:* Swarm intenta actualizar la primera réplica. Al fallar la resolución o inicio, detecta el error y emite:
  `rollback: update rolled back due to failure...` y `verify: Service swarm02_web converged`.

- **Paso 3: Evidencia irrefutable de no degradación del servicio**
  ```bash
  docker service ps swarm02_web
  docker service inspect swarm02_web --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'
  ```
  *Evidencia:* La réplica 2 (`swarm02_web.2`) permaneció en estado `Running` todo el tiempo. La réplica 1 fue restaurada a `nginx:1.27-alpine`.

---

## 4. Trade-offs y límites (1 min)
- **Trade-offs asumidos:**
  - **Velocidad vs. Seguridad:** El despliegue de una en una con ventana de monitoreo de 10s incrementa el tiempo total del release, pero previene caídas totales.
  - **Recursos (`order: start-first`):** Requiere un buffer temporal de CPU/RAM durante el despliegue para albergar la réplica adicional temporalmente.
- **Límites del entorno mono-nodo:**
  - No valida escenarios de partición de red ni pérdida de un host físico o nodo de computación.

---

## 5. Preguntas de la mesa redonda (3 min)

### ¿Qué incidente resolvieron y qué trade-offs asumieron?
Resolvimos la caída total del portal de NexaCloud causada por un despliegue sin validación progresiva. Asumimos el trade-off de un despliegue más lento (`parallelism: 1` con `monitor: 10s` y `delay: 5s`) a cambio de garantizar disponibilidad continua y recuperación automática sin requerir intervención humana en momentos críticos.

### ¿Qué comando demuestra el resultado, en vez de solo mostrar YAML?
El comando:
```bash
docker service ps swarm02_web
```
Demuestra que la réplica 2 nunca dejó de correr (`Running`), mientras que la réplica 1 muestra el historial con el intento fallido en `nginx:invalid-training-tag` (`Rejected`/`Shutdown`) y el re-despliegue exitoso de `nginx:1.27-alpine`. Adicionalmente:
```bash
docker service inspect swarm02_web --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'
```
demuestra en tiempo real que el orquestador revirtió la especificación a la imagen original.

### ¿Qué limitación tiene su prueba en un único nodo?
En un solo nodo, todas las tareas comparten el mismo daemon de Docker, CPU, memoria y subsistema de red local. No permite simular caídas de nodos de cómputo (hardware failures), latencias o particiones en la red overlay multi-host, ni validar el comportamiento de balanceo de carga ingress entre múltiples servidores físicos.

### ¿Qué cambiarían para una instalación de producción?
1. **Multi-nodo en alta disponibilidad:** Mínimo 3 nodos Manager en quórum Raft y múltiples nodos Worker con distribución anti-afinidad (`placement.preferences`).
2. **Registro de imágenes privado y seguro:** Despliegue con digest inmutable (`image@sha256:...`) firmado criptográficamente (Docker Content Trust / Cosign) en lugar de tags flotantes.
3. **Observabilidad:** Exportación de métricas a Prometheus/Grafana y alertas automáticas disparadas por eventos de rollback (`docker events`).
4. **Healthchecks profundos:** No solo chequear el servidor web local, sino endpoints de readiness que validen conectividad a bases de datos y microservicios upstream.

# INC-101 — Evidencias de resiliencia (Team 01)

> **Estado:** plantilla de prueba para PR. Los resultados deben completarse después de ejecutar los comandos; no se afirma que la prueba ya haya pasado.

## 1. Contexto y objetivo

NexaCloud necesita que su API de pedidos recupere automáticamente la cantidad deseada de instancias después de un fallo de contenedor. La misión consiste en configurar **3 réplicas**, política de reinicio `on-failure`, **healthcheck** y un **límite de memoria**.

## 2. Diseño de la solución

- Servicio: `api`.
- Imagen: `nginx:1.27-alpine` (imagen pública para simular la API).
- Orquestador: Docker Swarm local, un nodo.
- Estado deseado: tres tareas activas.
- Política de reinicio: `on-failure`.
- Healthcheck HTTP local al contenedor.
- Límite de memoria configurado en `deploy.resources.limits`.

**Archivo de infraestructura:** `submissions/team-01/docker-stack.yml`.

## 3. Procedimiento reproducible

```bash
# 1. Inicializar Swarm (solo si aún no está activo)
docker swarm init

# 2. Desplegar la solución del equipo desde la raíz del repositorio
docker stack deploy -c submissions/team-01/docker-stack.yml inc101

# 3. Consultar estado y configuración
docker stack services inc101
docker service ps inc101_api
docker service inspect inc101_api --pretty

# 4. Verificar que existen tres réplicas activas
docker service ls

# 5. Simular un fallo eliminando una tarea en ejecución
TASK_CONTAINER=$(docker ps -q --filter label=com.docker.swarm.service.name=inc101_api | head -n 1)
test -n "$TASK_CONTAINER" && docker kill "$TASK_CONTAINER"

# 6. Observar la reconciliación de estado (repetir hasta estabilizar)
docker service ps inc101_api
docker stack services inc101

# 7. Limpiar recursos al terminar
docker stack rm inc101
```

> Nota: el reemplazo de una tarea puede tardar algunos segundos. En un Swarm de un nodo, esto **no** demuestra tolerancia a la caída de la máquina.

## 4. Experimento funcional

**Hipótesis:** cuando una tarea falla, Swarm crea otra para recuperar el estado deseado de tres réplicas.

**Resultado esperado:** antes del fallo, `3/3` réplicas; tras matar un contenedor, Swarm vuelve a `3/3` y `docker service ps` muestra la tarea fallida y su reemplazo.

**Resultado observado:** `PENDIENTE DE EJECUCIÓN`.

**Tiempo de recuperación:** `PENDIENTE DE MEDICIÓN`.

## 5. Checklist de criterios de aceptación

- [ ] `docker stack deploy` finaliza correctamente.
- [ ] El servicio alcanza `3/3` réplicas.
- [ ] La política `on-failure` aparece en la inspección del servicio.
- [ ] El servicio define un healthcheck.
- [ ] Existe un límite de memoria.
- [ ] Se simula un fallo y se observa la recuperación.
- [ ] GitHub Actions reporta los checks del PR.

## 6. Evidencia adjunta

Pegar aquí extractos reales de `docker service ps`, `docker stack services` y el enlace al workflow de GitHub Actions. **No incluir credenciales ni secretos.**

## 7. Limitaciones y decisiones

- El laboratorio usa un único nodo: no verifica alta disponibilidad frente a fallos del host.
- El healthcheck informa el estado de salud, pero no debe asumirse que por sí solo produce siempre un reemplazo automático de una tarea Swarm.
- Un límite de memoria restringe recursos; no implica escalado automático.
- La imagen Nginx es un sustituto de la API empresarial, por lo que no se validan reglas de negocio.

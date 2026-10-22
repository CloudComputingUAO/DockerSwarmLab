# INC-101 — Presentación de resultados (Team 01)

**Mesa redonda: máximo 10 minutos (7 minutos de exposición y 3 de preguntas).**

## 1. Incidente empresarial — 1 minuto

NexaCloud experimenta interrupciones cuando falla un contenedor de la API de pedidos. Nuestro objetivo es mantener tres réplicas y recuperar el estado deseado sin intervención manual.

## 2. Arquitectura y decisiones — 2 minutos

```text
               Docker Swarm (1 nodo)
                        |
                 Servicio inc101_api
                  /      |      \
              Réplica 1  2       3
                 |       |       |
             Healthcheck HTTP por contenedor
```

Explicar la imagen seleccionada, `deploy.replicas`, `restart_policy`, `healthcheck` y el límite de memoria. Aclarar que un único nodo no ofrece tolerancia a fallos físicos.

## 3. Demostración — 3 minutos

```bash
docker stack deploy -c submissions/team-01/docker-stack.yml inc101
docker stack services inc101
docker service ps inc101_api

# Fallo controlado
TASK_CONTAINER=$(docker ps -q --filter label=com.docker.swarm.service.name=inc101_api | head -n 1)
test -n "$TASK_CONTAINER" && docker kill "$TASK_CONTAINER"

# Recuperación
docker service ps inc101_api
docker stack services inc101
```

**Antes:** registrar réplicas `___/3`.

**Después del fallo:** registrar réplicas `___/3`.

**Tiempo de recuperación observado:** `___` segundos.

**Resultado del PR en GitHub Actions:** `___` (enlace: `___`).

## 4. Trade-offs y límites — 1 minuto

- Swarm reconcilia el estado deseado, pero la recuperación no es instantánea.
- Un nodo único es un punto único de fallo.
- Healthcheck, restart policy y resource limits resuelven problemas diferentes.

## 5. Preguntas — 3 minutos

**¿Qué diferencia hay entre reiniciar un contenedor y recrear una tarea?**

**¿Qué sucede si falla el único nodo?**

**¿Cómo validarían la disponibilidad con tres nodos?**

---

**Antes de presentar:** sustituir todos los campos `___` con resultados reales. No presentar datos hipotéticos como mediciones.

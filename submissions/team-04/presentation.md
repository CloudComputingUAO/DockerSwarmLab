# Presentación — team-04 - NET-404 Redes privadas y DNS (<= 10 min)

## 1. Problema (1 min)
En NexaCloud un backend interno quedó expuesto a Internet. Cualquiera podía llamarlo directamente, sin pasar por la puerta pública.

## 2. Arquitectura (2 min)
- Red `private_net` overlay compartida por ambos servicios.
- `frontend` (whoami, 1 réplica): único servicio con `ports` (`18080:80`).
- `backend` (whoami, 2 réplicas): **sin `ports`**, descubierto por DNS con su nombre de servicio.

## 3. Demo (3 min)
1. `docker stack deploy -c submissions/team-04/docker-stack.yml team04` y `docker stack services team04`.
2. `curl http://localhost:18080` -> responde el frontend.
3. `docker service inspect team04_backend --format '{{json .Endpoint.Ports}}'` -> `null`.
4. `docker run --rm --network team04_private_net busybox:1.36 nslookup backend` -> VIP `10.0.2.5`; `tasks.backend` -> 2 IPs.
5. `wget http://backend` 4 veces -> alternan los 2 hostnames (balanceo).
6. Mismo `wget` fuera de la red -> `bad address 'backend'`.

## 4. Decisiones y trade-offs (2 min)
- `attachable: true` para poder probar; en producción se desactiva.
- Sin `depends_on`: Swarm lo ignora; dependemos de DNS y reinicios.

## 5. Limitaciones y aprendizajes (2 min)
- Un solo nodo: no se prueba overlay entre hosts ni cifrado.
- En producción: red `encrypted`, TLS, proxy real hacia `backend`, healthchecks.
- Aprendimos que publicar `ports` expone el servicio en todos los nodos por la routing mesh.

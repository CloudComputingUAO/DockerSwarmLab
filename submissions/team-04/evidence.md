# Evidencias — team-04 (NET-404 - Redes privadas y DNS)

## Diseño propuesto

- **Red `private_net`** con driver `overlay`: red privada de la stack, con DNS interno de Swarm. Se marca `attachable: true` solo para poder conectar un contenedor temporal de pruebas (busybox).
- **`frontend`** (`traefik/whoami:v1.11`, 1 réplica): única puerta pública, publicada en `18080:80` mediante la *ingress routing mesh*.
- **`backend`** (`traefik/whoami:v1.11`, 2 réplicas): **sin `ports`**, solo alcanzable dentro de `private_net` por su nombre de servicio `backend`.
- `restart_policy: on-failure` en ambos servicios.
- No se usa `depends_on` ni `build` porque `docker stack deploy` los ignora; el descubrimiento se resuelve por DNS, no por orden de arranque.

```
Internet/host --:18080--> [frontend] --+
                                       |  private_net (overlay)
                         backend (VIP) +--> [backend.1] [backend.2]
            (sin puerto publicado: no accesible desde fuera)
```

## Comandos de despliegue

```bash
docker info --format '{{.Swarm.LocalNodeState}}'   # active
docker stack deploy -c submissions/team-04/docker-stack.yml team04
docker stack services team04
docker stack ps team04
```

Salida observada:

```
NAME              MODE         REPLICAS   IMAGE                  PORTS
team04_backend    replicated   2/2        traefik/whoami:v1.11
team04_frontend   replicated   1/1        traefik/whoami:v1.11   *:18080->80/tcp
```

## Experimento ejecutado

| # | Prueba | Comando | Esperado | Observado |
|---|--------|---------|----------|-----------|
| 1 | Frontend público | `curl http://localhost:18080` | Responde whoami | OK `Hostname: ab2bc4dd7696`, `IP: 10.0.0.6` (ingress) |
| 2 | Backend sin puertos | `docker service inspect team04_backend --format '{{json .Endpoint.Ports}}'` | `null` | OK `null` (frontend: `PublishedPort 18080`) |
| 3 | DNS interno (VIP) | `docker run --rm --network team04_private_net busybox:1.36 nslookup backend` | Resuelve una IP virtual | OK `backend -> 10.0.2.5` (servidor DNS `127.0.0.11`) |
| 4 | DNS por réplica | `... nslookup tasks.backend` | Una IP por réplica | OK `10.0.2.6`, `10.0.2.7` |
| 5 | HTTP interno + balanceo | `... sh -c 'for i in 1 2 3 4; do wget -qO- http://backend \| grep Hostname; done'` | Responden las 2 réplicas | OK `c33526c44b61`, `b5b68f6c0975`, `c33526c44b61`, `b5b68f6c0975` (round-robin) |
| 6 | Aislamiento | `docker run --rm busybox:1.36 wget -T 3 -qO- http://backend` (fuera de la red) | Falla | OK `wget: bad address 'backend'` |

## Evidencias

- El backend **no tiene puerto publicado** y **solo se resuelve dentro de `private_net`**: desde la red bridge por defecto el nombre `backend` no existe.
- El DNS de Swarm entrega una **VIP** para `backend` y el balanceo IPVS reparte las peticiones entre las 2 réplicas.
- `tasks.backend` expone las IPs individuales de cada tarea (útil para depurar).

## Limitaciones y lecciones

- Prueba en **un solo nodo**: no se demuestra tráfico overlay entre hosts distintos (VXLAN entre nodos) ni el cifrado de la red (`driver_opts: encrypted`).
- `traefik/whoami` es una imagen *scratch* sin shell, por eso la prueba de DNS se hace con un contenedor busybox conectado a la red (`attachable: true`).
- En producción: `attachable: false` (o una red separada solo para depuración), red `encrypted`, TLS en el frontend, un proxy real (nginx/traefik) que reenvíe a `backend`, y healthchecks.
- Lección: `ports` publica en **todos** los nodos vía la routing mesh; un servicio interno nunca debe declararlos.


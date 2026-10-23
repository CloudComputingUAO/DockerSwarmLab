# Evidencias — team-03

## Diseño propuesto
Para resolver el incidente de exposición de credenciales detectado en la auditoría de NexaCloud, se implementó una arquitectura basada en Docker Swarm Secrets nativos:

- **Gestión externa (external: true)**: El secreto demo_token se desacopla del ciclo de vida de la stack y del control de versiones (Git), evitando que quede incluido en imágenes o archivos YAML.
- **Principio de mínimo privilegio**: El secreto se monta exclusivamente en el servicio autorizado (vault_client) mediante un sistema de archivos en memoria (tmpfs) ubicado en /run/secrets/demo_token. No se expone ningún puerto hacia el exterior (ports) ni se inyecta en variables de entorno.
- **Control del proceso en busybox**: Dado que utilizamos la imagen busybox:1.36 se diseñó un command que valida la existencia del archivo del secreto, emite el log secret-mounted para trazabilidad y mantiene el proceso PID 1 activo para garantizar la estabilidad de la réplica 1/1.
Se agrega "Command" debido a que se necesita para que el servicio tenga una acción a realizar.
Añadimos el servicio secreto en base a la documentación oficial de Docker Swarm y se agrega también dentro del archivo `docker-stack.yml` con la propiedad `external: true` para evitar exponer la contraseña en el código fuente.

## Comandos de despliegue
```bash

'training-only-token' | docker secret create demo_token -   # El secreto debe crearse antes de realizar el stack/despliegue debido a que es externo

docker stack deploy -c docker-stack.yml team03  # Desplegar stack

docker stack services team03  # Verificar estado
docker stack ps team03 #Verificar estado
docker service logs team03_vault_client #Verificar logs del servicio debe arrojar "secret-mounted"
```

## Experimento funcional
**Hipótesis:**
Al desplegar la stack en Docker Swarm, el secreto externo `demo_token` será aprovisionado de forma segura y accesible exclusivamente en la ruta `/run/secrets/demo_token` del contenedor `vault_client`, garantizando que el servicio alcance el estado `1/1` réplicas y que el valor del secreto no sea expuesto en el código, en el archivo YAML ni en las variables de entorno (`env`).

**Procedimiento:**
```bash
'training-only-token' | docker secret create demo_token -   # Crear el secreto

docker stack deploy -c docker-stack.yml team03  # Desplegar stack

docker stack services team03  # Verificar estado
docker stack ps team03 # Verificar estado
docker service logs team03_vault_client # Verificar logs del servicio debe arrojar "secret-mounted"

python -m pip install -r requirements.txt # Instalamos dependencias (En mi caso ya tenia la libreria PyYAML con una version mas actualizada)

python -m pytest -q tests/test_contracts.py --team team-03 # Ejecutamos pruebas unitarias
```
**Resultado esperado:**
Se necesita que apruebe todas las pruebas unitarias aparte de eso, la contraseña o "secret" no debe ser visible en el github, ni configuracion del yaml, debe ser externa y unica para Docker

**Resultado observado:**
Los resultados obtenidos validaron completamente la hipótesis planteada:
1. **Estado del servicio:** `docker stack services` y `docker stack ps` confirmaron `1/1` réplicas activas en estado `Running` sobre el nodo `docker-desktop`.
2. **Lectura y trazabilidad:** `docker service logs` registró la salida esperada `secret-mounted`.
3. **Aislamiento del secreto:** La ejecución de `cat /run/secrets/demo_token` dentro del contenedor devolvió `training-only-token`, comprobando que fue montado correctamente.
4. **Seguridad de entorno:** La consulta de variables de entorno mediante `env` demostró que ninguna credencial fue inyectada en el entorno del proceso.
5. **Cumplimiento de contrato:** Las pruebas estáticas con `pytest` pasaron exitosamente (`4 passed in 0.06s`).


## Evidencias
```bash
PS C:\SPEC303\DockerSwarmLab\submissions\team-03> docker stack ps team03
ID             NAME                    IMAGE          NODE             DESIRED STATE   CURRENT STATE           ERROR     PORTS
ki998w5nif6s   team03_vault_client.1   busybox:1.36   docker-desktop   Running         Running 3 seconds ago

PS C:\SPEC303\DockerSwarmLab\submissions\team-03> docker stack services team03
ID             NAME                  MODE         REPLICAS   IMAGE          PORTS
0yz8nwgh2uvk   team03_vault_client   replicated   1/1        busybox:1.36

PS C:\SPEC303\DockerSwarmLab\submissions\team-03> docker service logs team03_vault_client
team03_vault_client.1.ki998w5nif6s@docker-desktop    | secret-mounted

PS C:\SPEC303\DockerSwarmLab> python -m pytest -q tests/test_contracts.py --team team-03
....
4 passed in 0.06s   [100%]

PS C:\SPEC303\DockerSwarmLab> docker service inspect team03_vault_client --pretty  # Con este comando podemos ver que el secreto no es visible en el yaml y si en el servicio

ID:             0yz8nwgh2uvkuq3afjmpd14x7
Name:           team03_vault_client
Labels:
 com.docker.stack.image=busybox:1.36
 com.docker.stack.namespace=team03
Service Mode:   Replicated
 Replicas:      1
Placement:
UpdateConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:   1
 On failure:    pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:         busybox:1.36@sha256:73aaf090f3d85aa34ee199857f03fa3a95c8ede2ffd4cc2cdb5b94e566b11662
 Args:          sh -c test -f /run/secrets/demo_token && echo 'secret-mounted' && tail -f /dev/null
Secrets:
 Target:        demo_token
  Source:       demo_token
Resources:
Networks: 9qxhftib5dwp563q4ek8i21td
Endpoint Mode:  vip
```

## Limitaciones y lecciones

1. **Dependencia de preexistencia del secreto (Trade-off operacional):**
   Al definir el secreto como `external: true`, el despliegue queda desacoplado del ciclo de vida del repositorio. Sin embargo, esto introduce una fuerte dependencia de orden: si el secreto `demo_token` no ha sido provisionado previamente en el clúster Swarm, `docker stack deploy` falla de forma bloqueante. En un pipeline de CI/CD esto exige un paso previo de aprovisionamiento de infraestructura o gestión de configuración.

2. **Limitaciones de la prueba en un solo nodo:**
   La prueba se realizó en Docker Desktop, utilizando un solo nodo. Esto permitió comprobar que el secreto se puede leer de forma segura dentro del contenedor y que no es necesario almacenarlo en las variables de entorno. Sin embargo, al utilizar un solo nodo, no fue posible comprobar cómo se comparten los secretos entre diferentes equipos ni cómo respondería el sistema ante fallos de conexión o la pérdida de uno de los nodos.

3. **Gestión de dependencias del entorno local:**
   Durante la ejecución de las pruebas estáticas en Windows con Python 3.14, la instalación de `PyYAML==6.0.2` desde `requirements.txt` requería compiladores de C++ (`Microsoft Visual C++ 14.0`). Se resolvió utilizando la rueda binaria precompilada (`PyYAML 6.0.3`), confirmando que la compatibilidad del entorno local debe gestionarse asegurando dependencias con paquetes precompilados.

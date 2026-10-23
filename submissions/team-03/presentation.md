# Presentación — team-03

**Duración máxima: 10 minutos (7 exposición + 3 preguntas).**

**1. Incidente y objetivo (1 min)**
El incidente a resolver (SEC-303) era que las credenciales estaban expuestas directamente en el código fuente. Nuestro objetivo fue eliminar esta vulnerabilidad aislando la contraseña mediante la implementación de Docker Secrets nativos de Swarm.

**2. Arquitectura y decisiones (2 min)**
Decidimos utilizar un secreto llamado `demo_token` configurado con la propiedad `external: true` en el archivo `docker-stack.yml`. Esto garantiza que la clave no se guarde en el repositorio. El contenedor está configurado para montar este secreto en la ruta `/run/secrets/demo_token`, la cual funciona sobre una memoria RAM temporal (tmpfs), asegurando que la clave nunca se escriba en el disco duro físico.

**3. Demo: comandos y evidencia (3 min)**
El flujo de ejecución para demostrar la solución es el siguiente:
- Se inyecta el secreto externamente: `echo "clave-secreta" | docker secret create demo_token -`
- Se despliega el entorno: `docker stack deploy -c docker-stack.yml team03`
- Se verifica la configuración asignada por el orquestador: `docker service inspect team03_vault_client`
- Se lee la credencial directamente desde la memoria del contenedor activo: `docker exec $(docker ps -q -f "name=team03_vault_client") cat /run/secrets/demo_token`

**4. Trade-offs y límites (1 min)**
- **Trade-off:** Aumentamos la seguridad pero sacrificamos la automatización total; ahora el despliegue depende de que un operador cree el secreto en la terminal previamente.
- **Límites:** Al probar esto en una sola máquina local (nodo único), no es posible observar el mecanismo de seguridad de red (mTLS) que Swarm utiliza para distribuir contraseñas de forma encriptada entre múltiples servidores.

**5. Preguntas preparadas para la audiencia (3 min)**
*   **¿Qué pasa si un atacante logra entrar al contenedor mientras está encendido?**
    El atacante sí podría leer el archivo `/run/secrets/demo_token`. Docker Secrets protege la clave para que no esté en el código fuente ni en el disco duro, pero si el contenedor es vulnerado por dentro, la clave es visible. Por eso es vital asegurar también el código de la aplicación.
*   **¿Por qué es importante que el secreto se guarde en "tmpfs" (memoria RAM) y no en el disco?**
    Porque si el servidor se apaga repentinamente o alguien roba el disco duro físico, la memoria RAM se borra por completo, sin dejar ningún rastro de la contraseña.
*   **Si el secreto es "external: true", ¿cómo lo automatizarían en un entorno real?**
    En lugar de escribirlo a mano, usaríamos una herramienta de integración continua (como GitHub Actions o Jenkins) conectada a un gestor como HashiCorp Vault para inyectar el secreto automáticamente justo antes del despliegue.
---
description: Stop the TODO development environment without removing containers, volumes or data
allowed-tools: Bash, Read
---

# Para el entorno de desarrollo

Detén solo lo que pertenezca a este repositorio. No elimines contenedores, volúmenes ni datos, y no uses `docker compose down`.

## Detecta el modo

- **Docker completo:** `docker compose ps` muestra el servicio `app` en marcha.
- **Host:** hay un proceso `buffalo dev` cuyo directorio de trabajo es la raíz de este repositorio.

Si no hay nada en marcha, informa de ello y termina.

## Modo Docker completo

1. Ejecuta `docker compose stop`.
2. Comprueba con `docker compose ps` que ya no queda ningún servicio en marcha.

## Modo host

1. Localiza los procesos `buffalo dev` con `pgrep -f "buffalo dev"`. Para cada uno, comprueba con `lsof -a -p <pid> -d cwd` que su directorio de trabajo es la raíz de este repositorio. Si pertenece a otro proyecto, no lo toques.
2. Detén los de este repositorio con `kill <pid>`.
3. Ejecuta `docker compose stop db`.

## Comprobación

`http://localhost:3004/healthz` ya no responde y nada de este repositorio sigue escuchando en los puertos `3004` ni `5434`.

## Resultado

Informa de:

- modo detectado;
- servicios y procesos detenidos;
- resultado de la comprobación;
- procesos de otros proyectos que hayas dejado intactos.

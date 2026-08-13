---
title: Creación del stack mediaserver
description: Este stack alojará los servicios necesarios para la automatización del servidor multimedia alojado en el Homelab.
published: true
date: 2026-08-13T10:52:04.047Z
tags: docker, portainer, mediaserver, homelab
editor: markdown
dateCreated: 2026-08-13T10:52:04.047Z
---

# Despliegue del Stack Multimedia (mediaserver)

Guía paso a paso del proceso de limpieza, configuración y despliegue del stack de gestión multimedia mediante **Portainer** y **Docker Compose** en el Homelab.

---

## Resumen del Entorno

* **Nombre del Stack:** `mediaserver`
* **Orquestador:** Portainer (Docker Engine)
* **Ruta Base de Configuración:** `/opt/mediastack`
* **Rutas de Almacenamiento y Medios:** `/data`

---

## Servicios y Puertos Expuestos

| Servicio | Imagen Docker | Puerto | Descripción / Función |
| :--- | :--- | :--- | :--- |
| **Homarr** | `ghcr.io/homarr-labs/homarr:latest` | `7575` | Dashboard centralizado de control |
| **Wizarr** | `ghcr.io/wizarrrr/wizarr:latest` | `5690` | Portal de invitaciones y bienvenida a usuarios |
| **qBittorrent** | `lscr.io/linuxserver/qbittorrent:latest` | `8080` | Cliente de descargas Torrent |
| **Prowlarr** | `lscr.io/linuxserver/prowlarr:latest` | `9696` | Gestor global de indexadores/trackers |
| **Radarr** | `lscr.io/linuxserver/radarr:latest` | `7878` | Gestión automatizada de Películas |
| **Sonarr** | `lscr.io/linuxserver/sonarr:latest` | `8989` | Gestión automatizada de Series de TV |
| **Bazarr** | `lscr.io/linuxserver/bazarr:latest` | `6767` | Gestión y descarga de Subtítulos |
| **Overseerr** | `lscr.io/linuxserver/overseerr:latest` | `5055` | Portal de peticiones e integración con Plex |

---

## Preparación de Directorios y Permisos en la VM

Antes de realizar el despliegue en Portainer, se preparó el entorno local en la máquina virtual ejecutando los siguientes comandos en terminal para asegurar permisos absolutos sobre los volúmenes de datos:

```bash
# 1. Crear directorio base para configuraciones persistentes
sudo mkdir -p /opt/mediastack

# 2. Ajustar permisos globales para permitir el mapeo de contenedores
sudo chmod -R 777 /opt/mediastack

# 3. Crear directorios para descargas y bibliotecas de medios
sudo mkdir -p /data/downloads /data/media/movies /data/media/tv
```
---
## Especificación `docker-compose.yml`
Por defecto, el docker-compose recoge los puertos por defecto de los diferentes servicios. Por razones de seguridad, sería aconsejable modificar los puertos, para dificultar posibles ataques que procedan del exterior. Todo depende de la finalidad que necesitemos en nuestra infraestructura.
```yml
version: "3.8"

networks:
  mediastack_net:
    driver: bridge

services:
  # -------------------------------------------------------------------
  # DASHBOARD & GESTIÓN DE USUARIOS
  # -------------------------------------------------------------------
  homarr:
    image: ghcr.io/homarr-labs/homarr:latest
    container_name: homarr
    restart: unless-stopped
    ports:
      - "7575:7575"
    environment:
      - SECRET_ENCRYPTION_KEY=INTRODUCE_TU_CONTRASEÑA
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /opt/mediastack/homarr/appdata:/appdata
    networks:
      - mediastack_net

  wizarr:
    image: ghcr.io/wizarrrr/wizarr:latest
    container_name: wizarr
    restart: unless-stopped
    ports:
      - "5690:5690"
    environment:
      - TZ=Europe/Madrid
      - APP_URL=http://<IP_DE_TU_VM>:5690
    volumes:
      - /opt/mediastack/wizarr/database:/data/database
    networks:
      - mediastack_net

  # -------------------------------------------------------------------
  # CLIENTE DE DESCARGAS
  # -------------------------------------------------------------------
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    ports:
      - "8080:8080" # WebUI
      - "6881:6881"
      - "6881:6881/udp"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
      - WEBUI_PORT=8080
    volumes:
      - /opt/mediastack/qbittorrent/config:/config
      - /data/downloads:/downloads
    networks:
      - mediastack_net

  # -------------------------------------------------------------------
  # INDEXADORES Y GESTORES DE MEDIOS (*ARR)
  # -------------------------------------------------------------------
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    ports:
      - "9696:9696"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
    volumes:
      - /opt/mediastack/prowlarr/config:/config
    networks:
      - mediastack_net

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    ports:
      - "7878:7878"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
    volumes:
      - /opt/mediastack/radarr/config:/config
      - /data/media/movies:/movies
      - /data/downloads:/downloads
    networks:
      - mediastack_net

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    ports:
      - "8989:8989"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
    volumes:
      - /opt/mediastack/sonarr/config:/config
      - /data/media/tv:/tv
      - /data/downloads:/downloads
    networks:
      - mediastack_net

  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    restart: unless-stopped
    ports:
      - "6767:6767"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
    volumes:
      - /opt/mediastack/bazarr/config:/config
      - /data/media/movies:/movies
      - /data/media/tv:/tv
    networks:
      - mediastack_net

  # -------------------------------------------------------------------
  # PETICIONES Y SOLICITUDES DE MEDIOS
  # -------------------------------------------------------------------
  overseerr:
    image: lscr.io/linuxserver/overseerr:latest
    container_name: overseerr
    restart: unless-stopped
    ports:
      - "5055:5055"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
    volumes:
      - /opt/mediastack/overseerr/config:/config
    networks:
      - mediastack_net
```
Tras esto, pulsamos el botón Deploy the stack, y esperamos a que cree el stack.
Una vez creado, si accedemos a la sección Stacks, veremos que aparece un nuevo stack, con el nombre que le indicamos:
![comprobación_creación_stack_mediaserver.png](/comprobación_creación_stack_mediaserver.png)
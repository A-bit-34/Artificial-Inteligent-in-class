# Manual técnico --- Stack de IA local con Docker, Ubuntu y NVIDIA

**Proyecto:** Proyect based on SDD (Spec Driven Development)\
**Versión del manual:** 1.0\
**Fecha:** 2026-10-06\
**Rol objetivo:** Administrador de sistemas GNU/Linux / DevOps\
**Plataforma:** Ubuntu Server 24.04/26.04, Docker Engine + Docker
Compose v2, NVIDIA GPU\
**Directorio raíz:** `$HOME/proyecto`

------------------------------------------------------------------------

## 0. Alcance y decisiones de diseño

Este documento transforma la especificación SDD del proyecto en un
procedimiento operativo reproducible.

La arquitectura contiene:

-   **Ollama** --- servidor local de modelos LLM.
-   **Open WebUI** --- interfaz web tipo ChatGPT.
-   **Hermes Agent** --- agente autónomo.
-   **OpenCode** --- entorno de programación asistida por IA vía web.
-   **ComfyUI** --- generación y procesamiento de imágenes.
-   **YOLO / Ultralytics** --- visión artificial y detección.
-   **SearXNG** --- metabuscador privado.
-   **RAG** --- servicio local de recuperación aumentada con documentos,
    implementado como un pequeño servicio FastAPI que utiliza Ollama
    para embeddings/generación.
-   **Valkey** --- dependencia técnica de SearXNG para funciones de
    limitación/estado; no es un servicio de usuario del proyecto.

### 0.1 Correcciones respecto a la especificación original

La especificación inicial contiene algunos valores que no coinciden con
las interfaces actuales de las aplicaciones:

1.  **OpenCode:** el servidor actual usa **4096/TCP** por defecto. El
    diseño conserva el puerto externo solicitado `8443`, pero publica
    `8443:4096`.
2.  **Hermes Agent:** el gateway OpenAI-compatible usa **8642/TCP**. Se
    conserva el puerto externo solicitado `8000`, publicando
    `8000:8642`.
3.  **RAG:** no puede publicar `11434` porque ese puerto ya lo utiliza
    Ollama. Se publica `8001:8000`; también puede eliminarse
    completamente la publicación para que RAG sea sólo interno.
4.  **SearXNG:** la instalación moderna utiliza Valkey para algunas
    funciones del limiter. Se incluye como dependencia auxiliar.
5.  **GPU:** Docker debe tener correctamente configurado el NVIDIA
    Container Toolkit antes de iniciar los contenedores GPU.
6.  **ComfyUI:** no se presupone una imagen Docker oficial estable de
    ComfyUI; el ejemplo utiliza una imagen Docker mantenida por terceros
    y se recomienda fijar una versión/digest después de validar el
    despliegue.
7.  **YOLO:** se utiliza la imagen oficial de Ultralytics.

Estas correcciones son necesarias para cumplir simultáneamente los
requisitos de funcionalidad y ausencia de colisiones de puertos.

------------------------------------------------------------------------

# 1. Arquitectura final

## 1.1 Diagrama lógico

``` text
                         LAN / navegador
                               |
                    +----------+----------+
                    |                     |
                 :3000                 :8443
                    |                     |
              +-----v------+        +-----v------+
              | Open WebUI |        |  OpenCode  |
              +-----+------+        +-----+------+
                    |                     |
                    |                     |
              +-----v---------------------v------------------+
              |                red-ia (bridge)              |
              |                                              |
              |  ollama:11434       hermes-agent:8642       |
              |  searxng:8080       comfyui:8188             |
              |  yolo:5000          rag:8000                 |
              |                                              |
              +------------------------------------------------
                    |             |              |
                    |             |              |
                NVIDIA GPU    Internet       Documentos
                    |             |              |
              +-----v------+  +---v----+   +---v------+
              | Ollama GPU |  | SearXNG |   | RAG data |
              +------------+  +--------+   +----------+
```

## 1.2 Puertos

  -------------------------------------------------------------------------------------------------
  Servicio                 Host       Contenedor URL desde LAN         URL dentro de `red-ia`
  ------------ ---------------- ---------------- --------------------- ----------------------------
  Ollama                  11434            11434 `http://HOST:11434`   `http://ollama:11434`

  Open WebUI               3000             8080 `http://HOST:3000`    `http://openwebui:8080`

  Hermes Agent             8000             8642 `http://HOST:8000`    `http://hermes-agent:8642`

  OpenCode                 8443             4096 `http://HOST:8443`    `http://opencode:4096`

  ComfyUI                  8188             8188 `http://HOST:8188`    `http://comfyui:8188`

  YOLO                     5000             5000 `http://HOST:5000`    `http://yolo:5000`

  SearXNG                  8080             8080 `http://HOST:8080`    `http://searxng:8080`

  RAG                      8001             8000 `http://HOST:8001`    `http://rag:8000`

  Valkey                    ---             6379 No publicar           `valkey:6379`
  -------------------------------------------------------------------------------------------------

**Recomendación:** en una instalación real, publique hacia la LAN
solamente los servicios que realmente necesite. Para máxima seguridad,
RAG, Ollama y SearXNG pueden mantenerse internos y exponerse únicamente
mediante Open WebUI/reverse proxy.

------------------------------------------------------------------------

# 2. Prerrequisitos

## 2.1 Hardware recomendado

La especificación contempla NVIDIA RTX 3050/4060 o similares.

Antes de instalar nada:

``` bash
lscpu
free -h
lsblk
df -h
lspci | grep -i -E 'vga|3d|nvidia'
```

Comprobar GPU:

``` bash
nvidia-smi
```

Si `nvidia-smi` no existe o no detecta la tarjeta, **no continuar con
Docker GPU** hasta solucionar primero el driver.

### Memoria

Como orientación:

-   16 GB RAM: instalación básica y modelos pequeños.
-   32 GB RAM: recomendada.
-   64 GB o más: mejor para agentes, RAG y modelos grandes.

La VRAM limita directamente el tamaño práctico de los modelos.

------------------------------------------------------------------------

# 3. Instalación de Ubuntu y preparación del sistema

## 3.1 Actualizar el sistema

``` bash
sudo apt update
sudo apt full-upgrade -y
sudo apt install -y \
  ca-certificates \
  curl \
  git \
  gnupg \
  jq \
  unzip \
  zip \
  htop \
  nvme-cli \
  openssl \
  python3 \
  python3-pip
```

Reiniciar si se actualizó el kernel:

``` bash
sudo reboot
```

Comprobar:

``` bash
uname -a
lsb_release -a
```

------------------------------------------------------------------------

# 4. Instalación del driver NVIDIA

## 4.1 Detectar el driver recomendado

``` bash
ubuntu-drivers devices
```

Instalación automática:

``` bash
sudo ubuntu-drivers autoinstall
```

Reiniciar:

``` bash
sudo reboot
```

Comprobar:

``` bash
nvidia-smi
```

Debe aparecer la GPU, versión del driver, VRAM y procesos.

### Importante sobre CUDA

Para este proyecto no es necesario instalar manualmente el CUDA Toolkit
completo en el host sólo para ejecutar contenedores CUDA. Lo importante
es:

1.  Driver NVIDIA funcional en Ubuntu.
2.  NVIDIA Container Toolkit.
3.  Docker configurado para entregar la GPU al contenedor.

Las imágenes de aplicaciones pueden incluir sus propias librerías CUDA.

------------------------------------------------------------------------

# 5. Instalar Docker Engine

Docker recomienda utilizar su repositorio oficial APT.

## 5.1 Eliminar paquetes conflictivos

``` bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
  sudo apt-get remove -y "$pkg" 2>/dev/null || true
done
```

## 5.2 Instalar dependencias

``` bash
sudo apt-get update

sudo apt-get install -y ca-certificates curl
```

## 5.3 Añadir la clave oficial

``` bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

## 5.4 Añadir repositorio

``` bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## 5.5 Instalar Docker

``` bash
sudo apt-get update

sudo apt-get install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Comprobar:

``` bash
docker --version
docker compose version
```

Prueba:

``` bash
sudo docker run --rm hello-world
```

------------------------------------------------------------------------

# 6. Permitir Docker sin sudo

Añadir el usuario actual al grupo Docker:

``` bash
sudo usermod -aG docker "$USER"
```

Cerrar sesión y volver a entrar, o ejecutar:

``` bash
newgrp docker
```

Comprobar:

``` bash
docker ps
```

> **Advertencia de seguridad:** pertenecer al grupo `docker` equivale en
> la práctica a disponer de privilegios muy elevados sobre el host.

------------------------------------------------------------------------

# 7. NVIDIA Container Toolkit

## 7.1 Instalar repositorio

``` bash
curl -fsSL \
  https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor \
  -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L \
  https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

Actualizar:

``` bash
sudo apt-get update
```

Instalar:

``` bash
sudo apt-get install -y nvidia-container-toolkit
```

## 7.2 Configurar Docker

``` bash
sudo nvidia-ctk runtime configure --runtime=docker
```

Reiniciar Docker:

``` bash
sudo systemctl restart docker
```

Comprobar:

``` bash
sudo systemctl status docker --no-pager
```

## 7.3 Prueba GPU dentro de Docker

Prueba compatible con el método clásico:

``` bash
docker run --rm --gpus all \
  nvidia/cuda:12.6.2-base-ubuntu24.04 \
  nvidia-smi
```

Si Docker \>= 28.2 y NVIDIA Container Toolkit \>= 1.18 están
disponibles, también se puede utilizar CDI:

``` bash
sudo nvidia-ctk cdi generate --output=/var/run/cdi/nvidia.yaml
```

Comprobar:

``` bash
nvidia-ctk cdi list
```

Y probar:

``` bash
docker run --rm \
  --device nvidia.com/gpu=all \
  nvidia/cuda:12.6.2-base-ubuntu24.04 \
  nvidia-smi
```

Para este manual se mantienen ejemplos Compose con `gpus: all`, porque
son más portables entre las configuraciones Compose habituales. Si el
entorno está estandarizado en CDI, se puede migrar posteriormente.

------------------------------------------------------------------------

# 8. Crear la estructura del proyecto

``` bash
mkdir -p "$HOME/proyecto"
cd "$HOME/proyecto"
```

Estructura recomendada:

``` text
$HOME/proyecto/
├── .env
├── .gitignore
├── README.md
├── docker-ollama.yml
├── docker-openwebui.yml
├── docker-hermes-agent.yml
├── docker-opencode.yml
├── docker-comfyui.yml
├── docker-yolo.yml
├── docker-searxng.yml
├── docker-rag.yml
│
├── config/
│   ├── searxng/
│   │   ├── settings.yml
│   │   └── limiter.toml
│   │
│   ├── opencode/
│   │   └── opencode.json
│   │
│   └── rag/
│       └── app.py
│
├── data/
│   ├── ollama/
│   ├── openwebui/
│   ├── hermes/
│   ├── opencode/
│   ├── comfyui/
│   │   ├── models/
│   │   ├── input/
│   │   ├── output/
│   │   ├── user/
│   │   └── custom_nodes/
│   ├── yolo/
│   │   ├── datasets/
│   │   └── runs/
│   ├── searxng/
│   └── rag/
│       ├── documents/
│       ├── index/
│       └── uploads/
│
└── backups/
```

Crear:

``` bash
cd "$HOME/proyecto"

mkdir -p \
  config/searxng \
  config/opencode \
  config/rag \
  data/ollama \
  data/openwebui \
  data/hermes \
  data/opencode \
  data/comfyui/models \
  data/comfyui/input \
  data/comfyui/output \
  data/comfyui/user \
  data/comfyui/custom_nodes \
  data/yolo/datasets \
  data/yolo/runs \
  data/searxng \
  data/rag/documents \
  data/rag/index \
  data/rag/uploads \
  backups
```

------------------------------------------------------------------------

# 9. Crear `.env`

Generar secretos:

``` bash
cd "$HOME/proyecto"

openssl rand -hex 32
openssl rand -hex 32
openssl rand -hex 32
```

Crear:

``` bash
nano .env
```

Contenido base:

``` dotenv
# ============================================================
# PROYECTO IA LOCAL
# ============================================================

TZ=Europe/Madrid

# ============================================================
# RED
# ============================================================

DOCKER_NETWORK=red-ia

# ============================================================
# OLLAMA
# ============================================================

OLLAMA_HOST=0.0.0.0:11434

# Modelo conversacional inicial.
# Sustituir por el modelo realmente instalado.
OLLAMA_MODEL=qwen3:8b

# Modelo de embeddings para RAG.
OLLAMA_EMBED_MODEL=nomic-embed-text

# ============================================================
# OPEN WEBUI
# ============================================================

WEBUI_SECRET_KEY=CAMBIAR_POR_UN_SECRETO_LARGO

OLLAMA_BASE_URL=http://ollama:11434

ENABLE_WEB_SEARCH=true
WEB_SEARCH_ENGINE=searxng
SEARXNG_QUERY_URL=http://searxng:8080/search
SEARXNG_LANGUAGE=all

ENABLE_IMAGE_GENERATION=true
IMAGE_GENERATION_ENGINE=comfyui
COMFYUI_BASE_URL=http://comfyui:8188

# ============================================================
# HERMES
# ============================================================

HERMES_API_SERVER_ENABLED=true
HERMES_API_SERVER_HOST=0.0.0.0
HERMES_API_SERVER_KEY=CAMBIAR_POR_UN_SECRETO_LARGO

HERMES_MODEL=qwen3:8b
HERMES_OLLAMA_BASE_URL=http://ollama:11434/v1

# ============================================================
# OPENCODE
# ============================================================

OPENCODE_SERVER_USERNAME=opencode
OPENCODE_SERVER_PASSWORD=CAMBIAR_POR_UN_SECRETO_LARGO

# ============================================================
# SEARXNG
# ============================================================

SEARXNG_SECRET=CAMBIAR_POR_UN_SECRETO_LARGO
SEARXNG_BASE_URL=http://searxng:8080/
SEARXNG_LIMITER=true
SEARXNG_VALKEY_URL=valkey://valkey:6379/0

# ============================================================
# RAG
# ============================================================

RAG_OLLAMA_URL=http://ollama:11434
RAG_CHAT_MODEL=qwen3:8b
RAG_EMBED_MODEL=nomic-embed-text
RAG_TOP_K=5
RAG_CHUNK_SIZE=1200
RAG_CHUNK_OVERLAP=200

# ============================================================
# IDENTIDAD HOST
# ============================================================

PUID=1000
PGID=1000
```

Proteger:

``` bash
chmod 600 .env
```

> Nunca subir `.env` a Git.

------------------------------------------------------------------------

# 10. `.gitignore`

Crear:

``` bash
nano .gitignore
```

Contenido:

``` gitignore
.env
*.key
*.pem
*.token

data/
backups/

__pycache__/
*.pyc

.DS_Store
```

------------------------------------------------------------------------

# 11. Crear la red Docker

La red debe existir antes de lanzar los Compose independientes:

``` bash
docker network inspect red-ia >/dev/null 2>&1 || \
docker network create --driver bridge red-ia
```

Comprobar:

``` bash
docker network inspect red-ia
```

El DNS interno de Docker permitirá:

``` text
ollama
openwebui
hermes-agent
opencode
comfyui
yolo
searxng
rag
valkey
```

No utilizar IPs internas fijas.

------------------------------------------------------------------------

# 12. Compose de Ollama

Crear:

``` bash
nano docker-ollama.yml
```

Contenido:

``` yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    hostname: ollama
    restart: unless-stopped

    ports:
      - "11434:11434"

    environment:
      TZ: ${TZ}
      OLLAMA_HOST: ${OLLAMA_HOST}

    volumes:
      - ./data/ollama:/root/.ollama

    gpus: all

    networks:
      - red-ia

networks:
  red-ia:
    external: true
    name: ${DOCKER_NETWORK}
```

Validar:

``` bash
docker compose -f docker-ollama.yml config
```

Arrancar:

``` bash
docker compose -f docker-ollama.yml up -d
```

Verificar:

``` bash
docker ps --filter name=ollama
docker logs --tail 100 ollama
curl http://127.0.0.1:11434/api/tags
```

------------------------------------------------------------------------

# 13. Instalar modelos en Ollama

Entrar al contenedor:

``` bash
docker exec -it ollama bash
```

Descargar modelo:

``` bash
ollama pull qwen3:8b
```

Modelo de embeddings:

``` bash
ollama pull nomic-embed-text
```

Salir:

``` bash
exit
```

Verificar:

``` bash
docker exec ollama ollama list
```

Prueba:

``` bash
curl http://127.0.0.1:11434/api/tags | jq
```

Prueba de chat:

``` bash
curl http://127.0.0.1:11434/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3:8b",
    "messages": [
      {
        "role": "user",
        "content": "Responde exactamente: GPU OK"
      }
    ],
    "stream": false
  }'
```

------------------------------------------------------------------------

# 14. Compose de Open WebUI

Crear:

``` bash
nano docker-openwebui.yml
```

Contenido:

``` yaml
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:cuda
    container_name: openwebui
    hostname: openwebui
    restart: unless-stopped

    ports:
      - "3000:8080"

    environment:
      TZ: ${TZ}

      WEBUI_SECRET_KEY: ${WEBUI_SECRET_KEY}

      ENABLE_OLLAMA_API: "true"
      OLLAMA_BASE_URL: ${OLLAMA_BASE_URL}

      ENABLE_WEB_SEARCH: ${ENABLE_WEB_SEARCH}
      WEB_SEARCH_ENGINE: ${WEB_SEARCH_ENGINE}
      SEARXNG_QUERY_URL: ${SEARXNG_QUERY_URL}
      SEARXNG_LANGUAGE: ${SEARXNG_LANGUAGE}

      ENABLE_IMAGE_GENERATION: ${ENABLE_IMAGE_GENERATION}
      IMAGE_GENERATION_ENGINE: ${IMAGE_GENERATION_ENGINE}
      COMFYUI_BASE_URL: ${COMFYUI_BASE_URL}

    volumes:
      - ./data/openwebui:/app/backend/data

    gpus: all

    depends_on:
      - ollama

    networks:
      - red-ia

networks:
  red-ia:
    external: true
    name: ${DOCKER_NETWORK}
```

Validar:

``` bash
docker compose -f docker-openwebui.yml config
```

Arrancar:

``` bash
docker compose -f docker-openwebui.yml up -d
```

Abrir:

``` text
http://IP_DEL_SERVIDOR:3000
```

------------------------------------------------------------------------

# 15. Configurar Ollama en Open WebUI

Desde Open WebUI:

1.  Crear el primer usuario administrador.
2.  Ir a configuración de administración.
3.  Comprobar la conexión con Ollama.
4.  La URL interna debe ser:

``` text
http://ollama:11434
```

No utilizar:

``` text
http://localhost:11434
```

porque dentro del contenedor `localhost` significa el propio contenedor
de Open WebUI.

Comprobar desde Open WebUI:

``` bash
docker exec openwebui \
  python -c "import urllib.request; print(urllib.request.urlopen('http://ollama:11434/api/tags').read().decode())"
```

------------------------------------------------------------------------

# 16. Compose de Hermes Agent

Hermes Agent dispone de una imagen Docker oficial de Nous Research y
guarda su estado persistente en `/opt/data`.

Crear:

``` bash
nano docker-hermes-agent.yml
```

Contenido:

``` yaml
services:
  hermes-agent:
    image: docker.io/nousresearch/hermes-agent:latest
    container_name: hermes-agent
    hostname: hermes-agent
    restart: unless-stopped

    command:
      - gateway
      - run

    ports:
      # Host 8000 -> Hermes 8642
      - "8000:8642"

    environment:
      TZ: ${TZ}

      API_SERVER_ENABLED: ${HERMES_API_SERVER_ENABLED}
      API_SERVER_HOST: ${HERMES_API_SERVER_HOST}
      API_SERVER_KEY: ${HERMES_API_SERVER_KEY}

      HERMES_UID: ${PUID}
      HERMES_GID: ${PGID}

    volumes:
      - ./data/hermes:/opt/data

    shm_size: "1g"

    networks:
      - red-ia

networks:
  red-ia:
    external: true
    name: ${DOCKER_NETWORK}
```

Validar:

``` bash
docker compose -f docker-hermes-agent.yml config
```

Arrancar:

``` bash
docker compose -f docker-hermes-agent.yml up -d
```

Ver logs:

``` bash
docker logs -f hermes-agent
```

------------------------------------------------------------------------

# 17. Configurar Hermes para Ollama

La configuración debe apuntar al nombre DNS Docker:

``` text
http://ollama:11434/v1
```

Nunca:

``` text
http://localhost:11434/v1
```

Para configurar interactivamente:

``` bash
docker exec -it hermes-agent hermes model
```

Seleccionar proveedor personalizado:

``` text
Provider: Custom endpoint
Base URL: http://ollama:11434/v1
API key: ollama
Model: qwen3:8b
```

Comprobar que Ollama es visible:

``` bash
docker exec hermes-agent \
  curl -s http://ollama:11434/api/tags | jq
```

------------------------------------------------------------------------

# 18. Integrar Hermes con Open WebUI

Hermes expone un API compatible con OpenAI cuando se habilita el API
server.

Desde Open WebUI se puede añadir una conexión OpenAI-compatible:

``` text
URL:
http://hermes-agent:8642/v1

API key:
<valor de HERMES_API_SERVER_KEY>
```

El puerto publicado para acceso externo es:

``` text
http://IP_DEL_SERVIDOR:8000
```

Dentro de Docker se utiliza:

``` text
http://hermes-agent:8642
```

------------------------------------------------------------------------

# 19. Compose de OpenCode

OpenCode utiliza actualmente 4096 como puerto de servidor habitual. Se
publicará en el host como 8443 para respetar la especificación del
proyecto.

Crear:

``` bash
nano docker-opencode.yml
```

Contenido:

``` yaml
services:
  opencode:
    image: ghcr.io/anomalyco/opencode:latest
    container_name: opencode
    hostname: opencode
    restart: unless-stopped

    command:
      - web
      - --hostname
      - 0.0.0.0
      - --port
      - "4096"

    ports:
      - "8443:4096"

    environment:
      OPENCODE_SERVER_USERNAME: ${OPENCODE_SERVER_USERNAME}
      OPENCODE_SERVER_PASSWORD: ${OPENCODE_SERVER_PASSWORD}

    working_dir: /workspace

    volumes:
      - ./data/opencode:/home/ubuntu/.local
      - ./config/opencode:/root/.config/opencode
      - ./data/opencode/workspace:/workspace

    networks:
      - red-ia

networks:
  red-ia:
    external: true
    name: ${DOCKER_NETWORK}
```

Validar:

``` bash
docker compose -f docker-opencode.yml config
```

Arrancar:

``` bash
docker compose -f docker-opencode.yml up -d
```

Abrir:

``` text
http://IP_DEL_SERVIDOR:8443
```

### Seguridad

No ejecutar OpenCode sin contraseña si se publica en una red no
confiable.

------------------------------------------------------------------------

# 20. Configurar OpenCode para Ollama

OpenCode necesita un proveedor compatible con su versión instalada. El
principio de conexión es:

``` text
http://ollama:11434/v1
```

El modelo debe coincidir exactamente con el modelo existente:

``` bash
docker exec ollama ollama list
```

Si se utiliza un archivo `opencode.json`, mantener la configuración del
servidor separada de las credenciales y adaptar el bloque de proveedor a
la versión instalada.

Ejemplo conceptual:

``` json
{
  "$schema": "https://opencode.ai/config.json",
  "server": {
    "port": 4096,
    "hostname": "0.0.0.0"
  }
}
```

**Nota:** no fijar aquí un esquema de proveedor antiguo sin comprobar la
versión concreta de OpenCode. Las opciones de proveedores cambian con
rapidez.

------------------------------------------------------------------------

# 21. Compose de ComfyUI

ComfyUI no se debe tratar como un servidor LLM. Utiliza modelos de
difusión/visión propios.

El ejemplo siguiente utiliza una imagen comunitaria de ComfyUI con
soporte NVIDIA.

Crear:

``` bash
nano docker-comfyui.yml
```

Contenido:

``` yaml
services:
  comfyui:
    image: ghcr.io/lecode-official/comfyui-docker:latest
    container_name: comfyui
    hostname: comfyui
    restart: unless-stopped

    ports:
      - "8188:8188"

    environment:
      TZ: ${TZ}
      WANTED_UID: ${PUID}
      WANTED_GID: ${PGID}

    volumes:
      - ./data/comfyui:/comfy/mnt

    gpus: all

    networks:
      - red-ia

networks:
  red-ia:
    external: true
    name: ${DOCKER_NETWORK}
```

Validar:

``` bash
docker compose -f docker-comfyui.yml config
```

Arrancar:

``` bash
docker compose -f docker-comfyui.yml up -d
```

Ver:

``` bash
docker logs -f comfyui
```

Acceso:

``` text
http://IP_DEL_SERVIDOR:8188
```

------------------------------------------------------------------------

# 22. Modelos de ComfyUI

No descargar modelos indiscriminadamente.

Primero decidir qué familia se utilizará:

-   SDXL.
-   Flux.
-   Qwen Image.
-   Otros modelos compatibles con la GPU disponible.

La estructura típica:

``` text
data/comfyui/
├── models/
│   ├── checkpoints/
│   ├── clip/
│   ├── text_encoders/
│   ├── diffusion_models/
│   ├── vae/
│   ├── loras/
│   └── controlnet/
├── input/
├── output/
├── user/
└── custom_nodes/
```

Comprobar espacio:

``` bash
df -h "$HOME/proyecto"
du -sh "$HOME/proyecto/data/comfyui"
```

------------------------------------------------------------------------

# 23. Integrar ComfyUI con Open WebUI

En Open WebUI:

``` text
Settings
  -> Admin
    -> Experience / Images
```

Activar generación de imágenes.

Seleccionar:

``` text
Image Generation Engine = ComfyUI
```

URL:

``` text
http://comfyui:8188
```

Si la configuración se realiza mediante `.env`:

``` dotenv
ENABLE_IMAGE_GENERATION=true
IMAGE_GENERATION_ENGINE=comfyui
COMFYUI_BASE_URL=http://comfyui:8188
```

El workflow de ComfyUI debe guardarse en formato **API Format** y
después mapear:

-   Prompt.
-   Modelo.
-   Width.
-   Height.
-   Steps.
-   Seed.

La correspondencia exacta depende del workflow y de sus IDs de nodo.

------------------------------------------------------------------------

# 24. Compose de YOLO

Se utiliza la imagen oficial de Ultralytics.

Crear:

``` bash
nano docker-yolo.yml
```

Contenido:

``` yaml
services:
  yolo:
    image: ultralytics/ultralytics:latest
    container_name: yolo
    hostname: yolo
    restart: unless-stopped

    working_dir: /workspace

    ports:
      - "5000:5000"

    volumes:
      - ./data/yolo:/workspace

    gpus: all

    ipc: host

    command:
      - sh
      - -c
      - |
        python -m http.server 5000 --bind 0.0.0.0

    networks:
      - red-ia

networks:
  red-ia:
    external: true
    name: ${DOCKER_NETWORK}
```

### Importante

Este Compose proporciona el contenedor Ultralytics y un servidor HTTP
básico para verificar conectividad. **No es todavía una API REST de
inferencia YOLO.**

Para inferencia:

``` bash
docker exec -it yolo yolo predict \
  model=yolo26n.pt \
  source=/workspace/datasets \
  project=/workspace/runs
```

La API de producción debería ser un servicio FastAPI/Flask específico si
se necesita que otros contenedores envíen imágenes HTTP a YOLO.

------------------------------------------------------------------------

# 25. Prueba GPU de YOLO

``` bash
docker exec yolo python -c \
"import torch; print('CUDA:', torch.cuda.is_available()); print('GPU:', torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'NONE')"
```

Resultado esperado:

``` text
CUDA: True
GPU: NVIDIA ...
```

También:

``` bash
docker exec yolo yolo checks
```

------------------------------------------------------------------------

# 26. Compose de SearXNG

SearXNG requiere configuración persistente. La versión moderna utiliza
Valkey para ciertas funciones del limiter.

Crear:

``` bash
nano docker-searxng.yml
```

Contenido:

``` yaml
services:
  searxng:
    image: docker.io/searxng/searxng:latest
    container_name: searxng
    hostname: searxng
    restart: unless-stopped

    ports:
      - "8080:8080"

    environment:
      SEARXNG_BASE_URL: ${SEARXNG_BASE_URL}
      SEARXNG_SECRET: ${SEARXNG_SECRET}
      SEARXNG_LIMITER: ${SEARXNG_LIMITER}
      SEARXNG_VALKEY_URL: ${SEARXNG_VALKEY_URL}

    volumes:
      - ./config/searxng:/etc/searxng
      - ./data/searxng:/var/cache/searxng

    depends_on:
      - valkey

    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETGID
      - SETUID
      - DAC_OVERRIDE

    networks:
      - red-ia

  valkey:
    image: docker.io/valkey/valkey:8-alpine
    container_name: valkey
    hostname: valkey
    restart: unless-stopped

    volumes:
      - ./data/searxng/valkey:/data

    networks:
      - red-ia

networks:
  red-ia:
    external: true
    name: ${DOCKER_NETWORK}
```

------------------------------------------------------------------------

# 27. Configuración inicial de SearXNG

Crear:

``` bash
nano config/searxng/settings.yml
```

Contenido mínimo:

``` yaml
use_default_settings: true

general:
  debug: false
  instance_name: "SearXNG - AI Local"

search:
  safe_search: 1
  autocomplete: ""
  formats:
    - html
    - json

server:
  secret_key: "CAMBIAR_MEDIANTE_SEARXNG_SECRET"
  limiter: true
  image_proxy: true
  port: 8080
  bind_address: "0.0.0.0"

valkey:
  url: valkey://valkey:6379/0
```

El secreto real debe venir del entorno:

``` bash
grep SEARXNG_SECRET .env
```

Crear el fichero del limiter si se desea personalizar:

``` bash
nano config/searxng/limiter.toml
```

Ejemplo conservador:

``` toml
[botdetection.ip_limit]
link_token = false

[botdetection.ip_lists]
block_ip = []
pass_ip = []
```

Arrancar:

``` bash
docker compose -f docker-searxng.yml up -d
```

Verificar:

``` bash
docker ps --filter name=searxng
docker ps --filter name=valkey
```

Prueba HTML:

``` bash
curl -I http://127.0.0.1:8080
```

Prueba JSON:

``` bash
curl -G \
  --data-urlencode "q=Docker NVIDIA" \
  --data-urlencode "format=json" \
  http://127.0.0.1:8080/search | jq
```

Si aparece `403`, revisar que `json` esté incluido en:

``` yaml
search:
  formats:
    - html
    - json
```

------------------------------------------------------------------------

# 28. Integrar SearXNG con Open WebUI

Open WebUI debe utilizar el DNS Docker:

``` text
http://searxng:8080/search
```

Variables:

``` dotenv
ENABLE_WEB_SEARCH=true
WEB_SEARCH_ENGINE=searxng
SEARXNG_QUERY_URL=http://searxng:8080/search
SEARXNG_LANGUAGE=all
```

Recrear Open WebUI:

``` bash
docker compose -f docker-openwebui.yml up -d --force-recreate
```

Probar desde el contenedor:

``` bash
docker exec openwebui \
  python -c "import urllib.request; u='http://searxng:8080/search?q=test&format=json'; print(urllib.request.urlopen(u).read()[:1000])"
```

En la interfaz:

``` text
Settings
  -> Admin
    -> Web Search
```

Seleccionar SearXNG y activar Web Search.

------------------------------------------------------------------------

# 29. Servicio RAG

## 29.1 Decisión

La especificación original define RAG como concepto y reserva el puerto
11434, pero no identifica una aplicación RAG concreta.

Por ello este manual implementa un servicio RAG pequeño y controlable:

``` text
Documentos
   |
   v
RAG API
   |
   +----> Ollama embeddings
   |
   +----> índice local
   |
   +----> Ollama chat
```

Esto evita introducir una base de datos vectorial adicional en la
primera versión.

## 29.2 Funciones

El servicio tendrá:

``` text
GET  /health
POST /ingest
POST /query
```

Soportará inicialmente:

-   TXT.
-   MD. 
-   PDF.
-   DOCX.

Los documentos se guardarán en:

``` text
data/rag/documents
```

El índice:

``` text
data/rag/index
```

------------------------------------------------------------------------

# 30. Código del RAG

Crear:

``` bash
nano config/rag/app.py
```

Contenido:

``` python
import json
import os
import re
from pathlib import Path
from typing import List

import numpy as np
import requests
from fastapi import FastAPI, File, UploadFile, HTTPException
from pydantic import BaseModel

try:
    from pypdf import PdfReader
except Exception:
    PdfReader = None

try:
    from docx import Document
except Exception:
    Document = None


APP = FastAPI(title="Local RAG API")

OLLAMA_URL = os.getenv("RAG_OLLAMA_URL", "http://ollama:11434").rstrip("/")
CHAT_MODEL = os.getenv("RAG_CHAT_MODEL", "qwen3:8b")
EMBED_MODEL = os.getenv("RAG_EMBED_MODEL", "nomic-embed-text")

DATA_DIR = Path("/data")
DOCS_DIR = DATA_DIR / "documents"
INDEX_DIR = DATA_DIR / "index"
INDEX_FILE = INDEX_DIR / "index.json"

TOP_K = int(os.getenv("RAG_TOP_K", "5"))
CHUNK_SIZE = int(os.getenv("RAG_CHUNK_SIZE", "1200"))
CHUNK_OVERLAP = int(os.getenv("RAG_CHUNK_OVERLAP", "200"))

DOCS_DIR.mkdir(parents=True, exist_ok=True)
INDEX_DIR.mkdir(parents=True, exist_ok=True)


class QueryRequest(BaseModel):
    question: str
    top_k: int = TOP_K


def clean_text(text: str) -> str:
    text = text.replace("\x00", " ")
    text = re.sub(r"\s+", " ", text)
    return text.strip()


def chunk_text(text: str) -> List[str]:
    text = clean_text(text)
    chunks = []
    start = 0

    while start < len(text):
        end = min(len(text), start + CHUNK_SIZE)
        chunks.append(text[start:end])

        if end >= len(text):
            break

        start = max(0, end - CHUNK_OVERLAP)

    return chunks


def read_document(path: Path) -> str:
    suffix = path.suffix.lower()

    if suffix in [".txt", ".md", ".markdown"]:
        return path.read_text(encoding="utf-8", errors="ignore")

    if suffix == ".pdf":
        if PdfReader is None:
            raise RuntimeError("pypdf no está instalado")
        reader = PdfReader(str(path))
        return "\n".join(page.extract_text() or "" for page in reader.pages)

    if suffix == ".docx":
        if Document is None:
            raise RuntimeError("python-docx no está instalado")
        doc = Document(str(path))
        return "\n".join(p.text for p in doc.paragraphs)

    raise ValueError(f"Formato no soportado: {suffix}")


def embed(texts: List[str]) -> List[List[float]]:
    response = requests.post(
        f"{OLLAMA_URL}/api/embed",
        json={
            "model": EMBED_MODEL,
            "input": texts,
        },
        timeout=300,
    )
    response.raise_for_status()
    return response.json()["embeddings"]


def load_index():
    if not INDEX_FILE.exists():
        return []

    return json.loads(INDEX_FILE.read_text(encoding="utf-8"))


def save_index(items):
    INDEX_FILE.write_text(
        json.dumps(items, ensure_ascii=False),
        encoding="utf-8",
    )


def cosine(a, b):
    a = np.asarray(a, dtype=np.float32)
    b = np.asarray(b, dtype=np.float32)

    na = np.linalg.norm(a)
    nb = np.linalg.norm(b)

    if na == 0 or nb == 0:
        return 0.0

    return float(np.dot(a, b) / (na * nb))


@APP.get("/health")
def health():
    return {
        "status": "ok",
        "ollama": OLLAMA_URL,
        "chat_model": CHAT_MODEL,
        "embed_model": EMBED_MODEL,
    }


@APP.post("/ingest")
async def ingest(file: UploadFile = File(...)):
    filename = Path(file.filename or "document.bin").name
    destination = DOCS_DIR / filename

    data = await file.read()
    destination.write_bytes(data)

    try:
        text = read_document(destination)
        chunks = chunk_text(text)

        if not chunks:
            raise HTTPException(
                status_code=400,
                detail="El documento no contiene texto extraíble",
            )

        vectors = embed(chunks)

        index = load_index()

        # Elimina entradas anteriores del mismo documento.
        index = [
            item for item in index
            if item.get("source") != filename
        ]

        for chunk, vector in zip(chunks, vectors):
            index.append(
                {
                    "source": filename,
                    "text": chunk,
                    "embedding": vector,
                }
            )

        save_index(index)

        return {
            "status": "indexed",
            "source": filename,
            "chunks": len(chunks),
        }

    except Exception as exc:
        raise HTTPException(
            status_code=500,
            detail=str(exc),
        )


@APP.post("/query")
def query(request: QueryRequest):
    index = load_index()

    if not index:
        raise HTTPException(
            status_code=404,
            detail="El índice está vacío",
        )

    question_vector = embed([request.question])[0]

    ranked = sorted(
        (
            {
                **item,
                "score": cosine(
                    question_vector,
                    item["embedding"],
                ),
            }
            for item in index
        ),
        key=lambda x: x["score"],
        reverse=True,
    )

    selected = ranked[:request.top_k]

    context = "\n\n".join(
        f"[Fuente: {item['source']}]\n{item['text']}"
        for item in selected
    )

    prompt = f"""
Utiliza únicamente el contexto proporcionado para contestar.
Si el contexto no contiene la respuesta, dilo claramente.

CONTEXTO:
{context}

PREGUNTA:
{request.question}
"""

    response = requests.post(
        f"{OLLAMA_URL}/api/chat",
        json={
            "model": CHAT_MODEL,
            "messages": [
                {
                    "role": "user",
                    "content": prompt,
                }
            ],
            "stream": False,
        },
        timeout=600,
    )
    response.raise_for_status()

    answer = response.json()["message"]["content"]

    return {
        "answer": answer,
        "sources": [
            {
                "source": item["source"],
                "score": item["score"],
            }
            for item in selected
        ],
    }
```

------------------------------------------------------------------------

# 31. Dockerfile del RAG

Crear:

``` bash
nano config/rag/Dockerfile
```

Contenido:

``` dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN pip install --no-cache-dir \
    fastapi \
    uvicorn[standard] \
    requests \
    numpy \
    python-multipart \
    pypdf \
    python-docx

COPY app.py /app/app.py

EXPOSE 8000

CMD ["uvicorn", "app:APP", "--host", "0.0.0.0", "--port", "8000"]
```

------------------------------------------------------------------------

# 32. Compose del RAG

Crear:

``` bash
nano docker-rag.yml
```

Contenido:

``` yaml
services:
  rag:
    build:
      context: ./config/rag
      dockerfile: Dockerfile

    image: local/rag:1.0

    container_name: rag
    hostname: rag
    restart: unless-stopped

    ports:
      # RAG no necesita exposición pública.
      # 8001 sirve únicamente para pruebas desde el host.
      - "127.0.0.1:8001:8000"

    environment:
      TZ: ${TZ}

      RAG_OLLAMA_URL: ${RAG_OLLAMA_URL}
      RAG_CHAT_MODEL: ${RAG_CHAT_MODEL}
      RAG_EMBED_MODEL: ${RAG_EMBED_MODEL}
      RAG_TOP_K: ${RAG_TOP_K}
      RAG_CHUNK_SIZE: ${RAG_CHUNK_SIZE}
      RAG_CHUNK_OVERLAP: ${RAG_CHUNK_OVERLAP}

    volumes:
      - ./data/rag:/data

    depends_on:
      - ollama

    networks:
      - red-ia

networks:
  red-ia:
    external: true
    name: ${DOCKER_NETWORK}
```

Construir:

``` bash
docker compose -f docker-rag.yml build
```

Arrancar:

``` bash
docker compose -f docker-rag.yml up -d
```

Probar:

``` bash
curl http://127.0.0.1:8001/health | jq
```

------------------------------------------------------------------------

# 33. Probar RAG

Crear documento:

``` bash
cat > data/rag/documents/prueba.md <<'EOF'
# Proyecto IA

El stack utiliza Docker sobre Ubuntu Server.
Ollama proporciona el servidor de modelos.
Open WebUI proporciona la interfaz web.
SearXNG proporciona búsqueda web privada.
EOF
```

Indexar:

``` bash
curl -X POST \
  -F "file=@data/rag/documents/prueba.md" \
  http://127.0.0.1:8001/ingest | jq
```

Consultar:

``` bash
curl -X POST \
  http://127.0.0.1:8001/query \
  -H "Content-Type: application/json" \
  -d '{
    "question": "¿Qué proporciona Ollama?",
    "top_k": 3
  }' | jq
```

------------------------------------------------------------------------

# 34. Orden correcto de despliegue

No lanzar todo a la vez la primera vez.

Orden recomendado:

``` text
1. Ubuntu
2. NVIDIA driver
3. Docker
4. NVIDIA Container Toolkit
5. red-ia
6. Ollama
7. modelos Ollama
8. SearXNG + Valkey
9. Open WebUI
10. Hermes
11. OpenCode
12. ComfyUI
13. YOLO
14. RAG
15. pruebas de integración
```

------------------------------------------------------------------------

# 35. Script de arranque

Crear:

``` bash
nano start-stack.sh
```

Contenido:

``` bash
#!/usr/bin/env bash
set -euo pipefail

cd "$(dirname "$0")"

docker network inspect red-ia >/dev/null 2>&1 || \
  docker network create --driver bridge red-ia

echo "[1/8] Ollama"
docker compose -f docker-ollama.yml up -d

echo "[2/8] SearXNG"
docker compose -f docker-searxng.yml up -d

echo "[3/8] Open WebUI"
docker compose -f docker-openwebui.yml up -d

echo "[4/8] Hermes"
docker compose -f docker-hermes-agent.yml up -d

echo "[5/8] OpenCode"
docker compose -f docker-opencode.yml up -d

echo "[6/8] ComfyUI"
docker compose -f docker-comfyui.yml up -d

echo "[7/8] YOLO"
docker compose -f docker-yolo.yml up -d

echo "[8/8] RAG"
docker compose -f docker-rag.yml up -d

echo
echo "Stack iniciado."
docker ps
```

Permisos:

``` bash
chmod +x start-stack.sh
```

Ejecutar:

``` bash
./start-stack.sh
```

------------------------------------------------------------------------

# 36. Script de parada

Crear:

``` bash
nano stop-stack.sh
```

``` bash
#!/usr/bin/env bash
set -euo pipefail

cd "$(dirname "$0")"

docker compose -f docker-rag.yml down
docker compose -f docker-yolo.yml down
docker compose -f docker-comfyui.yml down
docker compose -f docker-opencode.yml down
docker compose -f docker-hermes-agent.yml down
docker compose -f docker-openwebui.yml down
docker compose -f docker-searxng.yml down
docker compose -f docker-ollama.yml down
```

Permisos:

``` bash
chmod +x stop-stack.sh
```

------------------------------------------------------------------------

# 37. Verificación global

## 37.1 Contenedores

``` bash
docker ps
```

Esperado:

``` text
ollama
openwebui
hermes-agent
opencode
comfyui
yolo
searxng
valkey
rag
```

## 37.2 Estado Compose

``` bash
for f in docker-*.yml; do
  echo "===== $f ====="
  docker compose -f "$f" ps
done
```

## 37.3 Redes

``` bash
docker network inspect red-ia
```

Todos los contenedores principales deben aparecer conectados.

------------------------------------------------------------------------

# 38. Verificación de puertos

En el host:

``` bash
sudo ss -lntp
```

Filtrar:

``` bash
sudo ss -lntp | grep -E '3000|5000|8000|8001|8080|8188|8443|11434'
```

Resultado esperado conceptualmente:

``` text
3000   Open WebUI
5000   YOLO
8000   Hermes
8001   RAG localhost
8080   SearXNG
8188   ComfyUI
8443   OpenCode
11434  Ollama
```

------------------------------------------------------------------------

# 39. Verificación GPU global

Host:

``` bash
nvidia-smi
```

Mientras se genera una respuesta o imagen:

``` bash
watch -n 1 nvidia-smi
```

Contenedores GPU:

``` bash
docker exec ollama nvidia-smi
docker exec comfyui nvidia-smi
docker exec yolo nvidia-smi
```

En Open WebUI:

``` bash
docker exec openwebui nvidia-smi
```

Si alguno falla:

``` bash
docker inspect <contenedor> | grep -i -A5 -B5 gpu
```

------------------------------------------------------------------------

# 40. Prueba de conectividad entre contenedores

Desde Open WebUI:

``` bash
docker exec openwebui \
  python -c "import urllib.request; print(urllib.request.urlopen('http://ollama:11434/api/tags').status)"
```

SearXNG:

``` bash
docker exec openwebui \
  python -c "import urllib.request; print(urllib.request.urlopen('http://searxng:8080').status)"
```

ComfyUI:

``` bash
docker exec openwebui \
  python -c "import urllib.request; print(urllib.request.urlopen('http://comfyui:8188').status)"
```

RAG:

``` bash
docker exec openwebui \
  python -c "import urllib.request; print(urllib.request.urlopen('http://rag:8000/health').read().decode())"
```

------------------------------------------------------------------------

# 41. Diagnóstico de DNS Docker

Desde un contenedor:

``` bash
docker exec openwebui getent hosts ollama
docker exec openwebui getent hosts searxng
docker exec openwebui getent hosts comfyui
docker exec openwebui getent hosts rag
```

Debe devolver una IP privada de la red `red-ia`.

Si no:

``` bash
docker network inspect red-ia
```

------------------------------------------------------------------------

# 42. Logs

Ollama:

``` bash
docker logs --tail 200 ollama
```

Open WebUI:

``` bash
docker logs --tail 200 openwebui
```

Hermes:

``` bash
docker logs --tail 200 hermes-agent
```

OpenCode:

``` bash
docker logs --tail 200 opencode
```

ComfyUI:

``` bash
docker logs --tail 200 comfyui
```

YOLO:

``` bash
docker logs --tail 200 yolo
```

SearXNG:

``` bash
docker logs --tail 200 searxng
```

RAG:

``` bash
docker logs --tail 200 rag
```

Seguir:

``` bash
docker logs -f <contenedor>
```

------------------------------------------------------------------------

# 43. Diagnóstico por capas

Cuando algo falle, no cambiar cinco cosas a la vez.

Seguir:

``` text
CAPA 1: Host
  |
  +-- GPU
  +-- RAM
  +-- disco
  +-- red
  |
CAPA 2: Docker
  |
  +-- daemon
  +-- red-ia
  +-- contenedor
  |
CAPA 3: Servicio
  |
  +-- proceso
  +-- puerto
  +-- configuración
  |
CAPA 4: Integración
  |
  +-- DNS
  +-- URL
  +-- autenticación
  +-- API
  |
CAPA 5: Aplicación
  |
  +-- modelo
  +-- workflow
  +-- permisos
```

------------------------------------------------------------------------

# 44. Problemas comunes

## 44.1 `permission denied`

Comprobar:

``` bash
ls -la data/
```

Corregir propietario para directorios gestionados por el usuario:

``` bash
sudo chown -R "$USER:$USER" "$HOME/proyecto/data"
```

No utilizar:

``` bash
chmod -R 777 .
```

como solución permanente.

------------------------------------------------------------------------

## 44.2 Ollama no ve la GPU

Host:

``` bash
nvidia-smi
```

Docker:

``` bash
docker run --rm --gpus all \
  nvidia/cuda:12.6.2-base-ubuntu24.04 \
  nvidia-smi
```

Toolkit:

``` bash
nvidia-ctk --version
```

Configuración:

``` bash
cat /etc/docker/daemon.json
```

Reconfigurar:

``` bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

------------------------------------------------------------------------

## 44.3 El contenedor GPU funciona pero pierde la GPU después de actualizar el host

Puede existir interacción entre `systemd`/cgroups y el método clásico
`--gpus all`.

Revisar:

``` bash
docker version
nvidia-ctk --version
nvidia-ctk cdi list
```

Si el entorno cumple requisitos, considerar migrar a CDI:

``` bash
sudo nvidia-ctk cdi generate --output=/var/run/cdi/nvidia.yaml
```

------------------------------------------------------------------------

## 44.4 Open WebUI no conecta con Ollama

No usar:

``` text
http://localhost:11434
```

Usar:

``` text
http://ollama:11434
```

Comprobar:

``` bash
docker exec openwebui \
  curl -s http://ollama:11434/api/tags
```

------------------------------------------------------------------------

## 44.5 Open WebUI no encuentra SearXNG

Comprobar:

``` bash
docker exec openwebui \
  curl -s "http://searxng:8080/search?q=test&format=json"
```

Si falla:

``` bash
docker logs searxng
```

Comprobar:

``` yaml
search:
  formats:
    - html
    - json
```

------------------------------------------------------------------------

## 44.6 SearXNG devuelve 403 a Open WebUI

Causa habitual:

``` text
JSON no habilitado
```

Corregir:

``` yaml
search:
  formats:
    - html
    - json
```

Reiniciar:

``` bash
docker compose -f docker-searxng.yml restart
```

------------------------------------------------------------------------

## 44.7 OpenCode arranca pero no se puede acceder

Comprobar:

``` bash
docker logs opencode
```

Dentro:

``` bash
docker exec opencode sh -c \
  "wget -qO- http://127.0.0.1:4096/global/health || true"
```

Comprobar puerto:

``` bash
sudo ss -lntp | grep 8443
```

Comprobar contraseña:

``` bash
grep OPENCODE_SERVER_PASSWORD .env
```

------------------------------------------------------------------------

## 44.8 Hermes arranca pero no puede utilizar Ollama

Comprobar:

``` bash
docker exec hermes-agent \
  curl -s http://ollama:11434/api/tags
```

Revisar configuración de Hermes:

``` bash
docker exec hermes-agent \
  sh -c 'cat /opt/data/config.yaml 2>/dev/null || true'
```

El endpoint debe ser:

``` text
http://ollama:11434/v1
```

------------------------------------------------------------------------

## 44.9 ComfyUI no ve modelos

Comprobar montaje:

``` bash
docker inspect comfyui \
  --format '{{json .Mounts}}' | jq
```

Comprobar directorio:

``` bash
docker exec comfyui ls -la /comfy/mnt
```

No descargar un modelo en una ruta distinta a la que ComfyUI está
utilizando.

------------------------------------------------------------------------

## 44.10 YOLO utiliza CPU

``` bash
docker exec yolo python -c \
"import torch; print(torch.cuda.is_available())"
```

Si devuelve:

``` text
False
```

revisar:

``` bash
docker inspect yolo
docker exec yolo nvidia-smi
```

------------------------------------------------------------------------

# 45. Mantenimiento

## 45.1 Ver imágenes

``` bash
docker images
```

## 45.2 Espacio Docker

``` bash
docker system df
```

## 45.3 Contenedores detenidos

``` bash
docker ps -a
```

## 45.4 Imágenes huérfanas

``` bash
docker image prune
```

No utilizar automáticamente:

``` bash
docker system prune -a
```

en producción, porque puede eliminar imágenes que se quieran conservar.

------------------------------------------------------------------------

# 46. Actualización de servicios

Antes de actualizar:

``` bash
cd "$HOME/proyecto"

./backup-stack.sh
```

Actualizar Ollama:

``` bash
docker compose -f docker-ollama.yml pull
docker compose -f docker-ollama.yml up -d
```

Open WebUI:

``` bash
docker compose -f docker-openwebui.yml pull
docker compose -f docker-openwebui.yml up -d
```

Hermes:

``` bash
docker compose -f docker-hermes-agent.yml pull
docker compose -f docker-hermes-agent.yml up -d
```

OpenCode:

``` bash
docker compose -f docker-opencode.yml pull
docker compose -f docker-opencode.yml up -d
```

ComfyUI:

``` bash
docker compose -f docker-comfyui.yml pull
docker compose -f docker-comfyui.yml up -d
```

YOLO:

``` bash
docker compose -f docker-yolo.yml pull
docker compose -f docker-yolo.yml up -d
```

SearXNG:

``` bash
docker compose -f docker-searxng.yml pull
docker compose -f docker-searxng.yml up -d
```

RAG:

``` bash
docker compose -f docker-rag.yml build --pull
docker compose -f docker-rag.yml up -d
```

------------------------------------------------------------------------

# 47. Política de actualización recomendada

No actualizar todo el stack automáticamente el mismo día.

Orden:

``` text
1. Backup
2. Ollama
3. Open WebUI
4. SearXNG
5. Hermes
6. OpenCode
7. ComfyUI
8. YOLO
9. RAG
10. pruebas
```

Después de cada actualización:

``` bash
docker ps
```

y:

``` bash
docker logs --tail 50 <servicio>
```

------------------------------------------------------------------------

# 48. Backups

Los datos importantes son:

``` text
.env
config/
data/ollama/
data/openwebui/
data/hermes/
data/opencode/
data/comfyui/
data/yolo/
data/searxng/
data/rag/
```

No es necesario guardar las imágenes Docker si se pueden volver a
descargar.

## 48.1 Script de backup

Crear:

``` bash
nano backup-stack.sh
```

Contenido:

``` bash
#!/usr/bin/env bash
set -euo pipefail

PROJECT="$HOME/proyecto"
DATE="$(date +%Y%m%d-%H%M%S)"
DEST="$PROJECT/backups/$DATE"

mkdir -p "$DEST"

echo "Creando backup en $DEST"

tar \
  --exclude="$PROJECT/backups" \
  --exclude="$PROJECT/data/comfyui/output" \
  --exclude="$PROJECT/data/yolo/runs" \
  -czf "$DEST/proyecto-config-data.tar.gz" \
  -C "$PROJECT" \
  .env \
  config \
  data/ollama \
  data/openwebui \
  data/hermes \
  data/opencode \
  data/comfyui \
  data/yolo \
  data/searxng \
  data/rag

echo "Backup terminado:"
ls -lh "$DEST"
```

Permisos:

``` bash
chmod +x backup-stack.sh
```

Ejecutar:

``` bash
./backup-stack.sh
```

------------------------------------------------------------------------

# 49. Restauración

Detener servicios:

``` bash
./stop-stack.sh
```

Localizar backup:

``` bash
ls -lah backups/
```

Restaurar:

``` bash
tar -xzf \
  backups/AAAAmmdd-HHMMSS/proyecto-config-data.tar.gz \
  -C "$HOME/proyecto"
```

Comprobar permisos:

``` bash
sudo chown -R "$USER:$USER" "$HOME/proyecto"
chmod 600 "$HOME/proyecto/.env"
```

Arrancar:

``` bash
./start-stack.sh
```

------------------------------------------------------------------------

# 50. Backup de modelos Ollama

Los modelos pueden ocupar decenas o cientos de GB.

Antes de copiarlos:

``` bash
du -sh data/ollama
```

Para backup completo:

``` bash
tar -czf \
  backups/ollama-$(date +%Y%m%d).tar.gz \
  data/ollama
```

En servidores con poco espacio, puede ser mejor reconstruir modelos con:

``` bash
ollama pull <modelo>
```

en lugar de copiar todos los blobs.

------------------------------------------------------------------------

# 51. Seguridad de red

## 51.1 UFW

Comprobar:

``` bash
sudo ufw status verbose
```

Si se requiere acceso SSH:

``` bash
sudo ufw allow OpenSSH
```

Permitir sólo los puertos realmente necesarios:

``` bash
sudo ufw allow 3000/tcp
sudo ufw allow 8443/tcp
sudo ufw allow 8188/tcp
```

Evitar exponer directamente a Internet:

``` text
11434 Ollama
8000 Hermes
8080 SearXNG
8001 RAG
5000 YOLO
```

si no es estrictamente necesario.

## 51.2 Escucha sólo local

RAG ya utiliza:

``` yaml
- "127.0.0.1:8001:8000"
```

Esto significa que sólo el host puede acceder directamente.

------------------------------------------------------------------------

# 52. Reverse proxy

Para un entorno más avanzado:

``` text
Internet/LAN
    |
    v
Reverse Proxy
    |
    +-- /        -> Open WebUI
    +-- /code    -> OpenCode
    +-- /images  -> ComfyUI
```

Utilizar TLS.

Opciones:

-   Caddy.
-   Nginx.
-   Traefik.

Para una primera instalación local no es obligatorio.

------------------------------------------------------------------------

# 53. DNS local

El DNS de Docker sólo funciona dentro de las redes Docker.

Dentro de `red-ia`:

``` text
ollama
openwebui
hermes-agent
opencode
comfyui
yolo
searxng
rag
valkey
```

No asumir que desde otro ordenador de la LAN funcionará:

``` text
http://ollama:11434
```

Desde la LAN se utiliza:

``` text
http://IP_DEL_SERVIDOR:11434
```

o, preferiblemente, un nombre DNS real:

``` text
http://ai-server.local:3000
```

------------------------------------------------------------------------

# 54. Integración completa

## 54.1 Open WebUI -\> Ollama

``` text
Open WebUI
    |
    | HTTP
    v
http://ollama:11434
    |
    v
Modelo LLM
```

Prueba:

``` bash
docker exec openwebui \
  curl -s http://ollama:11434/api/tags
```

------------------------------------------------------------------------

## 54.2 Open WebUI -\> SearXNG

``` text
Open WebUI
    |
    v
http://searxng:8080/search
    |
    v
Internet / motores de búsqueda
```

Configuración:

``` text
WEB_SEARCH_ENGINE=searxng
SEARXNG_QUERY_URL=http://searxng:8080/search
```

------------------------------------------------------------------------

## 54.3 Open WebUI -\> ComfyUI

``` text
Open WebUI
    |
    v
http://comfyui:8188
    |
    v
Workflow ComfyUI
    |
    v
NVIDIA GPU
```

Variables:

``` text
ENABLE_IMAGE_GENERATION=true
IMAGE_GENERATION_ENGINE=comfyui
COMFYUI_BASE_URL=http://comfyui:8188
```

------------------------------------------------------------------------

## 54.4 Hermes -\> Ollama

``` text
Hermes
   |
   v
http://ollama:11434/v1
   |
   v
LLM
```

------------------------------------------------------------------------

## 54.5 OpenCode -\> Ollama

``` text
OpenCode
   |
   v
Ollama OpenAI-compatible endpoint
   |
   v
http://ollama:11434/v1
```

------------------------------------------------------------------------

## 54.6 RAG -\> Ollama

Embeddings:

``` text
RAG
 |
 +-- /api/embed -> Ollama
```

Respuesta:

``` text
RAG
 |
 +-- contexto recuperado
 |
 +-- /api/chat -> Ollama
```

------------------------------------------------------------------------

## 54.7 Hermes/OpenCode -\> SearXNG

Cuando una herramienta del agente necesite búsquedas, el endpoint
interno es:

``` text
http://searxng:8080/search
```

No utilizar el puerto del host desde otro contenedor.

------------------------------------------------------------------------

# 55. Matriz de integración

  Origen       Destino      Endpoint                       Requiere GPU
  ------------ ------------ ------------------------------ ----------------
  Open WebUI   Ollama       `http://ollama:11434`          Indirectamente
  Open WebUI   SearXNG      `http://searxng:8080/search`   No
  Open WebUI   ComfyUI      `http://comfyui:8188`          ComfyUI sí
  Hermes       Ollama       `http://ollama:11434/v1`       Indirectamente
  OpenCode     Ollama       `http://ollama:11434/v1`       Indirectamente
  RAG          Ollama       `http://ollama:11434`          Indirectamente
  RAG          documentos   `/data/documents`              No
  YOLO         GPU          NVIDIA CUDA                    Sí
  ComfyUI      GPU          NVIDIA CUDA                    Sí
  Ollama       GPU          NVIDIA CUDA                    Sí

------------------------------------------------------------------------

# 56. Checklist de aceptación

## Requisito 1 --- YAML funcional

Validar todos:

``` bash
for f in docker-*.yml; do
  echo "VALIDANDO $f"
  docker compose -f "$f" config >/dev/null
done
```

Resultado esperado:

``` text
Sin errores
```

------------------------------------------------------------------------

## Requisito 2 --- GPU

``` bash
nvidia-smi
```

Y:

``` bash
docker exec ollama nvidia-smi
docker exec comfyui nvidia-smi
docker exec yolo nvidia-smi
```

------------------------------------------------------------------------

## Requisito 3 --- sin colisiones

``` bash
sudo ss -lntp
```

Puertos:

``` text
3000
5000
8000
8001
8080
8188
8443
11434
```

ninguno debe estar ocupado por otro proceso.

------------------------------------------------------------------------

## Requisito 4 --- documentación para administrador novato

El procedimiento recomendado es:

``` text
1. Instalar Ubuntu
2. Actualizar Ubuntu
3. Instalar driver NVIDIA
4. Probar nvidia-smi
5. Instalar Docker
6. Probar Docker
7. Instalar NVIDIA Container Toolkit
8. Probar GPU dentro de Docker
9. Crear proyecto
10. Crear red
11. Crear .env
12. Desplegar Ollama
13. Descargar modelos
14. Desplegar SearXNG
15. Desplegar Open WebUI
16. Desplegar Hermes
17. Desplegar OpenCode
18. Desplegar ComfyUI
19. Desplegar YOLO
20. Desplegar RAG
21. Ejecutar pruebas
22. Configurar backups
```

------------------------------------------------------------------------

# 57. Comprobación final automatizada

Crear:

``` bash
nano healthcheck.sh
```

Contenido:

``` bash
#!/usr/bin/env bash
set -u

echo "=========================================="
echo " HEALTH CHECK - STACK IA LOCAL"
echo "=========================================="

echo
echo "[HOST]"
hostname
uname -r

echo
echo "[GPU]"
nvidia-smi --query-gpu=name,driver_version,memory.total \
  --format=csv,noheader || true

echo
echo "[DOCKER]"
docker --version
docker compose version

echo
echo "[CONTAINERS]"
docker ps --format \
  'table {{.Names}}\t{{.Status}}\t{{.Ports}}'

echo
echo "[OLLAMA]"
curl -fsS http://127.0.0.1:11434/api/tags >/dev/null \
  && echo "OK" \
  || echo "FAIL"

echo
echo "[OPEN WEBUI]"
curl -fsS http://127.0.0.1:3000 >/dev/null \
  && echo "OK" \
  || echo "FAIL"

echo
echo "[HERMES]"
curl -fsS http://127.0.0.1:8000 >/dev/null \
  && echo "OK/endpoint reachable" \
  || echo "CHECK"

echo
echo "[OPENCODE]"
curl -kfsS http://127.0.0.1:8443 >/dev/null \
  && echo "OK" \
  || echo "CHECK"

echo
echo "[COMFYUI]"
curl -fsS http://127.0.0.1:8188 >/dev/null \
  && echo "OK" \
  || echo "FAIL"

echo
echo "[SEARXNG]"
curl -fsS http://127.0.0.1:8080 >/dev/null \
  && echo "OK" \
  || echo "FAIL"

echo
echo "[RAG]"
curl -fsS http://127.0.0.1:8001/health >/dev/null \
  && echo "OK" \
  || echo "FAIL"

echo
echo "=========================================="
echo " FIN"
echo "=========================================="
```

Ejecutar:

``` bash
chmod +x healthcheck.sh
./healthcheck.sh
```

------------------------------------------------------------------------

# 58. Operación diaria

## Ver estado

``` bash
docker ps
```

## Ver GPU

``` bash
nvidia-smi
```

## Ver consumo

``` bash
docker stats
```

## Ver logs

``` bash
docker logs --tail 100 ollama
docker logs --tail 100 openwebui
docker logs --tail 100 hermes-agent
docker logs --tail 100 opencode
docker logs --tail 100 comfyui
docker logs --tail 100 yolo
docker logs --tail 100 searxng
docker logs --tail 100 rag
```

## Reiniciar un servicio

``` bash
docker restart ollama
```

## Reiniciar Open WebUI

``` bash
docker compose -f docker-openwebui.yml restart
```

------------------------------------------------------------------------

# 59. Gestión de modelos

Listar:

``` bash
docker exec ollama ollama list
```

Descargar:

``` bash
docker exec ollama ollama pull <modelo>
```

Eliminar:

``` bash
docker exec ollama ollama rm <modelo>
```

Probar:

``` bash
docker exec ollama ollama run <modelo>
```

Para un servidor con RTX 3050/4060, comenzar con modelos
pequeños/medianos y aumentar gradualmente.

No asumir que un modelo grande funcionará sólo porque Docker tiene
acceso a la GPU: hay que considerar VRAM, RAM, contexto y cuantización.

------------------------------------------------------------------------

# 60. Recomendaciones de producción

Antes de considerar el sistema "producción":

-   Fijar versiones de imágenes en lugar de `latest`.
-   Preferir digests SHA256 para componentes críticos.
-   Mantener backups externos.
-   Configurar monitorización.
-   Configurar alertas de disco.
-   Configurar UFW.
-   No exponer Ollama directamente a Internet.
-   Proteger OpenCode con contraseña.
-   Proteger Hermes API con una clave larga.
-   Rotar secretos.
-   Mantener `.env` fuera de Git.
-   Documentar qué modelos están instalados.
-   Documentar workflows ComfyUI.
-   Documentar versiones CUDA/driver.
-   Probar restauración de backups.
-   Actualizar un servicio cada vez.
-   Mantener un registro de cambios.

------------------------------------------------------------------------

# 61. Registro de versiones

Crear:

``` bash
nano VERSIONES.md
```

Plantilla:

``` markdown
# Versiones del stack

Fecha: YYYY-MM-DD

## Host

- Ubuntu:
- Kernel:
- NVIDIA Driver:
- GPU:
- Docker:
- Docker Compose:
- NVIDIA Container Toolkit:

## Aplicaciones

- Ollama:
- Open WebUI:
- Hermes Agent:
- OpenCode:
- ComfyUI:
- YOLO/Ultralytics:
- SearXNG:
- Valkey:
- RAG:

## Modelos

### Ollama

- modelo:
- tamaño:
- cuantización:

### ComfyUI

- checkpoint:
- VAE:
- LoRA:
- workflow:

## Notas

- 
```

Obtener versiones:

``` bash
docker exec ollama ollama --version
docker exec yolo yolo version
docker exec opencode opencode --version
```

------------------------------------------------------------------------

# 62. Fuentes técnicas consultadas

La arquitectura y los comandos de instalación deben mantenerse alineados
con la documentación de los proyectos. Las referencias principales
utilizadas para esta versión del manual son:

-   Docker Engine: https://docs.docker.com/engine/install/

-   Docker Compose: https://docs.docker.com/compose/install/linux/

-   NVIDIA Container Toolkit:
    https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html

-   Ollama: https://docs.ollama.com/

-   Open WebUI: https://docs.openwebui.com/

-   Open WebUI + SearXNG:
    https://docs.openwebui.com/features/chat-conversations/web-search/providers/searxng/

-   Open WebUI + ComfyUI:
    https://docs.openwebui.com/features/chat-conversations/image-generation-and-editing/comfyui/

-   SearXNG: https://docs.searxng.org/admin/installation-docker

-   Hermes Agent:
    https://hermes-agent.nousresearch.com/docs/user-guide/docker

-   OpenCode: https://opencode.ai/docs/

-   OpenCode Web: https://opencode.ai/docs/web/

-   Ultralytics Docker:
    https://github.com/ultralytics/ultralytics/blob/main/docs/en/guides/docker-quickstart.md

------------------------------------------------------------------------

# 63. Referencia rápida

## Arrancar

``` bash
cd "$HOME/proyecto"
./start-stack.sh
```

## Parar

``` bash
./stop-stack.sh
```

## Estado

``` bash
docker ps
```

## GPU

``` bash
nvidia-smi
```

## Backup

``` bash
./backup-stack.sh
```

## Health check

``` bash
./healthcheck.sh
```

## Ollama

``` bash
docker exec ollama ollama list
```

## Logs

``` bash
docker logs -f <contenedor>
```

## Revalidar todos los Compose

``` bash
for f in docker-*.yml; do
  docker compose -f "$f" config >/dev/null || exit 1
done
```

------------------------------------------------------------------------

# 64. Estado de cumplimiento

  Criterio                   Estado
  -------------------------- -----------------------------------
  Ubuntu Server              Cubierto
  Docker Engine              Cubierto
  Docker Compose v2          Cubierto
  NVIDIA driver              Cubierto
  NVIDIA Container Toolkit   Cubierto
  Red `red-ia`               Cubierto
  DNS Docker                 Cubierto
  Ollama                     Cubierto
  Open WebUI                 Cubierto
  Hermes Agent               Cubierto
  OpenCode                   Cubierto
  ComfyUI                    Cubierto
  YOLO                       Cubierto
  SearXNG                    Cubierto
  Valkey                     Cubierto como dependencia
  RAG                        Cubierto con implementación local
  `.env`                     Cubierto
  Persistencia               Cubierto
  Backups                    Cubierto
  Actualizaciones            Cubierto
  Troubleshooting            Cubierto
  Integraciones internas     Cubierto
  Pruebas GPU                Cubierto
  Pruebas de red             Cubierto
  Sin colisión de puertos    Cubierto
  Manual para principiante   Cubierto

------------------------------------------------------------------------

# 65. Nota final

Este stack debe considerarse una **primera versión operativa** y no una
plataforma de alta disponibilidad.

La prioridad inicial debe ser:

1.  Obtener GPU funcional.
2.  Obtener Ollama funcional.
3.  Confirmar que un modelo responde.
4.  Conectar Open WebUI.
5.  Añadir SearXNG.
6.  Añadir Hermes y OpenCode.
7.  Añadir ComfyUI.
8.  Añadir YOLO.
9.  Añadir RAG.
10. Automatizar backups y mantenimiento.

No conviene solucionar todos los componentes simultáneamente. Si falla
una etapa, detener el despliegue y corregirla antes de continuar.

El criterio más importante para la operación es mantener siempre clara
la diferencia entre:

``` text
HOST
  IP_DEL_SERVIDOR:PUERTO_PUBLICADO

DOCKER
  nombre-contenedor:puerto-interno
```

Ejemplo:

``` text
Navegador
   |
   v
192.168.1.50:3000
   |
   v
openwebui:8080
   |
   v
ollama:11434
```

Esa separación evita la mayoría de los errores de conectividad de este
proyecto.

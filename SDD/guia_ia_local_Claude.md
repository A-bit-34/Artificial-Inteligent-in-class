---
TITLE: "Manual de instalación, configuración y operación: Stack de IA local con Docker en Ubuntu Server"
AUTHOR: "Jason"
DATE: 2026-10-06
CATEGORY: "Administración de sistemas / IA local"
TAGS: [markdown, ai, docker, ollama, nvidia, SDD]
VERSION: 1.0
---

# Manual: Despliegue de un stack de IA local con Docker en Ubuntu Server

> Manual generado a partir de la especificación SDD `proyecto_ai.md` (v1.0).
> Público objetivo: administrador de sistemas novato. Todos los pasos están explicados en orden.

## Índice

0. [Decisiones sobre la especificación (leer primero)](#0-decisiones-sobre-la-especificación-leer-primero)
1. [Prerrequisitos e instalación base](#1-prerrequisitos-e-instalación-base)
2. [Estructura del proyecto](#2-estructura-del-proyecto)
3. [Ficheros de configuración](#3-ficheros-de-configuración)
4. [Fichero de entorno `.env`](#4-fichero-de-entorno-env)
5. [Despliegue y verificación](#5-despliegue-y-verificación)
6. [Mantenimiento y actualización](#6-mantenimiento-y-actualización)
7. [Guía de integración entre servicios](#7-guía-de-integración-entre-servicios)
8. [Verificación de los criterios de aceptación](#8-verificación-de-los-criterios-de-aceptación)

---

## 0. Decisiones sobre la especificación (leer primero)

Al analizar la especificación aparecen incoherencias que, si no se resuelven, rompen el criterio de aceptación nº 3 (sin colisiones de puertos) o impiden que los contenedores se comuniquen. Estas son las decisiones tomadas en este manual:

| # | Problema en la especificación | Decisión adoptada |
| :-- | :--- | :--- |
| 1 | **RAG** usa el puerto 11434, el mismo que **Ollama** (colisión). | RAG se implementa con **Qdrant** (base de datos vectorial) en el puerto **6333**. Open WebUI lo usa como almacén de vectores. |
| 2 | La red se llama `red-ai` en la sección 3 y `red-ia` en la 3.1. | Se usa **`red-ai`** en todo el manual (configurable con `NETWORK_NAME` en `.env`). |
| 3 | Hermes Agent: contenedor `hermes-agent` pero URL `hermesagent`. | El nombre DNS oficial es **`hermes-agent`**. Se añade el alias `hermesagent` para que ambas URLs funcionen. |
| 4 | Open WebUI: la URL interna aparece como `:8080` (sección 3) y `:3000` (sección 3.1). | Dentro de la red Docker el puerto es **8080**. El **3000** solo existe en el host. URL interna correcta: `http://openwebui:8080`. |
| 5 | OpenCode: URL interna `:8443`. | El puerto interno es **8080** (8443 es el del host). URL interna: `http://opencode:8080`. |
| 6 | Se pide `docker_<servicio>.yml` (sección 6) y `docker-<servicio>.yml` (sección 5). | Se usa **`docker-<servicio>.yml`** (con guion). |
| 7 | "Todos los contenedores deben usar GPU". | La GPU solo aporta valor en **Ollama, Open WebUI (imagen `cuda`), ComfyUI y YOLO**. SearXNG, Qdrant, OpenCode y Hermes **no ejecutan cálculo en GPU** y no se les reserva. |
| 8 | Hermes Agent, OpenCode: no se indica imagen. | Hermes usa la imagen oficial `nousresearch/hermes-agent`. OpenCode, ComfyUI y YOLO se construyen con un `Dockerfile` propio (incluidos abajo). |
| 9 | Volúmenes con nombre (`ollama_data`...) mapeados a `$HOME/...`. | Se usan **bind mounts** bajo `DATA_ROOT` (por defecto tu `$HOME`), que es lo que describe la especificación. |

### Mapa final de puertos (sin colisiones)

| Servicio | Contenedor | Puerto interno | Puerto host | URL interna (red-ai) |
| :--- | :--- | :--- | :--- | :--- |
| Ollama | `ollama` | 11434 | 11434 | `http://ollama:11434` |
| Open WebUI | `openwebui` | 8080 | 3000 | `http://openwebui:8080` |
| Hermes Agent | `hermes-agent` | 8000 | 8000 | `http://hermes-agent:8000` |
| OpenCode | `opencode` | 8080 | 8443 | `http://opencode:8080` |
| ComfyUI | `comfyui` | 8188 | 8188 | `http://comfyui:8188` |
| YOLO | `yolo` | 5000 | 5000 | `http://yolo:5000` |
| SearXNG | `searxng` | 8080 | 8080 | `http://searxng:8080` |
| RAG (Qdrant) | `rag` | 6333 | 6333 | `http://rag:6333` |

Los puertos del host (11434, 3000, 8000, 8443, 8188, 5000, 8080, 6333) son todos distintos. Que varios contenedores usen 8080 **internamente** no es problema: cada contenedor tiene su propia dirección en la red.

---

## 1. Prerrequisitos e instalación base

### 1.1. Qué necesitas

- Ubuntu Server 24.04 LTS (o 26.04) recién instalado, con acceso `sudo` y conexión a Internet.
- Tarjeta NVIDIA (RTX 3050, 4060 o superior).
- Al menos **100 GB libres** en disco (los modelos de IA ocupan mucho) y **16 GB de RAM** recomendados.

> **Aviso sobre la VRAM:** una 3050 o 4060 tiene 6–8 GB de memoria de vídeo. No caben a la vez un LLM, ComfyUI y YOLO. Este manual configura Ollama para descargar los modelos de la GPU cuando no se usan (sección 4), de modo que los servicios se turnan la GPU.

### 1.2. Actualizar el sistema

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release git ubuntu-drivers-common openssl jq
```

### 1.3. Instalar el driver de NVIDIA

El driver se instala **en el host**. Los contenedores usan el driver del host, por lo que **no hace falta instalar el CUDA Toolkit completo en el host**: las librerías CUDA vienen dentro de las imágenes.

```bash
# Ver qué driver recomienda Ubuntu para tu tarjeta
ubuntu-drivers devices

# Instalar automáticamente el driver recomendado
sudo ubuntu-drivers install

# Reiniciar para cargar el driver
sudo reboot
```

Tras reiniciar, comprueba:

```bash
nvidia-smi
```

Debe mostrar una tabla con el nombre de tu GPU y la versión del driver. Si da error, no sigas: revisa la sección 6.4.

> **Requisito:** driver **550 o superior** (necesario para CUDA 12.4 usado por ComfyUI).

### 1.4. Instalar Docker Engine y Docker Compose (repositorio oficial apt)

```bash
# 1. Clave GPG de Docker
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 2. Repositorio
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 3. Instalación (incluye el plugin "docker compose")
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 4. Usar docker sin sudo (cierra sesión y vuelve a entrar después)
sudo usermod -aG docker $USER
newgrp docker

# 5. Comprobar
docker --version
docker compose version
docker run --rm hello-world
```

### 1.5. Instalar NVIDIA Container Toolkit (GPU dentro de Docker)

Es el componente que permite que los contenedores vean la GPU.

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit

# Registrar el runtime NVIDIA en Docker y reiniciar
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### 1.6. Prueba de GPU dentro de un contenedor

```bash
docker run --rm --gpus all ubuntu nvidia-smi
```

Si ves la misma tabla que en el host, la base está lista.

---

## 2. Estructura del proyecto

Todo el proyecto vive en `$HOME/proyecto`. Los datos persistentes viven en `DATA_ROOT` (por defecto `$HOME`), tal como indica la especificación.

```text
$HOME/proyecto/
├── .env                          # Variables de entorno (secretos: no subir a Git)
├── ai.sh                         # Script de ayuda: arrancar/parar/actualizar todo
├── backup.sh                     # Script de copia de seguridad
├── docker-ollama.yml
├── docker-openwebui.yml
├── docker-hermes-agent.yml
├── docker-opencode.yml
├── docker-comfyui.yml
├── docker-yolo.yml
├── docker-searxng.yml
├── docker-rag.yml
└── build/
    ├── opencode/
    │   └── Dockerfile
    ├── comfyui/
    │   └── Dockerfile
    └── yolo/
        ├── Dockerfile
        └── app.py

$HOME/                            # DATA_ROOT: datos persistentes
├── ollama/                       # Modelos LLM
├── openwebui/                    # Usuarios, chats, prompts, configuración
├── hermes/                       # Configuración de los agentes
├── opencode/
│   ├── config/                   # opencode.json
│   └── workspace/                # Código de los proyectos
├── comfyui/
│   ├── models/                   # checkpoints, loras, vae...
│   ├── input/
│   ├── output/                   # Imágenes y vídeos generados
│   └── user/                     # Workflows y ajustes
├── yolo/                         # Datasets y pesos
├── searxng/                      # settings.yml y caché
└── rag/                          # Base vectorial Qdrant
```

### 2.1. Crear directorios y la red Docker

```bash
# Proyecto
mkdir -p $HOME/proyecto/build/{opencode,comfyui,yolo}

# Datos persistentes
mkdir -p $HOME/{ollama,openwebui,hermes,yolo,searxng,rag}
mkdir -p $HOME/opencode/{config,workspace}
mkdir -p $HOME/comfyui/{input,output,user}
mkdir -p $HOME/comfyui/models/{checkpoints,loras,vae,controlnet,clip,unet,upscale_models,embeddings,diffusion_models,text_encoders}

# Permisos específicos
sudo chown -R 977:977 $HOME/searxng      # SearXNG corre con el usuario 977

# Red principal (bridge) con DNS interno automático
docker network create --driver bridge red-ai
docker network ls | grep red-ai
```

> **Cómo funciona el DNS local:** en una red *bridge definida por el usuario*, Docker resuelve automáticamente el nombre de cada contenedor (`ollama`, `openwebui`...) a su IP. No hay que instalar ningún servidor DNS.

---

## 3. Ficheros de configuración

Cada servicio tiene su propio `docker-<servicio>.yml`, con un `name:` distinto para que se gestionen de forma independiente. Todos leen las variables del fichero `.env` de la sección 4. Crea los ficheros dentro de `$HOME/proyecto`.

### 3.1. `docker-ollama.yml`

```yaml
name: ai-ollama

services:
  ollama:
    image: ollama/ollama:${OLLAMA_TAG:-latest}
    container_name: ollama
    restart: unless-stopped
    ports:
      - "${OLLAMA_PORT:-11434}:11434"
    environment:
      - TZ=${TZ:-UTC}
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_KEEP_ALIVE=${OLLAMA_KEEP_ALIVE:-5m}
      - OLLAMA_MAX_LOADED_MODELS=${OLLAMA_MAX_LOADED_MODELS:-1}
      - OLLAMA_NUM_PARALLEL=${OLLAMA_NUM_PARALLEL:-1}
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - ${DATA_ROOT}/ollama:/root/.ollama
    healthcheck:
      test: ["CMD", "ollama", "list"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 20s
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - red-ai

networks:
  red-ai:
    external: true
    name: ${NETWORK_NAME:-red-ai}
```

### 3.2. `docker-rag.yml` (Qdrant)

```yaml
name: ai-rag

services:
  rag:
    image: qdrant/qdrant:${QDRANT_TAG:-latest}
    container_name: rag
    restart: unless-stopped
    ports:
      - "${RAG_PORT:-6333}:6333"
    environment:
      - TZ=${TZ:-UTC}
    volumes:
      - ${DATA_ROOT}/rag:/qdrant/storage
    networks:
      - red-ai

networks:
  red-ai:
    external: true
    name: ${NETWORK_NAME:-red-ai}
```

### 3.3. `docker-searxng.yml`

```yaml
name: ai-searxng

services:
  searxng:
    image: searxng/searxng:${SEARXNG_TAG:-latest}
    container_name: searxng
    restart: unless-stopped
    ports:
      - "${SEARXNG_PORT:-8080}:8080"
    environment:
      - TZ=${TZ:-UTC}
      - SEARXNG_BASE_URL=http://localhost:${SEARXNG_PORT:-8080}/
    volumes:
      - ${DATA_ROOT}/searxng:/etc/searxng
    networks:
      - red-ai

networks:
  red-ai:
    external: true
    name: ${NETWORK_NAME:-red-ai}
```

Crea su configuración (imprescindible para que Open WebUI pueda usar formato JSON):

```bash
cat > $HOME/searxng/settings.yml <<EOF
use_default_settings: true
server:
  secret_key: "$(openssl rand -hex 32)"
  limiter: false
  image_proxy: true
search:
  formats:
    - html
    - json
EOF
sudo chown 977:977 $HOME/searxng/settings.yml
```

### 3.4. `docker-comfyui.yml`

```yaml
name: ai-comfyui

services:
  comfyui:
    build:
      context: ./build/comfyui
    image: local/comfyui:latest
    container_name: comfyui
    restart: unless-stopped
    ports:
      - "${COMFYUI_PORT:-8188}:8188"
    environment:
      - TZ=${TZ:-UTC}
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
      - CLI_ARGS=${COMFYUI_CLI_ARGS:---lowvram}
    volumes:
      - ${DATA_ROOT}/comfyui/models:/opt/ComfyUI/models
      - ${DATA_ROOT}/comfyui/input:/opt/ComfyUI/input
      - ${DATA_ROOT}/comfyui/output:/opt/ComfyUI/output
      - ${DATA_ROOT}/comfyui/user:/opt/ComfyUI/user
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - red-ai

networks:
  red-ai:
    external: true
    name: ${NETWORK_NAME:-red-ai}
```

`build/comfyui/Dockerfile`:

```dockerfile
FROM pytorch/pytorch:2.5.1-cuda12.4-cudnn9-runtime

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
        git libgl1 libglib2.0-0 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /opt
RUN git clone --depth 1 https://github.com/comfyanonymous/ComfyUI.git
WORKDIR /opt/ComfyUI
RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 8188
CMD ["sh", "-c", "python main.py --listen 0.0.0.0 --port 8188 ${CLI_ARGS}"]
```

> Si una versión muy reciente de ComfyUI exigiera un PyTorch más nuevo, cambia la primera línea del `Dockerfile` por una imagen `pytorch/pytorch` más reciente y reconstruye (`--build`).

### 3.5. `docker-yolo.yml`

```yaml
name: ai-yolo

services:
  yolo:
    build:
      context: ./build/yolo
    image: local/yolo-api:latest
    container_name: yolo
    restart: unless-stopped
    ports:
      - "${YOLO_PORT:-5000}:5000"
    environment:
      - TZ=${TZ:-UTC}
      - YOLO_MODEL=${YOLO_MODEL:-yolo11n.pt}
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - ${DATA_ROOT}/yolo:/data
    ipc: host
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - red-ai

networks:
  red-ai:
    external: true
    name: ${NETWORK_NAME:-red-ai}
```

`build/yolo/Dockerfile`:

```dockerfile
FROM ultralytics/ultralytics:latest

RUN pip install --no-cache-dir fastapi "uvicorn[standard]" python-multipart
COPY app.py /app/app.py

# Los pesos del modelo se descargan en /data (volumen persistente)
WORKDIR /data
EXPOSE 5000
CMD ["uvicorn", "app:app", "--app-dir", "/app", "--host", "0.0.0.0", "--port", "5000"]
```

`build/yolo/app.py`:

```python
import io
import os

import torch
from fastapi import FastAPI, File, UploadFile
from PIL import Image
from ultralytics import YOLO

app = FastAPI(title="YOLO API")
model = YOLO(os.getenv("YOLO_MODEL", "yolo11n.pt"))
DEVICE = 0 if torch.cuda.is_available() else "cpu"


@app.get("/health")
def health():
    return {"status": "ok", "device": "cuda" if DEVICE == 0 else "cpu"}


@app.post("/predict")
async def predict(file: UploadFile = File(...), conf: float = 0.25):
    img = Image.open(io.BytesIO(await file.read())).convert("RGB")
    result = model.predict(img, conf=conf, device=DEVICE, verbose=False)[0]
    detections = [
        {
            "class": result.names[int(box.cls)],
            "confidence": round(float(box.conf), 4),
            "bbox_xyxy": [round(v, 1) for v in box.xyxy[0].tolist()],
        }
        for box in result.boxes
    ]
    return {"count": len(detections), "detections": detections}
```

### 3.6. `docker-hermes-agent.yml`

```yaml
name: ai-hermes-agent

services:
  hermes-agent:
    image: nousresearch/hermes-agent:${HERMES_TAG:-latest}
    container_name: hermes-agent
    restart: unless-stopped
    command: gateway run
    ports:
      - "${HERMES_PORT:-8000}:8000"
    environment:
      - TZ=${TZ:-UTC}
      - API_SERVER_ENABLED=true
      - API_SERVER_HOST=0.0.0.0
      - API_SERVER_PORT=8000
      - API_SERVER_KEY=${HERMES_API_KEY}
    volumes:
      - ${DATA_ROOT}/hermes:/opt/data
    networks:
      red-ai:
        aliases:
          - hermesagent

networks:
  red-ai:
    external: true
    name: ${NETWORK_NAME:-red-ai}
```

> **Importante:** el directorio `/opt/data` y el comando `gateway run` están documentados por Nous Research. Los nombres exactos de las variables `API_SERVER_*` (servidor compatible con OpenAI) conviene contrastarlos con la documentación oficial de la versión que instales (`https://hermes-agent.nousresearch.com/docs`), porque este proyecto evoluciona rápido. Se configura una sola vez con el asistente (sección 7.4).

### 3.7. `docker-opencode.yml`

```yaml
name: ai-opencode

services:
  opencode:
    build:
      context: ./build/opencode
    image: local/opencode:latest
    container_name: opencode
    restart: unless-stopped
    ports:
      - "${OPENCODE_PORT:-8443}:8080"
    environment:
      - TZ=${TZ:-UTC}
      - OPENCODE_SERVER_PASSWORD=${OPENCODE_SERVER_PASSWORD}
    volumes:
      - ${DATA_ROOT}/opencode/config:/root/.config/opencode
      - ${DATA_ROOT}/opencode/workspace:/workspace
    working_dir: /workspace
    networks:
      - red-ai

networks:
  red-ai:
    external: true
    name: ${NETWORK_NAME:-red-ai}
```

`build/opencode/Dockerfile`:

```dockerfile
FROM node:22-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
        git curl ripgrep ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN npm install -g opencode-ai

WORKDIR /workspace
EXPOSE 8080
CMD ["opencode", "web", "--hostname", "0.0.0.0", "--port", "8080"]
```

`$HOME/opencode/config/opencode.json` (conecta OpenCode con Ollama y, opcionalmente, con SearXNG):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (local)",
      "options": {
        "baseURL": "http://ollama:11434/v1"
      },
      "models": {
        "qwen2.5-coder:7b": {
          "name": "Qwen2.5 Coder 7B"
        }
      }
    }
  },
  "mcp": {
    "searxng": {
      "type": "local",
      "command": ["npx", "-y", "mcp-searxng"],
      "environment": {
        "SEARXNG_URL": "http://searxng:8080"
      }
    }
  }
}
```

### 3.8. `docker-openwebui.yml`

```yaml
name: ai-openwebui

services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:${OPENWEBUI_TAG:-cuda}
    container_name: openwebui
    restart: unless-stopped
    ports:
      - "${OPENWEBUI_PORT:-3000}:8080"
    environment:
      - TZ=${TZ:-UTC}
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY}
      - WEBUI_AUTH=${WEBUI_AUTH:-true}
      - ENABLE_SIGNUP=${ENABLE_SIGNUP:-true}
      # --- Ollama ---
      - OLLAMA_BASE_URL=http://ollama:11434
      # --- RAG: base vectorial Qdrant + embeddings con Ollama ---
      - VECTOR_DB=qdrant
      - QDRANT_URI=http://rag:6333
      - RAG_EMBEDDING_ENGINE=ollama
      - RAG_OLLAMA_BASE_URL=http://ollama:11434
      - RAG_EMBEDDING_MODEL=${RAG_EMBEDDING_MODEL:-nomic-embed-text}
      # --- Búsqueda web con SearXNG ---
      - ENABLE_WEB_SEARCH=true
      - WEB_SEARCH_ENGINE=searxng
      - SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>&format=json
      # --- Generación de imágenes con ComfyUI ---
      - ENABLE_IMAGE_GENERATION=true
      - IMAGE_GENERATION_ENGINE=comfyui
      - COMFYUI_BASE_URL=http://comfyui:8188
      - NVIDIA_VISIBLE_DEVICES=all
    volumes:
      - ${DATA_ROOT}/openwebui:/app/backend/data
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - red-ai

networks:
  red-ai:
    external: true
    name: ${NETWORK_NAME:-red-ai}
```

> Open WebUI sigue cambiando los nombres de algunas variables entre versiones. Todo lo anterior también se puede configurar desde la interfaz (**Panel de administración → Ajustes**), que tiene prioridad la primera vez. Si una variable no surte efecto, usa la interfaz (sección 7).

### 3.9. Validar la sintaxis antes de arrancar

```bash
cd $HOME/proyecto
for f in docker-*.yml; do
  echo "== $f"; docker compose -f "$f" config -q && echo "OK"
done
```

Cada fichero debe imprimir `OK` (criterio de aceptación 1).

---

## 4. Fichero de entorno `.env`

Crea `$HOME/proyecto/.env`. **Sustituye `DATA_ROOT` por el resultado de `echo $HOME`** (ruta absoluta, p. ej. `/home/jason`).

```bash
cat > $HOME/proyecto/.env <<EOF
# ===== General =====
TZ=Europe/Madrid
DATA_ROOT=$HOME
NETWORK_NAME=red-ai

# ===== Puertos del host (todos distintos) =====
OLLAMA_PORT=11434
OPENWEBUI_PORT=3000
HERMES_PORT=8000
OPENCODE_PORT=8443
COMFYUI_PORT=8188
YOLO_PORT=5000
SEARXNG_PORT=8080
RAG_PORT=6333

# ===== Versiones de imagen =====
OLLAMA_TAG=latest
OPENWEBUI_TAG=cuda
HERMES_TAG=latest
QDRANT_TAG=latest
SEARXNG_TAG=latest

# ===== Ollama (ajustado a GPUs de 6-8 GB) =====
OLLAMA_KEEP_ALIVE=5m
OLLAMA_MAX_LOADED_MODELS=1
OLLAMA_NUM_PARALLEL=1

# ===== Open WebUI =====
WEBUI_SECRET_KEY=$(openssl rand -hex 32)
WEBUI_AUTH=true
ENABLE_SIGNUP=true
RAG_EMBEDDING_MODEL=nomic-embed-text

# ===== Hermes Agent =====
HERMES_API_KEY=$(openssl rand -hex 24)

# ===== OpenCode =====
OPENCODE_SERVER_PASSWORD=$(openssl rand -base64 18 | tr -d '/+=')

# ===== ComfyUI =====
COMFYUI_CLI_ARGS=--lowvram

# ===== YOLO =====
YOLO_MODEL=yolo11n.pt
EOF

chmod 600 $HOME/proyecto/.env
```

Los secretos (`WEBUI_SECRET_KEY`, `HERMES_API_KEY`, `OPENCODE_SERVER_PASSWORD`) se generan aleatoriamente. Para ver la contraseña de OpenCode:

```bash
grep OPENCODE_SERVER_PASSWORD $HOME/proyecto/.env
```

> **Seguridad:** `ENABLE_SIGNUP=true` permite crear la primera cuenta (que será administradora). Después, cámbialo a `false` y recrea el contenedor. No expongas estos puertos a Internet sin un proxy inverso con HTTPS.

---

## 5. Despliegue y verificación

### 5.1. Script de ayuda `ai.sh`

Los servicios están en ficheros separados y unos dependen de otros, así que este script los arranca en el orden correcto.

```bash
cat > $HOME/proyecto/ai.sh <<'EOF'
#!/usr/bin/env bash
# Uso: ./ai.sh {up|down|restart|ps|logs|pull} [servicio]
set -euo pipefail
cd "$(dirname "$0")"

# Orden de arranque: primero las bases, al final la interfaz
SERVICES=(ollama rag searxng comfyui yolo hermes-agent opencode openwebui)
ACTION="${1:-}"
ONLY="${2:-}"

list() { [[ -n "$ONLY" ]] && echo "$ONLY" || echo "${SERVICES[@]}"; }

case "$ACTION" in
  up)      for s in $(list); do echo ">> Arrancando $s"; docker compose -f "docker-$s.yml" up -d --build; sleep 2; done ;;
  down)    for s in $(list | tr ' ' '\n' | tac); do echo ">> Parando $s"; docker compose -f "docker-$s.yml" down; done ;;
  restart) for s in $(list); do docker compose -f "docker-$s.yml" restart; done ;;
  ps)      docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' ;;
  logs)    docker compose -f "docker-${ONLY:?indica el servicio}.yml" logs -f --tail=100 ;;
  pull)    for s in $(list); do docker compose -f "docker-$s.yml" pull --ignore-buildable; done ;;
  *) echo "Uso: $0 {up|down|restart|ps|logs|pull} [servicio]"; exit 1 ;;
esac
EOF
chmod +x $HOME/proyecto/ai.sh
```

### 5.2. Arranque

```bash
cd $HOME/proyecto
./ai.sh up
```

La primera vez tarda bastante (descarga imágenes y construye ComfyUI, YOLO y OpenCode). Equivale a ejecutar `docker compose -f docker-<servicio>.yml up -d` para cada servicio.

Para arrancar uno solo:

```bash
docker compose -f docker-ollama.yml up -d
```

### 5.3. Descargar los modelos iniciales

```bash
# Modelo de chat (7-8B en cuantización 4 bits cabe en 6-8 GB)
docker exec -it ollama ollama pull qwen2.5:7b

# Modelo de código para OpenCode
docker exec -it ollama ollama pull qwen2.5-coder:7b

# Modelo de embeddings para RAG
docker exec -it ollama ollama pull nomic-embed-text

docker exec -it ollama ollama list
```

Para ComfyUI, descarga un *checkpoint* (por ejemplo, un modelo de Stable Diffusion con licencia que te convenga) en `$HOME/comfyui/models/checkpoints/`.

### 5.4. Verificar el estado

```bash
./ai.sh ps
```

Todos los contenedores deben aparecer como `Up` (Ollama con `healthy`).

### 5.5. Revisar logs

```bash
./ai.sh logs ollama
./ai.sh logs openwebui
docker logs --tail 50 comfyui
```

Sal del modo seguimiento con `Ctrl + C`.

### 5.6. Verificar la GPU

```bash
# En el host
nvidia-smi

# Dentro de los contenedores
docker exec ollama nvidia-smi
docker exec comfyui nvidia-smi
docker exec yolo nvidia-smi

# PyTorch ve la GPU
docker exec comfyui python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"

# Ollama usando la GPU (columna PROCESSOR debe decir "100% GPU")
docker exec ollama ollama run qwen2.5:7b "Hola" && docker exec ollama ollama ps

# Monitorizar mientras trabajas
watch -n 1 nvidia-smi
```

### 5.7. Pruebas de cada servicio

| Servicio | Prueba | Resultado esperado |
| :--- | :--- | :--- |
| Ollama | `curl http://localhost:11434/api/tags` | JSON con los modelos |
| Open WebUI | Navegador: `http://IP_SERVIDOR:3000` | Pantalla de registro/login |
| SearXNG | `curl "http://localhost:8080/search?q=docker&format=json" \| jq '.results[0].title'` | Un título de resultado |
| RAG (Qdrant) | `curl http://localhost:6333/collections` | JSON con `"status":"ok"` |
| ComfyUI | Navegador: `http://IP_SERVIDOR:8188` | Editor de nodos |
| YOLO | `curl http://localhost:5000/health` | `{"status":"ok","device":"cuda"}` |
| Hermes Agent | `docker logs hermes-agent` y `curl -i http://localhost:8000/health` | Sin errores en el log; respuesta HTTP |
| OpenCode | Navegador: `http://IP_SERVIDOR:8443` | Pide usuario/contraseña (usuario `opencode`, contraseña `OPENCODE_SERVER_PASSWORD`) |

Prueba de YOLO con una imagen:

```bash
curl -F "file=@foto.jpg" http://localhost:5000/predict | jq
```

### 5.8. Verificar la comunicación interna (DNS)

```bash
docker exec openwebui curl -s http://ollama:11434/api/tags | head -c 200
docker exec openwebui curl -s http://searxng:8080/healthz
docker exec openwebui curl -s http://rag:6333/collections
docker exec openwebui curl -s http://comfyui:8188/system_stats | head -c 200
docker exec opencode curl -s http://ollama:11434/api/tags | head -c 200
```

Si una de estas falla, ese contenedor no está en `red-ai` o no está arrancado (sección 6.4).

---

## 6. Mantenimiento y actualización

### 6.1. Copias de seguridad

Lo que importa son los datos de `$HOME/{ollama,openwebui,hermes,opencode,comfyui,yolo,searxng,rag}`. Los modelos de Ollama y ComfyUI son muy grandes y se pueden volver a descargar, así que se excluyen por defecto.

```bash
cat > $HOME/proyecto/backup.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
DEST="${1:-$HOME/backups}"
FECHA=$(date +%F_%H%M)
mkdir -p "$DEST"

cd "$HOME/proyecto" && ./ai.sh down openwebui   # parar solo lo imprescindible
# Qdrant y OpenWebUI usan bases de datos: mejor con el servicio parado
docker stop rag 2>/dev/null || true

tar czf "$DEST/ai-datos_$FECHA.tar.gz" \
  --exclude="$HOME/ollama" \
  --exclude="$HOME/comfyui/models" \
  -C "$HOME" openwebui hermes opencode comfyui/user comfyui/output yolo searxng rag \
  -C "$HOME/proyecto" .env docker-*.yml build ai.sh backup.sh

docker start rag 2>/dev/null || true
cd "$HOME/proyecto" && ./ai.sh up openwebui

# Conservar solo las últimas 7 copias
ls -1t "$DEST"/ai-datos_*.tar.gz | tail -n +8 | xargs -r rm --
echo "Backup creado en $DEST/ai-datos_$FECHA.tar.gz"
EOF
chmod +x $HOME/proyecto/backup.sh
```

Uso y programación diaria a las 03:00:

```bash
./backup.sh                       # manual
crontab -e                        # añade esta línea:
0 3 * * * /home/TU_USUARIO/proyecto/backup.sh >> /home/TU_USUARIO/backup.log 2>&1
```

Restaurar:

```bash
./ai.sh down
tar xzf ~/backups/ai-datos_FECHA.tar.gz -C $HOME   # restaura los datos
./ai.sh up
```

> Prueba la restauración al menos una vez: una copia que nunca se ha restaurado no es una copia fiable.

### 6.2. Actualización de servicios

```bash
cd $HOME/proyecto
./backup.sh                       # 1) siempre copia antes

./ai.sh pull                      # 2) descarga imágenes nuevas
./ai.sh up                        # 3) recrea los contenedores cambiados

# Imágenes construidas localmente (ComfyUI, YOLO, OpenCode): reconstruir sin caché
docker compose -f docker-comfyui.yml build --no-cache
docker compose -f docker-comfyui.yml up -d

# Actualizar un modelo de Ollama
docker exec ollama ollama pull qwen2.5:7b

# Limpiar imágenes antiguas
docker image prune -f
```

Si una actualización falla, fija la versión anterior en `.env` (por ejemplo `OPENWEBUI_TAG=0.6.0`) y ejecuta `./ai.sh up openwebui`.

### 6.3. Comandos de operación diaria

```bash
./ai.sh ps                        # estado
./ai.sh restart ollama            # reiniciar un servicio
docker stats                      # CPU/RAM por contenedor
docker system df                  # espacio usado por Docker
df -h $HOME                       # espacio en disco
```

### 6.4. Resolución de errores comunes

| Síntoma | Causa probable | Solución |
| :--- | :--- | :--- |
| `nvidia-smi` falla en el host: *"couldn't communicate with the NVIDIA driver"* | Driver no cargado o Secure Boot | `sudo reboot`; si persiste, desactiva Secure Boot en la BIOS o registra la clave MOK al instalar el driver. |
| `docker: Error response from daemon: could not select device driver "nvidia"` | Falta NVIDIA Container Toolkit | Repite la sección 1.5 y `sudo systemctl restart docker`. |
| Ollama responde pero va lento (`ollama ps` marca CPU) | No ve la GPU o el modelo no cabe en VRAM | Revisa `docker exec ollama nvidia-smi`; usa un modelo más pequeño o con menor cuantización. |
| `CUDA out of memory` en ComfyUI/YOLO | Otro servicio ocupa la VRAM | `docker exec ollama ollama stop qwen2.5:7b` o baja `OLLAMA_KEEP_ALIVE` (p. ej. `30s`). Mantén `--lowvram` en ComfyUI. |
| `permission denied` al escribir en un volumen | El usuario del contenedor no es dueño de la carpeta | Averigua el UID (`docker exec <c> id`) y aplica `sudo chown -R UID:UID $HOME/<carpeta>`. SearXNG: `sudo chown -R 977:977 $HOME/searxng`. |
| `port is already allocated` / `address already in use` | Otro programa usa el puerto | `sudo ss -tulpn \| grep :8080`; cambia el puerto en `.env` y relanza. |
| `network red-ai declared as external, but could not be found` | No se creó la red | `docker network create red-ai`. |
| Open WebUI no muestra modelos | No alcanza a Ollama | `docker exec openwebui curl -s http://ollama:11434/api/tags`; comprueba que ambos están en `red-ai`. |
| SearXNG devuelve error 403 al pedir `format=json` | JSON no habilitado | Verifica `search.formats` en `$HOME/searxng/settings.yml` y `./ai.sh restart searxng`. |
| Un contenedor se reinicia en bucle | Error de arranque | `docker logs --tail 100 <contenedor>` y lee el último error. |
| Disco lleno | Modelos e imágenes | `docker system prune -a` (**cuidado**: borra imágenes sin uso), `ollama rm <modelo>`. |
| Cambio en `.env` sin efecto | Los contenedores guardan el entorno al crearse | `docker compose -f docker-<servicio>.yml up -d --force-recreate`. |

---

## 7. Guía de integración entre servicios

Dentro de `red-ai` los contenedores se llaman por su **nombre** y su **puerto interno**. Nunca uses `localhost` desde un contenedor para llegar a otro: `localhost` es el propio contenedor.

### 7.1. Open WebUI ↔ Ollama

Ya está configurado con `OLLAMA_BASE_URL=http://ollama:11434`. Verificación manual en la interfaz:

1. Entra en `http://IP_SERVIDOR:3000` y crea la cuenta (la primera es administradora).
2. **Panel de administración → Ajustes → Conexiones**. En *Ollama API* debe aparecer `http://ollama:11434`. Pulsa el icono de verificar.
3. Selecciona un modelo en el chat y escribe un mensaje.

### 7.2. Open WebUI ↔ SearXNG (búsqueda web)

1. Verifica que SearXNG devuelve JSON (sección 5.7).
2. **Panel de administración → Ajustes → Búsqueda web**: activa *Habilitar búsqueda web*, motor `searxng` y URL de consulta `http://searxng:8080/search?q=<query>&format=json`.
3. En un chat, activa el botón de búsqueda web (icono del globo) y pregunta algo reciente.

### 7.3. Open WebUI ↔ RAG (Qdrant + embeddings de Ollama)

1. Confirma que existe el modelo de embeddings: `docker exec ollama ollama list | grep nomic`.
2. **Panel de administración → Ajustes → Documentos**: motor de embeddings `Ollama`, URL `http://ollama:11434`, modelo `nomic-embed-text`.
3. Sube documentos en **Espacio de trabajo → Conocimiento** y úsalos en el chat escribiendo `#` y el nombre de la colección.
4. Comprueba que se guardaron vectores: `curl http://localhost:6333/collections`.

Los ficheros fuente que "enseñan" a la IA se guardan en `$HOME/openwebui`; los vectores, en `$HOME/rag`.

### 7.4. Hermes Agent ↔ Ollama (y SearXNG / ComfyUI)

Hermes se configura una vez con su asistente interactivo, que escribe en `$HOME/hermes`:

```bash
cd $HOME/proyecto
docker compose -f docker-hermes-agent.yml run --rm -it hermes-agent setup
```

En el asistente elige un proveedor **personalizado / compatible con OpenAI** e indica:

- URL base: `http://ollama:11434/v1`
- Clave de API: cualquier texto (Ollama no la comprueba), por ejemplo `ollama`
- Modelo: `qwen2.5:7b` (un modelo con soporte de *tool calling*)

Reinicia y comprueba:

```bash
./ai.sh restart hermes-agent
docker logs --tail 50 hermes-agent
```

Para la **búsqueda web**, Hermes puede usar el servidor MCP `mcp-searxng` con `SEARXNG_URL=http://searxng:8080`, igual que OpenCode (sección 7.5). Para **imágenes**, apunta a `http://comfyui:8188`. El formato exacto de estas dos opciones depende de la versión de Hermes: consúltalo en su documentación oficial.

### 7.5. OpenCode ↔ Ollama y SearXNG

El fichero `$HOME/opencode/config/opencode.json` (sección 3.7) ya define ambos:

- **Ollama:** proveedor `ollama` con `baseURL: http://ollama:11434/v1`. Cada modelo que quieras usar debe estar declarado en `models` y descargado en Ollama.
- **SearXNG:** servidor MCP `searxng` con `SEARXNG_URL=http://searxng:8080`.

Pasos:

```bash
docker exec -it ollama ollama pull qwen2.5-coder:7b
./ai.sh restart opencode
```

Abre `http://IP_SERVIDOR:8443`, inicia sesión (usuario `opencode`) y selecciona el modelo *Qwen2.5 Coder 7B*. Tus proyectos se guardan en `$HOME/opencode/workspace`.

> Si no quieres búsqueda web en OpenCode, elimina el bloque `"mcp"` del JSON (la especificación admite OpenCode con "Ollama o ninguno").

### 7.6. Open WebUI ↔ ComfyUI (generación de imágenes)

1. Descarga un checkpoint en `$HOME/comfyui/models/checkpoints/` y comprueba que ComfyUI lo ve (`http://IP_SERVIDOR:8188`).
2. En ComfyUI, ajusta un workflow y exporta con **Save (API Format)** (activa *modo desarrollador* en los ajustes si no aparece).
3. En Open WebUI: **Panel de administración → Ajustes → Imágenes**: motor `ComfyUI`, URL `http://comfyui:8188`, pega el JSON del workflow y asigna los nodos (prompt, modelo, semilla...).
4. En un chat, pulsa el icono de imagen bajo una respuesta para generarla.

Las imágenes se guardan en `$HOME/comfyui/output`.

### 7.7. YOLO desde otros servicios

YOLO expone una API REST: `POST http://yolo:5000/predict` con un campo `file` (imagen). Cualquier contenedor de la red puede llamarla:

```bash
docker exec openwebui curl -s -F "file=@/tmp/foto.jpg" http://yolo:5000/predict
```

Para usarla desde Open WebUI/Hermes hay que crear una *herramienta* (Open WebUI: **Espacio de trabajo → Herramientas**) o una *skill* que haga esta llamada HTTP. No es una integración automática.

### 7.8. Resumen de conexiones

```text
                    ┌────────────┐
   navegador ─────► │ openwebui  │───────────┬───────────┬──────────┐
                    └─────┬──────┘           │           │          │
                          │            ┌─────▼────┐ ┌────▼────┐ ┌───▼───┐
                          │            │ searxng  │ │ comfyui │ │  rag  │
                    ┌─────▼──────┐     └────▲─────┘ └─────────┘ │Qdrant │
                    │   ollama   │◄─────────┼───────────────────┴───────┘
                    └─────▲──────┘          │ (embeddings: ollama)
                          │                 │
             ┌────────────┴───┐      ┌──────┴──────┐        ┌────────┐
             │  hermes-agent  │      │  opencode   │        │  yolo  │ (API independiente)
             └────────────────┘      └─────────────┘        └────────┘
```

---

## 8. Verificación de los criterios de aceptación

| Criterio | Cómo comprobarlo |
| :--- | :--- |
| 1. Ficheros `docker-<servicio>.yml` funcionales y sin errores | `for f in docker-*.yml; do docker compose -f $f config -q && echo "$f OK"; done` |
| 2. Contenedores con NVIDIA/CUDA, usando GPU | `nvidia-smi` en el host y `docker exec <ollama\|comfyui\|yolo\|openwebui> nvidia-smi`; ver nota 7 de la sección 0 |
| 3. Sin colisiones de puertos | `docker ps --format '{{.Names}}: {{.Ports}}'` y `sudo ss -tulpn \| grep LISTEN`; cada puerto del host aparece una sola vez |
| 4. Explicaciones paso a paso para un novato | Seguir las secciones 1 a 5 en orden, sin saltarse la sección 0 |

### Orden recomendado de lectura para el primer despliegue

1. Sección 0 (decisiones) → 2. Sección 1 (base) → 3. Sección 2 (estructura) → 4. Secciones 3 y 4 (ficheros) → 5. Sección 5 (arranque) → 6. Sección 7 (integraciones) → 7. Sección 6 (backups, programarlos desde el primer día).

---

*Fin del manual. Versión 1.0.*

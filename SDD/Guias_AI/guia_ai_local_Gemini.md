# Manual Técnico de Instalación, Configuración y Operación: Stack de IA Local sobre Ubuntu Server con Docker y GPU NVIDIA

---
**Documento:** Manual Técnico de Requisitos e Infraestructura  
**Basado en:** Especificación SDD (Spec Driven Development) v1.0  
**Autor:** Jason  
**Rol:** Administrador de Sistemas / DevOps  
**Fecha:** 2026-09-22  
**Sistema Operativo Objetivo:** Ubuntu Server 24.04 LTS / 26.04 LTS  
---

## 1. Visión General del Proyecto

Este manual proporciona las instrucciones paso a paso para desplegar un entorno de Inteligencia Artificial autosuficiente, privado y desacoplado en la nube pública, utilizando contenedores Docker acelerados por GPU NVIDIA sobre Ubuntu Server.

### Resumen de Arquitectura de Servicios y Puertos

| Servicio | Nombre de Contenedor | Puerto Interno | Puerto Host | Propósito Principal | Dependencia Directa |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Ollama** | `ollama` | 11434 | 11434 | Servidor de modelos de lenguaje (LLM) y embeddings | Driver NVIDIA CUDA |
| **Open WebUI** | `openwebui` | 8080 | 3000 | Interfaz gráfica interactiva y gestión RAG/Prompts | Ollama, SearXNG, ComfyUI |
| **Hermes Agent**| `hermes-agent` | 8000 | 8000 | Agente autónomo para ejecución de tareas complejas | Ollama, SearXNG, ComfyUI |
| **OpenCode** | `opencode` | 8080 | 8443 | Entorno IDE Web para desarrollo asistido por IA | Ollama |
| **ComfyUI** | `comfyui` | 8188 | 8188 | Motor y flujo de trabajo para generación de imagen/video | Driver NVIDIA CUDA |
| **YOLO** | `yolo` | 5000 | 5000 | API de visión artificial y detección de objetos | Driver NVIDIA CUDA |
| **SearXNG** | `searxng` | 8080 | 8080 | Metabuscador privado para recuperación de información web | Ninguna |
| **RAG System** | `rag` | N/A | N/A | Módulo de base de datos vectorial integrado | Ollama, Open WebUI |

---

## 2. Prerrequisitos e Instalación Base

### 2.1 Actualización del Sistema Base y Herramientas Esenciales

Actualice los repositorios del sistema e instale las herramientas necesarias:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git build-essential ca-certificates gnupg lsb-release
```

### 2.2 Instalación de Drivers NVIDIA y CUDA

Para permitir que los contenedores hagan uso de tarjetas NVIDIA (RTX 3050, 4060, etc.), instale los controladores oficiales y el toolkit de CUDA:

```bash
# 1. Agregar repositorio de drivers gráficos de NVIDIA
sudo add-apt-repository ppa:graphics-drivers/ppa -y
sudo apt update

# 2. Instalar el driver recomendado de NVIDIA y el toolkit CUDA
sudo apt install -y nvidia-driver-550 nvidia-cuda-toolkit

# 3. Reiniciar el sistema para aplicar los cambios del módulo de kernel
sudo reboot
```

Una vez reiniciado el servidor, verifique la correcta comunicación con la tarjeta gráfica:

```bash
nvidia-smi
```

*Nota: La salida debe mostrar el modelo de GPU, la versión del driver instalado y la versión de CUDA soportada.*

### 2.3 Instalación del Engine de Docker y Docker Compose

Instale la última versión oficial de Docker Engine utilizando el repositorio oficial de Docker:

```bash
# Crear directorio para las claves del repositorio
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Configurar el repositorio oficial de Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker Engine, CLI y Plugin de Docker Compose
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Asignar permisos al usuario actual para operar Docker sin sudo
sudo usermod -aG docker $USER
newgrp docker
```

### 2.4 Instalación de NVIDIA Container Toolkit

Este kit de herramientas es indispensable para que los contenedores Docker puedan acceder de forma nativa a la GPU.

```bash
# Configurar el repositorio del NVIDIA Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb [^ ]* #&[signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] #' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Instalar el runtime de NVIDIA
sudo apt update
sudo apt install -y nvidia-container-toolkit

# Configurar Docker para utilizar el runtime de NVIDIA por defecto
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

#### Verificación de GPU dentro de Docker:
```bash
docker run --rm --gpus all ubuntu nvidia-smi
```

---

## 3. Estructura del Proyecto

Toda la configuración y persistencia de datos residirá en la carpeta principal `$HOME/proyecto`.

Ejecute los siguientes comandos para estructurar el proyecto:

```bash
mkdir -p $HOME/proyecto
mkdir -p $HOME/ollama
mkdir -p $HOME/openwebui
mkdir -p $HOME/hermes
mkdir -p $HOME/opencode
mkdir -p $HOME/comfyui
mkdir -p $HOME/yolo
mkdir -p $HOME/searxng
mkdir -p $HOME/rag

cd $HOME/proyecto
```

El árbol final de directorios quedará de la siguiente forma:

```text
$HOME/proyecto/
├── .env
├── docker-ollama.yml
├── docker-openwebui.yml
├── docker-hermes-agent.yml
├── docker-opencode.yml
├── docker-comfyui.yml
├── docker-yolo.yml
├── docker-searxng.yml
└── docker-rag.yml

Directorios de Persistencia en $HOME:
├── $HOME/ollama/        (Modelos LLM de Ollama)
├── $HOME/openwebui/     (Base de datos, chats, prompts)
├── $HOME/hermes/        (Memorias y configuraciones de Hermes)
├── $HOME/opencode/      (Archivos y workspace del IDE)
├── $HOME/comfyui/       (Modelos Checkpoints, VAE, outputs)
├── $HOME/yolo/          (Modelos .pt y datasets de visión)
├── $HOME/searxng/       (Ficheros de configuración de búsquedas)
└── $HOME/rag/           (Documentos e índices vectoriales)
```

---

## 4. Fichero de Entorno Unificado (`.env`)

Cree el archivo `$HOME/proyecto/.env` para centralizar la configuración de variables de entorno de todos los servicios.

```bash
cat << 'EOF' > $HOME/proyecto/.env
# Global Docker Settings
DOCKER_NETWORK=red-ai
TIMEZONE=Europe/Madrid

# Rutas de Persistencia Local ($HOME)
PATH_OLLAMA=/home/${USER}/ollama
PATH_OPENWEBUI=/home/${USER}/openwebui
PATH_HERMES=/home/${USER}/hermes
PATH_OPENCODE=/home/${USER}/opencode
PATH_COMFYUI=/home/${USER}/comfyui
PATH_YOLO=/home/${USER}/yolo
PATH_SEARXNG=/home/${USER}/searxng
PATH_RAG=/home/${USER}/rag

# Configuración de Ollama
OLLAMA_PORT=11434
OLLAMA_BASE_URL=http://ollama:11434

# Configuración de Open WebUI
OPENWEBUI_PORT=3000
WEBUI_SECRET_KEY=ClaveSuperSecretaIA2026ChangeMe

# Configuración de Hermes Agent
HERMES_PORT=8000

# Configuración de OpenCode IDE
OPENCODE_PORT=8443
PASSWORD_OPENCODE=admin1234

# Configuración de ComfyUI
COMFYUI_PORT=8188

# Configuración de YOLO API
YOLO_PORT=5000

# Configuración de SearXNG
SEARXNG_PORT=8080
SEARXNG_SECRET_KEY=GenerarClaveAleatoriaSearxng2026

EOF
```

---

## 5. Ficheros de Configuración `docker-<servicio>.yml`

Cree de manera independiente cada uno de los siguientes manifiestos Docker Compose dentro de `$HOME/proyecto/`.

### 5.1 `docker-ollama.yml`
```yaml
version: '3.8'

networks:
  red-ai:
    external: true

services:
  ollama:
    container_name: ollama
    image: ollama/ollama:latest
    restart: unless-stopped
    ports:
      - "${OLLAMA_PORT}:11434"
    volumes:
      - ${PATH_OLLAMA}:/root/.ollama
    networks:
      - red-ai
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

### 5.2 `docker-openwebui.yml`
```yaml
version: '3.8'

networks:
  red-ai:
    external: true

services:
  openwebui:
    container_name: openwebui
    image: ghcr.io/open-webui/open-webui:main
    restart: unless-stopped
    ports:
      - "${OPENWEBUI_PORT}:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_SECRET_KEY=${WEBUI_SECRET_KEY}
      - ENABLE_RAG_WEB_SEARCH=True
      - RAG_WEB_SEARCH_ENGINE=searxng
      - SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>
    volumes:
      - ${PATH_OPENWEBUI}:/app/backend/data
    networks:
      - red-ai
    depends_on:
      - ollama
```

### 5.3 `docker-hermes-agent.yml`
```yaml
version: '3.8'

networks:
  red-ai:
    external: true

services:
  hermes-agent:
    container_name: hermes-agent
    image: nousresearch/hermes-agent:latest
    restart: unless-stopped
    ports:
      - "${HERMES_PORT}:8000"
    environment:
      - OLLAMA_HOST=http://ollama:11434
      - SEARXNG_HOST=http://searxng:8080
      - COMFYUI_HOST=http://comfyui:8188
    volumes:
      - ${PATH_HERMES}:/app/data
    networks:
      - red-ai
    depends_on:
      - ollama
```

### 5.4 `docker-opencode.yml`
```yaml
version: '3.8'

networks:
  red-ai:
    external: true

services:
  opencode:
    container_name: opencode
    image: codercom/code-server:latest
    restart: unless-stopped
    ports:
      - "${OPENCODE_PORT}:8080"
    environment:
      - PASSWORD=${PASSWORD_OPENCODE}
      - OLLAMA_HOST=http://ollama:11434
    volumes:
      - ${PATH_OPENCODE}:/home/coder/project
    networks:
      - red-ai
```

### 5.5 `docker-comfyui.yml`
```yaml
version: '3.8'

networks:
  red-ai:
    external: true

services:
  comfyui:
    container_name: comfyui
    image: yanweiliu/comfyui:latest
    restart: unless-stopped
    ports:
      - "${COMFYUI_PORT}:8188"
    volumes:
      - ${PATH_COMFYUI}:/app/data
    networks:
      - red-ai
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

### 5.6 `docker-yolo.yml`
```yaml
version: '3.8'

networks:
  red-ai:
    external: true

services:
  yolo:
    container_name: yolo
    image: ultralytics/ultralytics:latest-python
    restart: unless-stopped
    ports:
      - "${YOLO_PORT}:5000"
    volumes:
      - ${PATH_YOLO}:/usr/src/app/data
    command: python3 -m ultralytics.models.yolo.server --host 0.0.0.0 --port 5000
    networks:
      - red-ai
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

### 5.7 `docker-searxng.yml`
```yaml
version: '3.8'

networks:
  red-ai:
    external: true

services:
  searxng:
    container_name: searxng
    image: searxng/searxng:latest
    restart: unless-stopped
    ports:
      - "${SEARXNG_PORT}:8080"
    environment:
      - SEARXNG_BASE_URL=http://searxng:8080/
      - INSTANCE_NAME=Local-AI-SearXNG
    volumes:
      - ${PATH_SEARXNG}:/etc/searxng
    networks:
      - red-ai
```

### 5.8 `docker-rag.yml`
```yaml
version: '3.8'

networks:
  red-ai:
    external: true

services:
  rag:
    container_name: rag
    image: chromadb/chroma:latest
    restart: unless-stopped
    ports:
      - "8000:8000"
    volumes:
      - ${PATH_RAG}:/chroma/chroma
    environment:
      - IS_PERSISTENT=TRUE
      - ALLOW_RESET=TRUE
    networks:
      - red-ai
```

---

## 6. Despliegue y Verificación

### 6.1 Creación de la Red Virtual Docker (`red-ai`)

Antes de lanzar cualquier contenedor, la red en modo `bridge` debe existir de forma explícita:

```bash
docker network create red-ai
```

### 6.2 Comando de Despliegue Secuencial

Despliegue todos los servicios leyendo sus ficheros de configuración específicos y aplicando el fichero `.env`:

```bash
cd $HOME/proyecto

docker compose -f docker-ollama.yml up -d
docker compose -f docker-searxng.yml up -d
docker compose -f docker-rag.yml up -d
docker compose -f docker-comfyui.yml up -d
docker compose -f docker-yolo.yml up -d
docker compose -f docker-openwebui.yml up -d
docker compose -f docker-hermes-agent.yml up -d
docker compose -f docker-opencode.yml up -d
```

### 6.3 Comprobación de Estado y Logs

Verifique que todos los contenedores estén en estado `Up`:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Compruebe los registros de log de un servicio específico si se detecta un fallo:

```bash
docker logs -f ollama
docker logs -f openwebui
```

### 6.4 Verificación del Funcionamiento de la GPU y Servicios API

#### 1. Descarga e inicio de un modelo LLM en Ollama:
```bash
docker exec -it ollama ollama run llama3.2:3b
```
*Si la respuesta responde en la terminal, la aceleración GPU en Ollama está plenamente operativa.*

#### 2. Prueba HTTP de respuesta de las APIs internas:
```bash
# Probar puerto de Ollama
curl http://localhost:11434/api/version

# Probar puerto del Metabuscador SearXNG
curl -I http://localhost:8080

# Probar estado de uso de la GPU por los procesos Docker
nvidia-smi
```

---

## 7. Mantenimiento y Actualización

### 7.1 Copias de Seguridad (Backups)

Para realizar un respaldo de las bases de datos, historiales de chats, prompts y configuraciones, ejecute el siguiente script para empaquetar los volúmenes mapeados:

```bash
#!/bin/bash
BACKUP_DIR="$HOME/backups_ia_$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

echo "Iniciando copia de seguridad de datos de IA..."
tar -czvf $BACKUP_DIR/openwebui_backup.tar.gz $HOME/openwebui
tar -czvf $BACKUP_DIR/hermes_backup.tar.gz $HOME/hermes
tar -czvf $BACKUP_DIR/opencode_backup.tar.gz $HOME/opencode
tar -czvf $BACKUP_DIR/rag_backup.tar.gz $HOME/rag

echo "Copia de seguridad completada en $BACKUP_DIR"
```

### 7.2 Actualización de Servicios

Para actualizar la imagen de cualquier contenedor a la versión más reciente sin perder datos:

```bash
cd $HOME/proyecto
docker compose -f docker-openwebui.yml pull
docker compose -f docker-openwebui.yml up -d --force-recreate
```

### 7.3 Resolución de Problemas Comunes

#### Error 1: Permisos denegados en volúmenes locales
Si los contenedores fallan al escribir en `$HOME/<servicio>`, ajuste los permisos del directorio:

```bash
sudo chown -R $USER:$USER $HOME/openwebui $HOME/comfyui $HOME/searxng $HOME/rag
sudo chmod -R 775 $HOME/openwebui $HOME/comfyui $HOME/searxng $HOME/rag
```

#### Error 2: `could not select device driver "" with capabilities: [[gpu]]`
Este error indica que el NVIDIA Container Toolkit no está bien registrado en la configuración del servicio Docker.

**Solución:**
```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

#### Error 3: Conflicto de puertos o dirección ocupada
Compruebe qué proceso está ocupando el puerto que causa la colisión:

```bash
sudo netstat -tulpn | grep :8080
```
Cambie la variable de puerto correspondiente en el archivo `.env` y vuelva a desplegar el contenedor.

---

## 8. Guía de Integración Interna entre Servicios

Gracias a la red Docker personalizada `red-ai` en modo `bridge`, los contenedores se comunican entre sí mediante resolución DNS de sus propios nombres de contenedor sin necesidad de pasar por las IP públicas o de interfaz física del servidor.

### 8.1 Integración de Ollama con Open WebUI

1. Acceda a la interfaz web de **Open WebUI** en la dirección `http://<IP-DEL-SERVIDOR>:3000`.
2. Vaya a **Admin Panel** -> **Settings** -> **Connections**.
3. En la sección **Ollama API**, configure la URL interna del contenedor:
   `http://ollama:11434`
4. Pulse **Save**. Open WebUI detectará automáticamente todos los modelos descargados en Ollama.

### 8.2 Integración de SearXNG con Open WebUI (Búsqueda Web en Vivo)

1. En **Open WebUI**, acceda a **Admin Panel** -> **Settings** -> **Web Search**.
2. Active la opción **Enable Web Search**.
3. Seleccione el motor **SearXNG**.
4. Configure la URL del motor de búsqueda interno:
   `http://searxng:8080/search?q=<query>`
5. Los modelos LLM de Open WebUI podrán buscar en internet en tiempo real para enriquecer sus respuestas (RAG Web).

### 8.3 Integración de ComfyUI con Open WebUI (Generación de Imágenes)

1. En **Open WebUI**, navegue a **Settings** -> **Images**.
2. En la opción **Image Generation Engine**, active **ComfyUI**.
3. Establezca la dirección del servidor de generación de imágenes:
   `http://comfyui:8188`
4. Indique los parámetros del modelo de difusión presente en `$HOME/comfyui`.

### 8.4 Integración de Hermes Agent con Ollama y SearXNG

Hermes Agent consulta los datos utilizando el archivo `.env` precargado en su contenedor:
* **LLM Engine:** `http://ollama:11434`
* **Search Engine:** `http://searxng:8080`
* **Image Engine:** `http://comfyui:8188`

El agente ejecuta razonamientos de múltiples pasos apoyándose en los modelos alojados en Ollama y en las consultas extrayendo información vía SearXNG.

### 8.5 Integración de OpenCode con Ollama (Asistente de Código)

1. Acceda a **OpenCode** (Code-Server) en `http://<IP-DEL-SERVIDOR>:8443`.
2. Instale la extensión **Continue.dev** o **Codeium** desde la pestaña de Extensiones.
3. Edite la configuración de la extensión para apuntar el proveedor de LLM local a:
   `http://ollama:11434`
4. Seleccione un modelo orientado a programación (ejemplo: `codellama`, `deepseek-coder` o `qwen2.5-coder`).

---
**Fin del Manual Técnico.**
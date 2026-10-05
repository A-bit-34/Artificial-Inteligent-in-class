---
TITLE: "Proyect based on SDD(Spec Driven Devolepment)"
AUTHOR: "Jason"
DATE: 2026-09-22
CATEGORY: "Promting AI"
TAGS: [markdown,ai,promt,local,SDD]
---
# Proyect based on SDD(Spec Driven Devolepment): Deploying a Local AI Stack with Docker on Ubuntu
- **version:** 1.0
- Rol of the creator: ** Administrador de Sistemas
- **Propósito:** Define a tecnical manual of requierment
   Definir un manual tecnico de requisitos y definiendo una arquitectura para la generacion de un manual tecnico detallado con la instalacion, configuracion, test y mantenimiento en formato markdown (.md)

## 1. Vision general del proyecto
El objetico del proyecto es desplegar uns infraestructura de Inteligenci Artificial local utilizando contenedores Docker en un sistenma operativoa Ubuntu Server. cada servucio residira en su propio Docker. El sistema dispone de tarjeta grafica NVIDIA (GPU).

## 2. Servicios, especificaciones y aplicaciones
Los servicios a desplegar son los siguientes:
| Servicio | Nombre de contenedor | Puerto interno | Puerto externo (Host) | Propósito principal | Dependencias |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Ollama** | ollama | 11434 | 11434 | Motor de LLMs locales y servidor de API | GPU NVIDIA y Driver (CUDA) |
| **Open WebUI** | openwebui | 8080 | 3000 | Interfaz web tipo ChatGPT para interactuar con LLMs | Ollama, SearXNG, ComfyUI |
| **Hermes Agent** | hermes-agent | 8000 | 8000 | Arnés para el motor LLMs, Agente autónomo para realizar tareas complejas | Ollama, SearXNG, ComfyUI |
| **OpenCode** | opencode | 8080 | 8443 | Entorno IDE para desarrollo de código (entorno web) similar a Claude  | Ollama o ninguno |
| **ComfyUI** | comfyui | 8188 | 8188 | Interfaz web para la generación, edición y procesamiento de imágenes y vídeo | GPU driver, sus propios LLMs |
| **YOLO** | yolo | 5000 | 5000 | API o herramienta par la visión artifical, detección y reconocimiento de patrones en tiempo real en imágenes | GPU driver, sus propios LLMs |
| **SearXNG** | searxng | 8080 | 8080 | Metabuscador privado para realizar búsquedas en Internet | GPU driver |
| **RAG** | rag | 11434 | 11434 | Sistema para aumentar la capacidad de un modelo de lenguaje LLM con información externa privada | Ollama, documentacion externa | 

## 3. Arquitectura de red y datos
Red principal que se va a llamar "red-ai" a la que van a pertenecer todos los contenedores para poder comunicarse entre si. Red en modo bridge y crear un sistema de naming (DNS) local del modo siguiente:
| Servicio | Nombre de contenedor | URL |
| :--- | :--- | :--- |
| **Ollama** | ollama | http://ollama:11434 |

### 3.1 Redes Docker

### 3.2 Volumenes de datos

## 4. Requisitos de sistemas y hardware

## 5. Instrucciones para generar el manual tecnico

## 6. Criterio de aceptacion

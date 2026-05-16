# CyberSec RAG — Sistema de Inteligencia Táctica para Red Team

> Sistema RAG (Retrieval-Augmented Generation) personal que transforma sesiones de clase, vídeos y material del Máster en Ciberseguridad en una base de conocimiento táctica consultable en lenguaje natural.

**Objetivo:** acelerar el camino hacia el Red Team profesional usando inteligencia artificial para organizar, recuperar y razonar sobre conocimiento técnico ofensivo.

---

## Índice

1. [Arquitectura general](#arquitectura-general)
2. [Estructura de la información](#estructura-de-la-información)
3. [Pipeline de ingesta](#pipeline-de-ingesta)
4. [Almacenamiento y recuperación semántica](#almacenamiento-y-recuperación-semántica)
5. [Módulos operativos](#módulos-operativos)
   - [Chat RAG](#chat-rag--chatbot)
   - [OSINT Lab](#osint-lab--lab)
   - [Framework Tracker](#framework-tracker--framework)
6. [Stack técnico](#stack-técnico)
7. [Estado actual del sistema](#estado-actual-del-sistema)
8. [Roadmap y mejoras futuras](#roadmap-y-mejoras-futuras)
9. [Instalación y uso](#instalación-y-uso)
10. [Ecosistema del proyecto](#ecosistema-del-proyecto)

---

## Arquitectura general

El sistema sigue una arquitectura RAG clásica extendida con tres capas:

```
┌─────────────────────────────────────────────────────────┐
│                     FUENTES DE DATOS                    │
│   Vídeos MP4 · PDFs · Apuntes · Herramientas OSINT     │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                  PIPELINE DE INGESTA                    │
│   ffmpeg → Whisper → chunking → embeddings → ChromaDB  │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│               MOTOR DE RECUPERACIÓN (RAG)               │
│   Query → embedding → búsqueda semántica → contexto    │
│                        ↓                               │
│              Claude API → respuesta                    │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                    INTERFACES                           │
│   /chat (Streamlit) · /lab (OSINT) · /framework        │
└─────────────────────────────────────────────────────────┘
```

---

## Estructura de la información

Todo el conocimiento del sistema está organizado en tres grandes colecciones dentro de ChromaDB:

### 1. Colección de sesiones de clase
Las 22 sesiones del máster se procesan y dividen en fragmentos de texto con sus metadatos:

```json
{
  "id": "sesion_14_chunk_032",
  "text": "El ataque de Pass-the-Hash consiste en...",
  "metadata": {
    "source": "sesion_14_post_explotacion.mp4",
    "timestamp": "00:34:12",
    "topic": "Post-Explotación",
    "category": "Movimiento lateral"
  }
}
```

### 2. Colección OSINT Lab
Las 1.169 herramientas OSINT están catalogadas con su descripción, uso táctico y categoría:

```json
{
  "tool": "Maltego",
  "category": "Reconocimiento · Grafo de relaciones",
  "use_case": "Mapeo de infraestructura y relaciones entre entidades",
  "tags": ["OSINT", "recon", "grafos", "personas", "dominios"]
}
```

### 3. Colección de frameworks
33 categorías de técnicas de ataque estructuradas según frameworks reconocidos (MITRE ATT&CK, Cyber Kill Chain):

```json
{
  "technique": "T1078 - Valid Accounts",
  "phase": "Persistence",
  "framework": "MITRE ATT&CK",
  "description": "Uso de credenciales válidas para mantener acceso...",
  "status": "estudiado"
}
```

---

## Pipeline de ingesta

El proceso completo desde un archivo de vídeo hasta un fragmento consultable:

```
clase.mp4
    │
    ▼ ffmpeg
audio.wav (16kHz, mono)
    │
    ▼ Whisper (OpenAI)
transcripcion.txt (texto completo con timestamps)
    │
    ▼ chunking semántico
[chunk_001, chunk_002, ..., chunk_N]
(fragmentos con solapamiento para preservar contexto)
    │
    ▼ sentence-transformers (embeddings)
[vector_001, vector_002, ..., vector_N]
(vectores de alta dimensionalidad)
    │
    ▼ ChromaDB
colección persistente en disco
```

### Ejecución del pipeline

```bash
# Procesar un vídeo nuevo
python ingestion/process_video.py --input clase.mp4 --output fragments/

# Indexar documentos PDF o texto
python ingestion/process_docs.py --input apuntes/ --collection sesiones

# Ver estadísticas de la colección
python scripts/stats.py
```

---

## Almacenamiento y recuperación semántica

### ChromaDB como base vectorial

ChromaDB almacena cada fragmento como un par `(embedding, metadata)`. Cuando se lanza una consulta:

1. La consulta se convierte a embedding con el mismo modelo usado en la ingesta
2. ChromaDB calcula la similitud coseno entre el embedding de la consulta y todos los fragmentos almacenados
3. Se recuperan los K fragmentos más similares (top-K retrieval)
4. Los fragmentos se inyectan como contexto en el prompt de Claude API
5. Claude genera una respuesta fundamentada en ese contexto

```python
# Flujo simplificado de una consulta
query = "¿Qué técnicas de evasión de EDR son más efectivas en Windows?"

# 1. Embedding de la consulta
query_embedding = embedder.encode(query)

# 2. Recuperación semántica
results = collection.query(
    query_embeddings=[query_embedding],
    n_results=5
)

# 3. Construcción del contexto
context = build_context(results)

# 4. Generación con Claude
response = claude.messages.create(
    model="claude-opus-4-7",
    messages=[{"role": "user", "content": f"{context}\n\n{query}"}]
)
```

### Por qué embeddings y no búsqueda de palabras clave

La búsqueda semántica permite encontrar fragmentos relevantes aunque no contengan exactamente las palabras de la consulta. "¿Cómo escalar privilegios en Linux?" recuperará fragmentos sobre `sudo`, `SUID`, `capabilities` o `cron jobs` aunque ninguno use exactamente esa frase.

---

## Módulos operativos

### Chat RAG · `/chat`

Interfaz principal del sistema. Permite realizar consultas en lenguaje natural sobre todo el material del máster.

**Funcionamiento interno:**
- El usuario escribe una pregunta táctica
- El sistema recupera los fragmentos más relevantes de ChromaDB
- Claude genera una respuesta citando las fuentes del máster
- La respuesta incluye el fragmento de origen (sesión y timestamp)

**Casos de uso:**
```
"¿Cómo se realiza un ataque de Kerberoasting?"
"¿Qué herramientas se vieron en la sesión de OSINT pasivo?"
"Explica el proceso de pivoting en redes segmentadas"
"¿Qué diferencia hay entre un shell reverso y un bind shell?"
```

**Stack:** Python · Streamlit · ChromaDB · Claude API

---

### OSINT Lab · `/lab`

Base de datos interactiva con las 1.169 herramientas OSINT catalogadas durante el máster. Diseñada para consulta rápida en operaciones de reconocimiento.

**Funcionalidades:**
- Búsqueda por nombre de herramienta o caso de uso
- Filtrado por categoría (personas, dominios, IPs, redes sociales, darkweb, etc.)
- Vista detallada con descripción táctica y ejemplos de uso
- Exportación de listas filtradas para reportes

**Categorías principales:**

| Categoría | Herramientas |
|---|---|
| Reconocimiento de personas | ~180 |
| Infraestructura y dominios | ~210 |
| Redes sociales | ~150 |
| Geolocalización | ~90 |
| Darkweb e inteligencia | ~120 |
| Vulnerabilidades y exploits | ~140 |
| Otras categorías | ~279 |

**Stack:** Python · FastAPI · Streamlit

---

### Framework Tracker · `/framework`

Sistema de seguimiento del progreso en frameworks de ciberseguridad ofensiva. Permite mapear qué técnicas del máster han sido estudiadas y cuáles quedan pendientes.

**Frameworks cubiertos:**
- **MITRE ATT&CK** — tácticas y técnicas organizadas por fase del ataque
- **Cyber Kill Chain** — fases desde reconocimiento hasta acción sobre objetivos
- **PTES** (Penetration Testing Execution Standard)

**Funcionalidades:**
- Visualización de las 33 categorías de técnicas de ataque
- Marcado de técnicas como estudiadas / en progreso / pendientes
- Progreso por fase (Recon, Enumeración, Explotación, Post-Explotación)
- Correlación con el material del máster disponible en el Chat RAG

**Stack:** Python · FastAPI · Streamlit

---

## Stack técnico

| Capa | Tecnología | Función |
|---|---|---|
| Transcripción de vídeo | Whisper (OpenAI) + ffmpeg | MP4 → texto con timestamps |
| Embeddings | sentence-transformers | Vectorización semántica |
| Base vectorial | ChromaDB | Almacenamiento y recuperación |
| Modelo de lenguaje | Claude API (Anthropic) | Generación de respuestas |
| Backend | FastAPI | API REST para interfaces |
| Frontend | Streamlit | Interfaces web interactivas |
| Entorno ofensivo | Kali Linux VM | Validación de técnicas |
| Sistema principal | Windows 11 | Desarrollo y ejecución |

---

## Estado actual del sistema

| Métrica | Valor |
|---|---|
| Fragmentos indexados | 4.146 |
| Sesiones de clase procesadas | 22 |
| Herramientas OSINT catalogadas | 1.169 |
| Categorías de ataque | 33 |
| Interfaces operativas | 3 |
| Fases del roadmap completadas | 1 / 5 |

---

## Roadmap y mejoras futuras

### Fases Red Team

| Fase | Contenido | Estado |
|---|---|---|
| 01 · Recon Pasivo & OSINT | OSINT Lab operativo, 1.169 herramientas indexadas | En curso |
| 02 · Recon Activo & Enumeración | Nmap, servicios, enumeración de AD | Planificado |
| 03 · Vulnerabilidades & Explotación | CVEs, Metasploit, exploits custom | Planificado |
| 04 · Post-Explotación & Lateral Movement | Mimikatz, BloodHound, pivoting | Planificado |
| 05 · Red Team AI Assistant autónomo | Agente con integración C2 | Visión a largo plazo |

---

### Mejoras técnicas planificadas

#### Red neuronal de correlación entre módulos
El mayor cuello de botella actual es que los tres módulos (Chat, OSINT Lab, Framework Tracker) operan de forma independiente. La mejora planificada es una capa de correlación neuronal que relacione automáticamente:
- Una técnica del Framework Tracker con las herramientas OSINT relevantes del Lab
- Un fragmento del Chat con las técnicas MITRE que cubre
- Las herramientas del Lab con los fragmentos de clase donde se explican

Esto convertiría el sistema en una red de conocimiento interconectada en lugar de tres bases de datos separadas.

#### Integración con Obsidian
Exportar el conocimiento del sistema RAG a una vault de Obsidian para aprovechar su sistema de grafos y backlinks. Cada fragmento indexado generaría una nota con enlaces automáticos a técnicas, herramientas y sesiones relacionadas, creando un mapa visual del conocimiento.

#### OSINT automatizado con Apollo.io
Integración con Apollo.io para automatizar la obtención de información pública de LinkedIn de personas objetivo:
- Extracción de perfil profesional, empresa, contactos
- Correlación con Google Dorks personalizados basados en los datos obtenidos
- Búsqueda de imagen de perfil en Yandex Image Search para identificar presencia en otras plataformas
- Todo integrado en el pipeline del OSINT Lab como un módulo de reconocimiento de personas

---

## Instalación y uso

### Requisitos previos

```bash
pip install chromadb anthropic openai streamlit fastapi uvicorn \
            sentence-transformers whisper ffmpeg-python PyMuPDF
```

### Variables de entorno

```bash
ANTHROPIC_API_KEY=tu_clave_api
```

### Lanzar el sistema completo

```bash
# Backend FastAPI
uvicorn interfaces.api:app --reload

# Chat RAG (interfaz web)
streamlit run interfaces/web/app.py

# OSINT Lab
streamlit run interfaces/osint_lab/app.py

# Framework Tracker
streamlit run interfaces/framework_tracker/app.py
```

### Consulta directa vía CLI

```bash
python rag_engine/query.py "¿Cómo se usa BloodHound para enumerar Active Directory?"
```

### Procesar nueva sesión de clase

```bash
python ingestion/process_video.py --input sesion_nueva.mp4 --output fragments/
```

---

## Ecosistema del proyecto

| Plataforma | Enlace |
|---|---|
| Portfolio | [alexc-projects.github.io/…](https://alexc-projects.github.io/Proyecto-Master-Ciberseguridad-Evolve-AlejandroCongil) |
| GitHub | [Alexc-projects/Proyecto-Master-Ciberseguridad-Evolve-AlejandroCongil](https://github.com/Alexc-projects/Proyecto-Master-Ciberseguridad-Evolve-AlejandroCongil) |
| Dev.to | [Cómo construí un sistema RAG para convertirme en Red Teamer con IA](https://dev.to/evolve-space/como-construi-un-sistema-rag-para-convertirme-en-red-teamer-con-ia-proyecto-en-evolve-4b2m) |
| LinkedIn | [De las Telecomunicaciones al Red Team](https://www.linkedin.com/pulse/de-las-telecomunicaciones-al-red-team-c%C3%B3mo-uso-ia-mi-congil-sainz-z6h4e/) |
| Medium | [De técnico de redes a Red Teamer: la IA como ventaja competitiva](https://medium.com/@alejandro.congil5/de-t%C3%A9cnico-de-redes-a-red-teamer-la-ia-como-ventaja-competitiva-en-ciberseguridad-68975e56dd7d) |

---

Proyecto académico desarrollado durante el **Máster en Ciberseguridad de [Evolve](https://evolve.es)**.

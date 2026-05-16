# CyberSec RAG — Sistema de Inteligencia Táctica para Red Team

Sistema RAG (Retrieval-Augmented Generation) personal que indexa sesiones de clase, vídeos y ejercicios del Máster en Ciberseguridad para construir una base de conocimiento táctica consultable en lenguaje natural.

**Objetivo:** acelerar el camino hacia el Red Team profesional usando inteligencia artificial para organizar, recuperar y razonar sobre conocimiento técnico ofensivo.

---

## Tecnologías y herramientas

| Capa | Stack |
|---|---|
| Ingesta de vídeo | Python · Whisper (OpenAI) · ffmpeg |
| Base de conocimiento | ChromaDB · sentence-transformers |
| Modelo de lenguaje | Claude API (Anthropic) |
| Interfaces | FastAPI · Streamlit · CLI |
| OSINT Lab | Python · 1169+ herramientas indexadas |
| Entorno | Windows 11 · Kali Linux VM |

---

## Estructura del repositorio

```
portfolio/
├── index.html          # Portfolio personal — página principal
├── project.html        # Detalle del proyecto CyberSec RAG
└── img/                # Assets visuales
```

El sistema RAG completo incluye los siguientes módulos operativos:

```
CyberSec-RAG/
├── ingestion/          # Pipeline: vídeo → audio → texto → chunks → embeddings
├── rag_engine/         # Motor de búsqueda semántica (ChromaDB + Claude API)
├── interfaces/
│   ├── web/            # Interfaz Streamlit (chat RAG)
│   ├── osint_lab/      # Base de datos de herramientas OSINT
│   └── framework_tracker/  # Seguimiento de frameworks de ciberseguridad
└── scripts/            # Utilidades de mantenimiento y actualización
```

---

## Estado actual del sistema

| Métrica | Valor |
|---|---|
| Fragmentos indexados | 4.146 |
| Sesiones procesadas | 22 |
| Herramientas OSINT catalogadas | 1.169 |
| Categorías de ataque | 33 |

---

## Cómo ejecutarlo

### Requisitos previos
```bash
pip install chromadb anthropic openai streamlit fastapi whisper ffmpeg-python
```

### Variables de entorno
```bash
ANTHROPIC_API_KEY=tu_clave_api
```

### Lanzar interfaz web
```bash
streamlit run interfaces/web/app.py
```

### Consulta vía CLI
```bash
python rag_engine/query.py "¿Cómo se usa Nmap para enumeración de servicios?"
```

### Procesar nuevo vídeo
```bash
python ingestion/process_video.py --input clase.mp4 --output fragments/
```

---

## Resultados principales

- **Pipeline completo funcional**: de vídeo MP4 a fragmento consultable en lenguaje natural
- **Recuperación semántica precisa**: el sistema identifica el fragmento relevante entre 4.146 con embeddings de alta dimensionalidad
- **3 interfaces operativas**: chat web, OSINT Lab y Framework Tracker
- **Cobertura Red Team**: 33 categorías que abarcan fases de Recon, Enumeración, Explotación y Post-Explotación

---

## Roadmap — Fases Red Team

| Fase | Descripción | Estado |
|---|---|---|
| 01 | Recon Pasivo & OSINT | En curso |
| 02 | Recon Activo & Enumeración | Planificado |
| 03 | Análisis de Vulnerabilidades & Explotación | Planificado |
| 04 | Post-Explotación & Movimiento Lateral | Planificado |
| 05 | Red Team AI Assistant autónomo | Visión a largo plazo |

---

## Ecosistema — anillo de navegación

Todos los nodos del proyecto están interconectados:

| Plataforma | Enlace |
|---|---|
| Portfolio | [alexc-projects.github.io/…](https://alexc-projects.github.io/Proyecto-Master-Ciberseguridad-Evolve-AlejandroCongil) |
| GitHub | [Alexc-projects/Proyecto-Master-Ciberseguridad-Evolve-AlejandroCongil](https://github.com/Alexc-projects/Proyecto-Master-Ciberseguridad-Evolve-AlejandroCongil) |
| Dev.to | [Cómo construí un sistema RAG para convertirme en Red Teamer con IA](https://dev.to/evolve-space/como-construi-un-sistema-rag-para-convertirme-en-red-teamer-con-ia-proyecto-en-evolve-4b2m) |
| LinkedIn | [De las Telecomunicaciones al Red Team](https://www.linkedin.com/pulse/de-las-telecomunicaciones-al-red-team-c%C3%B3mo-uso-ia-mi-congil-sainz-z6h4e/) |
| Medium | [De técnico de redes a Red Teamer: la IA como ventaja competitiva](https://medium.com/@alejandro.congil5/de-t%C3%A9cnico-de-redes-a-red-teamer-la-ia-como-ventaja-competitiva-en-ciberseguridad-68975e56dd7d) |

---

Proyecto académico desarrollado durante el **Máster en Ciberseguridad de [Evolve](https://evolve.es)**.

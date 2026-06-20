# AGENTS.md — EisenFlow: Eisenhower Matrix Task Orchestrator

Este documento describe la arquitectura de agentes de inteligencia artificial presentes en el ecosistema de **EisenFlow**, el rol de cada uno de ellos y las instrucciones para que cualquier agente de codificación pueda contribuir a este proyecto de forma correcta y coherente.

---

## 🤖 Visión General de los Agentes

EisenFlow integra dos tipos de agentes que trabajan de forma coordinada:

| Agente | Tecnología | Rol Principal |
| :--- | :--- | :--- |
| **Gemini 2.5 Flash (Redacción)** | Google Gemini API (nativo en n8n) | Redacción automatizada de correos de delegación en Q3 |
| **n8n MCP Server** | Docker + n8n MCP oficial | Interacción programática con flujos de n8n para desarrollo/validación |

---

## 1. Agente de Redacción IA (Cuadrante Q3)

### Descripción
Este es el agente de IA principal del flujo de producción. Se activa automáticamente cuando el webhook recibe una tarea clasificada en el cuadrante **Q3 (Urgente, No Importante)**. Su única responsabilidad es **redactar un correo formal de delegación** basándose en el título de la tarea recibida.

### Configuración en n8n
- **Nodo Activo:** `Message a model` (tipo: `@n8n/n8n-nodes-langchain.googleGemini`)
- **Modelo:** `models/gemini-2.5-flash`
- **Credencial:** `Google Gemini(PaLM) Api account`
- **Temperatura / Tokens Máximos:** Configurados con la opción por defecto del nodo.

### Prompt del Sistema
```text
Redacta un correo formal de delegación de la siguiente tarea: "{{ $json.titulo }}".
El correo debe ser profesional, conciso y solicitar al receptor que se haga cargo de la misma
debido a que es urgente pero no requiere mi atención directa.
Devuelve únicamente el cuerpo del correo sin saludos genéricos de plantilla ni firmas vacías.
```

### Flujo de Ejecución
```
Route by Quadrant [Q3] → Message a model (Gemini) → Gmail (Q3) → Evaluation1 (output)
```

### Tolerancia a Fallos
- **Reintentos:** 5 intentos automáticos (`retryOnFail: true`)
- **Espera entre reintentos:** 5000 ms
- **Acción en fallo definitivo:** `continueRegularOutput` (no bloquea el flujo global)

---

## 2. Agente de Evaluaciones IA (AI Evaluation Loop)

### Descripción
Este agente es exclusivo del entorno de **pruebas y control de calidad**. Utiliza el framework nativo de evaluaciones de n8n para ejecutar casos de prueba del cuadrante Q3 de forma aislada, medir la calidad de los correos generados por Gemini y escribir los resultados de vuelta en el dataset.

### Dataset de Prueba
- **Archivo:** [`tests/eisenflow_evaluation_dataset.csv`](tests/eisenflow_evaluation_dataset.csv)
- **Hoja de cálculo:** Google Sheets `EisenFlow - Evaluation Dataset` / pestaña `Q3 - Gemini Tests`
- **Columnas:**
  - `id` — Identificador del caso de prueba
  - `titulo` — Tarea de prueba enviada al modelo
  - `cuadrante` — Siempre `Q3`
  - `expected_keywords` — Palabras clave esperadas en la respuesta
  - `generated_email` — *(Salida)* Respuesta generada por Gemini

### Nodos del Ciclo de Evaluación
- **`When fetching a dataset row`**: Lee fila por fila del Google Sheet y activa el pipeline.
- **`Evaluation1`**: Captura la salida del modelo y la escribe en la columna `generated_email`.

### Cómo Ejecutar las Evaluaciones
1. Importa el CSV en Google Sheets como hoja `Q3 - Gemini Tests`.
2. En n8n, ve a la pestaña **Evaluations** del canvas del workflow principal.
3. Haz clic en **Run** para ejecutar todas las filas del dataset de forma secuencial.

---

## 3. Agente de Desarrollo (n8n MCP Server)

### Descripción
El servidor **n8n MCP** permite a agentes de codificación externos (como Antigravity IDE) interactuar directamente con tu instancia de n8n. Esto facilita tareas de desarrollo como crear, actualizar, validar y auditar flujos de trabajo sin necesidad de hacerlo manualmente desde la interfaz web.

### Tecnología
- **Imagen Docker:** `ghcr.io/czlonkowski/n8n-mcp:latest`
- **Protocolo:** `stdio` (Model Context Protocol)
- **URL de conexión:** `http://host.docker.internal:5678` (acceso al n8n local desde Docker)

### Configuración en `mcp_config.json`
```json
{
  "mcpServers": {
    "n8n-mcp": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm", "--init",
        "-e", "MCP_MODE=stdio",
        "-e", "LOG_LEVEL=error",
        "-e", "DISABLE_CONSOLE_OUTPUT=true",
        "-e", "N8N_API_URL=http://host.docker.internal:5678",
        "-e", "N8N_API_KEY=TU_API_KEY_AQUI",
        "-e", "WEBHOOK_SECURITY_MODE=moderate",
        "ghcr.io/czlonkowski/n8n-mcp:latest"
      ]
    }
  }
}
```

> [!CAUTION]
> Nunca incluyas tu `N8N_API_KEY` real en archivos commiteados al repositorio. Usa variables de entorno del sistema o tu archivo `.env` local que está cubierto por el `.gitignore`.

### Capacidades Habilitadas
| Herramienta MCP | Descripción |
| :--- | :--- |
| `n8n_list_workflows` | Lista todos los flujos de trabajo activos en la instancia |
| `n8n_get_workflow` | Obtiene la definición JSON de un flujo por ID |
| `n8n_create_workflow` | Crea un nuevo flujo desde una definición JSON |
| `n8n_update_full_workflow` | Reemplaza un flujo completo con una nueva definición |
| `n8n_validate_workflow` | Valida la estructura y conexiones de un flujo |
| `n8n_autofix_workflow` | Detecta y repara errores comunes en un flujo |
| `n8n_test_workflow` | Ejecuta un flujo en modo test con datos de prueba |
| `n8n_health_check` | Verifica el estado de la instancia de n8n |
| `search_nodes` | Busca nodos disponibles en n8n por nombre o categoría |
| `validate_node` | Valida la configuración de un nodo específico |

### Skills de Desarrollo Locales (`.gemini/skills/`)
El workspace incluye skills que guían al agente de codificación en buenas prácticas al trabajar con n8n:

| Skill | Función |
| :--- | :--- |
| `n8n-code-javascript` | Reglas y patrones para nodos de código en JavaScript |
| `n8n-code-python` | Restricciones y patrones para nodos de código en Python |
| `n8n-expression-syntax` | Uso correcto de expresiones `{{ }}` con Luxon y variables |
| `n8n-node-configuration` | Dependencias entre nodos y configuraciones correctas |
| `n8n-validation-expert` | Catálogo de errores comunes y cómo depurarlos |
| `n8n-workflow-patterns` | Patrones arquitectónicos reutilizables (Webhook, API, IA) |
| `n8n-mcp-tools-expert` | Guía de uso avanzado de las herramientas del servidor MCP |

---

## 📐 Diagrama de Interacción de Agentes

```
┌─────────────────────────────────────────────────┐
│               ENTORNO DE PRODUCCIÓN             │
│                                                  │
│  Webhook → Q3 → [ Gemini 2.5 Flash ] → Gmail    │
│                      ↑                           │
│              Redacción automática                │
│              de correos de delegación            │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│              ENTORNO DE EVALUACIONES             │
│                                                  │
│  CSV Dataset → Trigger → [ Gemini ] → Sheet     │
│                             ↑                    │
│                   Pruebas unitarias de IA        │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│              ENTORNO DE DESARROLLO               │
│                                                  │
│  IDE (Antigravity) → [ n8n MCP Server ]         │
│       ↓                    ↓                     │
│  Skills locales      API de n8n (localhost)      │
│  (.gemini/skills/)   Crear/Validar/Auditar       │
└─────────────────────────────────────────────────┘
```

---

## 📄 Referencias
- [Guía Técnica del Proyecto](docs/technical_guide.md)
- [Guía de Instalación de n8n Skills y MCP](docs/agents_guide.md)
- [n8n MCP Server (Docker)](https://github.com/czlonkowski/n8n-mcp)
- [Google Gemini API en n8n](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.googlegemini/)

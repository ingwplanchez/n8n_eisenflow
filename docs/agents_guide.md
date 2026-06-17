# Guía de Instalación: n8n Skills & n8n MCP Server (Self-Hosted en Docker)

Esta guía detalla el proceso paso a paso realizado para configurar el servidor de contexto MCP de n8n y sus correspondientes skills de agente en este workspace.

---

## Parte 1: Instalación de n8n Skills (Locales de Workspace)

Las skills son guías de comportamiento optimizadas que enseñan a los agentes cómo estructurar, validar y construir flujos de trabajo en n8n de manera experta.

### Pasos seguidos:
1. **Clonación temporal**: Se clonó el repositorio oficial de las skills:
   ```bash
   git clone https://github.com/czlonkowski/n8n-skills.git temp-skills
   ```
2. **Creación del directorio local**: Se creó la estructura de carpetas oculta dentro del workspace:
   ```powershell
   New-Item -ItemType Directory -Force -Path .gemini/skills
   ```
3. **Migración de recursos**: Se copiaron recursivamente los contenidos del directorio `skills/` del repositorio clonado hacia `.gemini/skills/` en la raíz de este workspace:
   ```powershell
   Copy-Item -Path .\temp-skills\skills\* -Destination .\.gemini\skills\ -Recurse -Force
   ```
4. **Limpieza**: Se eliminó de manera segura el directorio temporal `temp-skills`:
   ```powershell
   Remove-Item -Path .\temp-skills -Recurse -Force
   ```

### Skills instaladas:
* **n8n-code-javascript**: Buenas prácticas para nodos de código en JS.
* **n8n-code-python**: Reglas y restricciones para nodos de código en Python.
* **n8n-expression-syntax**: Uso y reglas de sintaxis de expresiones con `{{ }}`.
* **n8n-mcp-tools-expert**: Manual avanzado de uso de herramientas del MCP.
* **n8n-node-configuration**: Guía de dependencias y configuraciones de nodos.
* **n8n-validation-expert**: Depuración de errores en validaciones del flujo.
* **n8n-workflow-patterns**: Patrones arquitectónicos y casos de uso comunes.

---

## Parte 2: Configuración de n8n MCP en Docker (Self-Hosted)

Para permitir que el agente interactúe con el entorno local de n8n (crear, actualizar, auditar o validar flujos de trabajo), se configuró el servidor MCP oficial en Docker.

### Pasos seguidos:
1. **Descarga de la imagen**: Se descargó la imagen oficial de Docker (optimizada y ligera al no contener dependencias directas de n8n):
   ```bash
   docker pull ghcr.io/czlonkowski/n8n-mcp:latest
   ```
2. **Configuración del cliente MCP**: Se configuró el archivo global de servidores MCP en `C:\Users\USER\.gemini\config\mcp_config.json` agregando las siguientes credenciales y parámetros de red:

```json
{
  "mcpServers": {
    "n8n-mcp": {
      "command": "docker",
      "args": [
        "run",
        "-i",
        "--rm",
        "--init",
        "-e", "MCP_MODE=stdio",
        "-e", "LOG_LEVEL=error",
        "-e", "DISABLE_CONSOLE_OUTPUT=true",
        "-e", "N8N_API_URL=http://host.docker.internal:5678",
        "-e", "N8N_API_KEY=TU_API_KEY_AQUÍ",
        "-e", "WEBHOOK_SECURITY_MODE=moderate",
        "ghcr.io/czlonkowski/n8n-mcp:latest"
      ]
    }
  }
}
```

### Consideraciones clave:
* **`N8N_API_URL`**: Se configuró como `http://host.docker.internal:5678` debido a que n8n se ejecuta localmente y el servidor MCP dentro de su contenedor Docker necesita acceder a la máquina host (donde está n8n).
* **`WEBHOOK_SECURITY_MODE=moderate`**: Obligatorio cuando se usa `localhost` o `host.docker.internal` para evitar que las políticas estrictas de seguridad SSRF del servidor bloqueen la comunicación.

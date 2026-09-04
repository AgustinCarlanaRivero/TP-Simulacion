# TP Final — Simulación (UTN)

Estudio de la eficiencia técnica, operativa y financiera en la infraestructura de una estación de
carga de vehículos eléctricos a través de la simulación de eventos discretos en CABA.

Equipo: Carlana Rivero, Loglen, Millán, Ojeda Cabrera.

El entregable son notebooks de **Google Colab**. Este repositorio no contiene una aplicación que se
ejecute localmente: guarda la propuesta, la documentación del modelo y la base del TP 4.

## Mapa del repositorio

| Archivo | Qué es |
|---|---|
| [Propuesta_TP-FINAL.md](Propuesta_TP-FINAL.md) | Propuesta aprobada. **Fuente de verdad del modelo**: variables, eventos y condiciones |
| [CLAUDE.md](CLAUDE.md) | Modelo a simular, nomenclatura y convenciones del notebook |
| [herramientas.md](herramientas.md) | Método de trabajo: qué herramienta se adopta, cuál se descarta y por qué |
| [TP 4 Simu.ipynb](TP%204%20Simu.ipynb) | Base de código: ajuste de FDPs + motor evento a evento simple |

## Instalación

Solo hace falta para trabajar con Claude Code sobre los notebooks de Colab. Para leer la propuesta o
la documentación no se instala nada.

### Requisitos

- **Python 3.11+** en el PATH (`python --version`).
- **[Claude Code](https://claude.com/claude-code)**.
- **`uv`**, instalado **con pip**:

  ```bash
  pip install uv
  ```

  Tiene que ser el paquete de PyPI, no el instalador standalone de Astral: el `.mcp.json` invoca
  `python -m uv`, que necesita el paquete importable. En Windows, `pip install --user` deja el
  ejecutable `uv.exe` fuera del PATH, y esta forma de invocarlo justamente evita ese problema.

### Puesta en marcha

```bash
git clone https://github.com/AgustinCarlanaRivero/TP-Simulacion.git
cd TP-Simulacion
pip install uv
claude
```

Al iniciar, Claude Code pide aprobar el MCP server del proyecto (`colab-mcp`, oficial de Google,
declarado en [.mcp.json](.mcp.json)). Hay que aceptarlo: es el que edita y ejecuta los notebooks en
Colab. Se verifica con `/mcp`.

El server expone al principio una sola herramienta, `open_colab_browser_connection`; recién al abrir
esa conexión aparecen las de edición del notebook. Conviene abrirla solo en las sesiones que
efectivamente tocan el notebook (ver [herramientas.md](herramientas.md)).

## Datos

El dataset del TP 4 (`EVChargingStationUsage.csv`) vive en Drive, en
`/content/drive/MyDrive/Colab Notebooks/TP 4 Simu/Datos/`, y se monta desde el notebook con
`drive.mount('/content/drive')`. No está versionado en este repositorio.

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
| [Bibliografia/](Bibliografia/) | Copias locales de los papers y el informe que cita el TP, con su índice |

## Instalación

Para leer la propuesta o la documentación no se instala nada. Sí hacen falta dos cosas: `uv` y Claude
Code para trabajar sobre los notebooks de Colab, y `nbstripout` para cualquiera que vaya a commitear un
`.ipynb`.

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
pip install uv nbstripout
python -m nbstripout --install --attributes .gitattributes
claude
```

Al iniciar, Claude Code pide aprobar el MCP server del proyecto (`colab-mcp`, oficial de Google,
declarado en [.mcp.json](.mcp.json)). Hay que aceptarlo: es el que edita y ejecuta los notebooks en
Colab. Se verifica con `/mcp`.

El server expone al principio una sola herramienta, `open_colab_browser_connection`; recién al abrir
esa conexión aparecen las de edición del notebook. Conviene abrirla solo en las sesiones que
efectivamente tocan el notebook (ver [herramientas.md](herramientas.md)).

### Filtro de notebooks (`nbstripout`)

Obligatorio para todo el que vaya a commitear un `.ipynb`, use o no Claude Code:

```bash
pip install nbstripout
python -m nbstripout --install --attributes .gitattributes
```

Limpia los outputs y la metadata de ejecución en el momento del `git add`: el archivo en disco conserva
los gráficos, y lo que se versiona es solo el texto de las celdas. Sin el filtro, `TP 4 Simu.ipynb` son
266 KB de JSON minificado **en una sola línea** y cualquier diff es ilegible; con el filtro, 26 KB en
776 líneas comparables.

**Se instala una vez por clon.** El [.gitattributes](.gitattributes) viaja con el repositorio, pero la
definición del filtro vive en `.git/config`, que no se versiona. Si alguien no corre los dos comandos,
git ignora el atributo **sin avisar** y commitea el notebook entero con los outputs adentro.

## Datos

El dataset del TP 4 (`EVChargingStationUsage.csv`) vive en Drive, en
`/content/drive/MyDrive/Colab Notebooks/TP 4 Simu/Datos/`, y se monta desde el notebook con
`drive.mount('/content/drive')`. No está versionado en este repositorio.

Los datos del TP Final están en [Datos/](Datos/). Es una copia local de la carpeta de Drive
`/content/drive/MyDrive/Colab Notebooks/TP Final Simu/Datos/`, que es de donde los lee la versión de
Colab del notebook. Un archivo nuevo se agrega en los dos lugares.

| Archivo | Contenido | Uso |
|---|---|---|
| `Dataset California.csv` | Sesiones de la red pública de carga de Palo Alto (California, EE. UU.), 2011–2020 | FDPs de `TC` y de las seis `IA` (secciones 3 a 5) |
| `Dataset Escocia.xlsx` | Reportes de fallas de cargadores de la red pública de Escocia, 2021–2022 | FDPs de `TFC` y `TRC` (sección 6) |
| `Total Patentamientos CABA.xlsx` | Patentamientos trimestrales de automotores por tipo en CABA (IDECBA, cuadro AC_AU_AX03, sobre datos de ACARA) | Stock de enchufables de CABA (sección 2) |
| `Distribución Patentamientos CABA.xlsx` | Distribución trimestral de esos patentamientos por tipo de motor (IDECBA, cuadro AC_AU_AX08) | Stock de enchufables de CABA (sección 2) |
| `EV data by country 2026.xlsx` | Stock, ventas y cuota de autos eléctricos por país (IEA, *Global EV Outlook 2026*) | Techo de la cuota de enchufables en las ventas de los mercados maduros (sección 2.3) |
| `Parque Automotor CABA.xlsx` | Parque automotor mensual de CABA y del país, de 2010 a octubre de 2026 (IDECBA, cuadro TPTE_PA, sobre datos de la DNRPA). Incluye todos los automotores, no sólo automóviles; los datos desde mayo de 2026 son provisorios | Techo del parque, rama B1 de [Alternativas_Techo_K.md](Alternativas_Techo_K.md) (sección 2.3) |

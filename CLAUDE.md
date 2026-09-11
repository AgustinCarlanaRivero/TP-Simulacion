# TP Final — Simulación (UTN)

Trabajo Práctico Final de Simulación. Está basado en el trabajo que hicimos para el **TP 4** (simulación evento a evento de una
estación de carga de vehículos eléctricos), pero con un modelo bastante más complejo: red de estaciones con
expansión dinámica, demanda variable por franja horaria y por año, fallas/mantenimiento de cargadores
y análisis económico.

- Propuesta aprobada: [Propuesta_TP-FINAL.md](Propuesta_TP-FINAL.md) — fuente de verdad del modelo
  (variables, eventos, condiciones). Si se corrige el modelo, se corrige ahí.
- Base de código del TP 4: [TP 4 Simu.ipynb](TP%204%20Simu.ipynb) — ajuste de FDPs + motor evento a evento simple.

**Título:** *Estudio de la eficiencia técnica, operativa y financiera en la infraestructura de una
estación de carga de vehículos eléctricos a través de la simulación de eventos discretos en CABA.*

Equipo: Carlana Rivero, Loglen, Millán, Ojeda Cabrera.

## Entregable y forma de trabajo

- El entregable es **uno o varios notebooks** de Google Colab, que se ejecutan con el **MCP de Google
  Colab** (el entorno de referencia es Colab, no local).
  - El server está declarado en [.mcp.json](.mcp.json) (`colab-mcp`, oficial de Google) y se invoca
    como `python -m uv tool run git+https://github.com/googlecolab/colab-mcp`. Único requisito:
    `pip install uv` (el paquete de PyPI, no el instalador standalone) y aprobar el server del
    proyecto al iniciar Claude Code. Se usa `python -m uv` en lugar de `uvx` porque el ejecutable de
    `uv` no queda en el PATH tras `pip install --user`, y `python` sí está: así el `.mcp.json` no
    depende de rutas de una máquina concreta. Ver [README.md](README.md). Si el MCP no está
    disponible, avisar antes de improvisar un workaround.
- Todo el contenido (markdown, comentarios, nombres, prints) va en **español rioplatense**, con tono de
  informe técnico universitario: claro y ordenado, **sin tono de tutorial** ni explicaciones didácticas.
- No inventar datos. Si falta un dato, se adopta un supuesto razonable y se lo deja **explicitado en el
  notebook** (celda markdown corta con la suposición y su justificación).
- La nomenclatura del código debe coincidir con la de la propuesta (siglas en mayúscula, ver tablas de
  abajo). Si se cambia una sigla o su significado, actualizar también la propuesta.

## Modelo a simular

Metodología: **evento a evento**, avance al próximo evento por mínimo de la TEF.

### Complejidades respecto de lo visto en clase

1. **Demanda por franja horaria**: 6 FDPs de intervalo entre arribos (3 franjas × semana / fin de semana).
   Franjas: 08:00–12:59, 13:00–19:59, 20:00–07:59. El cambio de franja **no es un evento**: el `IA` se
   sortea con la FDP vigente al generarlo (un arribo puede caer en otra franja, sobre todo en la
   nocturna) y los acumuladores por franja y el costo de energía se **prorratean** entre las franjas
   que atraviesa cada tramo.
2. **Crecimiento a largo plazo**: curva logística `P(t) = K / (1 + A·e^(−r·t))` ajustada al parque de
   EV/PHEV de CABA. De ahí sale un **factor variable en el tiempo** que multiplica el IA generado
   (achica los IA en la fase de aceleración, los agranda en la de desaceleración).
3. **Múltiples estaciones y cargadores con expansión dinámica**: arranca con 1 estación y 4 cargadores.
   Cada mes ocurre un **Análisis de Expansión** que decide, por conveniencia económica, agregar un
   cargador y/o construir una estación. Topes: `CC_MAX` (cargadores por estación) y `CE_MAX` (estaciones).
   Cada estación tiene **su propio flujo de arribos** (`TPI(i)`, generado con la `IA` de la franja y el
   tipo de día vigentes): no hay un arribo global que después elija estación. Al ocurrir el arribo a la
   estación `i` se evalúa contra `PDCE` si el auto **efectivamente ingresa**; si no ingresa, es demanda
   no capturada (competencia) y no genera cola ni cuenta como arrepentimiento — `CARRUM(i)` sólo cuenta
   autos que ingresaron y se fueron por la cola única de la estación. Así la red capta `PDCE·CE` % de la demanda total, que es
   lo que acota la condición de construcción de estación.
4. **Fallas y mantenimiento**: cada cargador falla según `TFC` —el reloj de la falla corre sólo
   mientras el cargador está en servicio— y queda fuera de servicio según `TRC`. Si la falla ocurre
   con un auto cargando, ese auto **se pierde** (se va a la competencia): se cancela `TPC(i)(j)`,
   baja `CA(i)` además de `CD(i)`, la carga no se factura pero su energía entregada sí computa como
   costo, y el auto va a `CAPF(i)`, no a `CARRUM(i)` ni a `PARR`.
   Mantenimiento preventivo cada `TMP` días por estación, que **repara los cargadores fallados** y
   **recalcula la próxima falla** de cada cargador.
5. **Estructura de costos dinámica**: precio de energía por franja (pico/valle), mantenimiento,
   amortización de infraestructura nueva, e inversión por cargador (`CPN`) y por estación (`CEN`).

### Clasificación de variables (usar estos nombres en el código)

**Datos (FDPs)**

| Sigla | Significado |
|---|---|
| `IAS1` / `IAS2` / `IAS3` | Intervalo entre arribos, día de semana, franjas 08–13 / 13–20 / 20–08 (min) |
| `IAF1` / `IAF2` / `IAF3` | Ídem fin de semana (min) |
| `TC` | Tiempo de carga de un vehículo (min) |
| `TFC` | Tiempo hasta el fallo de un cargador, desde su reparación hasta que vuelve a fallar (min) |
| `TRC` | Tiempo para reparar un cargador (min) |

**Control** — `RC` (recaudación por carga, $/kWh), `TMP` (días entre mantenimientos preventivos),
`PDCE` (% de demanda a capturar por estación, aplicado como probabilidad de ingreso en cada arribo).

**Resultado** — por franja horaria `(i)` y su promedio: `BM`/`BMP` (beneficio mensual), `PTO`/`PTOP`
(% tiempo ocioso), `PEC`/`PECP` (% espera en cola), `PPS`/`PPSP` (permanencia en el sistema),
`PARR`/`PARRP` (% arrepentimiento), `PDC`/`PDCP` (% disponibilidad de cargadores), `PAPF`/`PAPFP`
(% autos perdidos por falla).

**Estado** — `CA(i)` autos en la estación `i` (cargando + en la cola única), `CD(i)` cargadores
disponibles (instalados y no fallados), `CC(i)` cargadores por estación, `CE` cantidad de estaciones.

**TEF** — `TPI(i)` (uno por estación, no uno global), `TPC(i)(j)`, `TPAE`, `TPIC(i)`, `TPCE` (global, sin índice), `TPFC(i)(j)`, `TPRC(i)(j)`, `TPMP(i)`.

**Auxiliares** — `CARRUM(i)` (arrepentidos del último mes), `ECP` (energía cargada promedio),
`CPAACUM` (promedio de autos atendidos por cargador en el último mes), `CAPF(i)` (autos perdidos por
falla, acumulado de toda la corrida), `ICC(i)(j)` (instante de comienzo de la carga en curso). Seguro
surjan más.

**Constantes** — `CC_MAX`, `CE_MAX`, `CCP` (costo por carga promedio), `CPN` (costo puesto nuevo),
`CEN` (costo estación nueva), `TIC` (tiempo de instalación de cargador: 1 semana = 10 080 min), `TCE`
(tiempo de construcción de estación: 6 meses = 259 200 min, con el mes de 30 días como convención).
`TIC` y `TCE` son los tiempos de obra entre la decisión del Análisis de Expansión y el evento que suma
la capacidad; son valores fijos supuestos y pueden ajustarse más adelante. Seguro surjan más.

### Eventos y condiciones

| Evento (no condicionado) | Evento condicionado que dispara | Condición |
|---|---|---|
| Ingreso de auto a estación `(i)` | Carga en cargador `(i)(j)` | `R < PDCE/100 && CA(i) ≤ CD(i)` |
| Carga en cargador `(i)(j)` | Carga en cargador `(i)(j)` | `CA(i) ≥ CD(i)` |
| Análisis de Expansión | Instalación de nuevo cargador `(i)` | `TPIC(i) = HV && CC(i) < CC_MAX && CARRUM(i)·(RC·ECP − CCP) > CPN` |
| Análisis de Expansión | Construcción de nueva estación | `TPCE = HV && CE < CE_MAX && PDCE·CE < 100 && CPAACUM·4·(RC·ECP − CCP) > CEN` |
| Falla de cargador `(i)(j)` | Reparación de cargador `(i)(j)` | `TPRC(i)(j) = HV` |
| Reparación de cargador `(i)(j)` | Carga en cargador `(i)(j)` | `CA(i) ≥ CD(i)` |
| Instalación de nuevo cargador `(i)` | Carga en cargador `(i)(j)` | `CA(i) ≥ CD(i)` |
| Mantenimiento preventivo `(i)` | Carga en cargador `(i)(j)` | `CA(i) ≥ CD(i)` |
| Construcción de nueva estación | — | — |

Los eventos que suman capacidad (Reparación, Instalación y Mantenimiento preventivo) enganchan al
primero de la cola si hay cola: repiten la condición del fin de carga, con `CD(i)` ya actualizado; el
Mantenimiento preventivo puede devolver varios cargadores a servicio de una vez, así que evalúa la
condición una vez por cada cargador que vuelve a servicio. La Instalación agenda además `TPFC(i)(j)`
del cargador nuevo. La Construcción no dispara ninguna carga porque la estación nace
vacía (`CA(i) = 0`), pero agenda los eventos propios de la estación que crea: `TPI(i)`, `TPFC(i)(j)`
de cada uno de sus **4 cargadores** (la cantidad que supone `CPAACUM·4` en su condición) y `TPMP(i)`.

Cadena de falla: **instalación → fallo, fallo → reparación, reparación → fallo**. La Falla no agenda
la falla siguiente (mientras el cargador está fuera de servicio, `TPFC(i)(j) = HV`); la agendan la
Reparación, la Instalación y el Mantenimiento preventivo, que además repara los fallados
(`TPRC(i)(j) = HV` y `CD(i)` queda en `CC(i)`) sin sacar de servicio a ninguno. La Falla tampoco
dispara ninguna carga condicionada aunque se lleve un auto: `CA(i)` y `CD(i)` bajan juntos y el
invariante `autos cargando = min(CA(i), CD(i))` se sostiene solo.

El Análisis de Expansión, como parte de su actualización de estado, **resetea `CARRUM(i)` y
`CPAACUM`**: son acumuladores del último mes y si no vuelven a cero crecen de forma monótona y las
condiciones de expansión se vuelven cada vez más fáciles de cumplir. Al disparar cada expansión agenda
`TPIC(i) = T + TIC` y `TPCE = T + TCE`, que son los tiempos de obra que les dan sentido a las guardas
`TPIC(i) = HV` y `TPCE = HV` ("no hay obra pendiente").

## Qué se reusa del TP 4 y qué cambia

Reusar (está bien resuelto):

- Pipeline de ajuste de FDPs con `fitter`: histograma → `Fitter().fit()` → `summary(method="ks_statistic")`
  → elección justificada mirando `ks_statistic`, `sumsquare_error` y `kl_div` → `obtener_dist()`.
- `rvs_truncado(dist, data)`: muestreo por transformada inversa **truncado** al rango observado
  (`cdf(min)`–`cdf(max)`), para no generar valores fuera del dominio real.
- Estructura general del loop evento a evento y de los acumuladores.

Resultados ya obtenidos en el TP 4 (como referencia, pero pueden ser re-derivados): `TC` → `gumbel_r(loc=84.39, scale=60.19)`
acotada en [0.1, 1375.92] min; `IA` global → `landau(loc=4.17, scale=3.00)` acotada en [0.5, 1538] min;
energía ≈ `0.0859·TC − 1.7595` kWh; `PE = 108.48 $/kWh`. Dataset: `Dataset California.csv` en Drive
(`/content/drive/MyDrive/Colab Notebooks/TP Final Simu/Datos/`).

Cambia (no copiar tal cual):

- **Un solo IA global → 6 IAs por franja/tipo de día**, más el factor logístico de crecimiento. El reloj
  tiene que mapear `T` (minutos desde el inicio) a hora del día y tipo de día.
- **Elección de servidor aleatoria** (`eleccion_server` usa `random.randrange`) → en el TP final **no hay
  elección de estación ni de cargador**: cada estación tiene su propio `TPI(i)` y `PDCE` decide, arribo
  por arribo, si el auto ingresa o se pierde; adentro, la estación atiende con **una única cola FCFS**
  (el que espera toma el primer cargador que se libera). Decidido y justificado en la propuesta, con dos
  supuestos explicitados ahí: cargadores homogéneos e intercambiables, y sin reserva de turno.
- **Arrepentimiento hardcodeado** (`PORCENTAJE_ARREPENTIMIENTO = 0.93`, corte duro en ≥3 autos) →
  parametrizar y justificar.
- **~20 globals sueltos y `calculo_resultados` que solo imprime** → el motor debe **devolver** las
  métricas para poder correr escenarios y compararlos. Encapsular el estado (clase o dataclass).
- **Regresión energía–tiempo**: el TP 4 hace `linregress(np.sort(tiempo), np.sort(energia))`, o sea ordena
  las dos series por separado. Eso no es una regresión sobre los pares reales y **infla el R²** (0.9835).
  Rehacerla sobre los pares `(TC_k, kWh_k)` alineados y reportar el R² verdadero.
- **`CCE`** en el TP 4 se calcula como `SCE[i]*100/CLL`, mezclando fórmula de porcentaje con un costo. En
  el TP final el resultado económico se define desde cero según la propuesta (`BM`, `BMP`).
- **Horizonte**: `TF = 1_000_000` min (~1,9 años) alcanzaba para el TP 4. Acá tiene que cubrir la curva
  logística (varios años) y los análisis de expansión mensuales.

## Paralelización (importante para el tiempo de cómputo)

El modelo es pesado: horizonte de varios años, expansión dinámica y varios escenarios a comparar.
**Paralelizar siempre donde se pueda**, en particular en estos dos puntos:

1. **Ajuste de FDPs**: las 6 `IA` por franja/tipo de día más `TC`, `TFC` y `TRC` son ajustes
   independientes entre sí → correrlos todos a la vez (un worker por serie) en lugar de uno tras otro.
   Verificar además si la versión instalada de `fitter` expone paralelismo interno (`n_jobs`) en `fit()`.
2. **Ejecución de la simulación**: cada corrida con una combinación distinta de variables de control
   (`RC`, `TMP`, `PDCE`) es independiente → lanzar la grilla de combinaciones en paralelo y recolectar
   los resultados, en vez de barrerla secuencialmente. Lo mismo aplica a las réplicas de una misma
   combinación.

Condiciones para que esto funcione:

- El motor tiene que ser **una función pura**: recibe los parámetros, devuelve las métricas, sin estado
  global compartido (ver "Cambia" arriba). Es el requisito que habilita todo lo demás.
- **Semilla por corrida**, derivada de una semilla base + índice de la combinación/réplica, para que el
  resultado no dependa del orden ni de la cantidad de workers y las corridas del informe sigan siendo
  reproducibles.
- En Colab, usar `concurrent.futures.ProcessPoolExecutor` (el trabajo es CPU-bound, los hilos no sirven
  por el GIL) y dimensionar los workers según `os.cpu_count()`, que en el entorno CPU de Colab es 2.
  Definir las funciones a paralelizar a nivel de módulo para que sean serializables.

### Entorno de ejecución en Colab

Usar **CPU** como entorno por defecto (*Entorno de ejecución → Cambiar tipo de entorno de ejecución*).
El motor es Python puro, evento a evento, dominado por ramas y saltos: no hay álgebra lineal densa que
un acelerador pueda aprovechar, y ni `fitter`/`scipy` ni el loop de simulación tocan GPU o TPU. Lo único
que mueve la aguja es la **cantidad de vCPUs** disponibles para los procesos worker.

- **GPU T4**: no aporta nada. Trae la misma cantidad de vCPUs que el entorno CPU (2 en el nivel gratuito),
  el acelerador queda ocioso y encima consume cuota de aceleradores.
- **TPU v5e-1**: la TPU tampoco sirve, pero la VM host sí es grande — el tipo de máquina de v5e de 1 chip
  (`ct5lp-hightpu-1t`) declara **24 vCPU y 48 GB de RAM**, contra 2 vCPU y ~13 GB del entorno CPU. Es el
  único caso en que un entorno con acelerador se justifica acá, y sólo como máquina de 24 núcleos para el
  barrido de escenarios. Contras: consume cuota/unidades de cómputo y Colab pide explícitamente volver al
  entorno estándar cuando no se usa el acelerador. Si se lo usa, verificar `os.cpu_count()` al inicio
  (Colab puede no exponer los 24) y dejar la decisión explicitada en el notebook.
- **High-RAM**: innecesario. El estado de la simulación es chico; lo que crece son métricas acumuladas.

Regla práctica: arrancar en CPU y medir. Si el barrido de combinaciones se vuelve el cuello de botella y
no alcanza con ajustar réplicas u horizonte, recién ahí evaluar v5e-1 por sus vCPUs.

## Convenciones del notebook

- Estructura por secciones numeradas en markdown: contexto → datos y FDPs → funciones auxiliares → motor
  de simulación → experimentación → conclusiones.
- Encabezado estándar de Colab: `drive.mount('/content/drive')`, `pip install fitter`, imports
  (`numpy`, `pandas`, `matplotlib`, `scipy.stats`, `fitter`, `math`, `random`).
- Cada elección de FDP va acompañada de una celda markdown que **justifica** la distribución elegida.
- Semilla fija para las corridas reproducibles del informe; escenarios de sensibilidad variando `RC`,
  `TMP` y `PDCE`.
- Si una sección depende de otra todavía no implementada, dejarla preparada con un bloque explícito y un
  comentario de qué falta — no dejarla vacía ni silenciosamente rota.

## Reglas de trabajo

Las cinco reglas que [herramientas.md](herramientas.md) propone anotar acá. El razonamiento y las
alternativas descartadas están en ese archivo; acá va sólo la regla operativa.

### Canario

Empezar cada respuesta nombrando **en qué sección del TP se está trabajando** (`[Datos y FDPs]`,
`[Motor de eventos]`, `[Validación]`, `[Experimentación]`, `[Documentación]`). Cuando el gesto
desaparece, es señal de que este archivo dejó de pesar en el contexto y conviene abrir sesión nueva.

### Objetivo verificable antes de arrancar

Traducir la tarea a algo chequeable antes de escribir la primera línea. "Ajustar las FDPs de IA" no es
un objetivo; "las seis FDPs elegidas, con su `ks_statistic` reportado y su celda de justificación" sí.
Para tareas de varios pasos, plan corto con la verificación de cada paso.

### Cambios quirúrgicos

Son cuatro personas sobre un notebook, donde un reformateo cosmético cambia el `.ipynb` entero y hace
incomparable el diff.

- No reordenar, renumerar ni reformatear celdas adyacentes al cambio.
- No reescribir la redacción de celdas markdown que escribió otro integrante.
- Si aparece código muerto del TP 4, **mencionarlo — no borrarlo**.
- Limpiar sólo lo que el propio cambio dejó huérfano (imports, variables sin uso).

### Verificación del motor

Un motor evento a evento siempre corre y siempre devuelve números: un bug en la TEF o en las
condiciones de expansión no tira excepción, produce resultados plausibles y equivocados. Por eso la
verificación se ancla en la propuesta, que es la especificación del modelo.

- Los invariantes y los asserts salen de la tabla de eventos y condiciones de
  [Propuesta_TP-FINAL.md](Propuesta_TP-FINAL.md), **nunca de leer el motor ya escrito**. Si la
  propuesta no cubre el caso, eso es un hueco de la propuesta: se frena y se pregunta, no se copia en
  silencio lo que el código ya hacía.
- Prohibido ensanchar la tolerancia hasta que el chequeo pase. Si la comparación contra M/M/c no
  cierra, el sospechoso es el motor, no el margen de error.
- Prohibido elegir la semilla que da el resultado lindo. Se fija una vez y antes de ver los
  resultados; si un escenario sólo funciona con una semilla, no funciona.
- Prohibido bajar la cantidad de réplicas para que el intervalo de confianza deje de contradecir la
  conclusión.
- Ningún chequeo vale por "no tiró excepción": tiene que afirmar sobre un valor o sobre un cambio de
  estado. Un motor que corre entero y devuelve basura es el escenario esperado, no el raro.
- Prohibido silenciar con `try/except` una excepción que aparece durante una corrida para que termine.

### Tiers de modelo al delegar

Al delegar en un subagente, `model` explícito siempre:

- `haiku` — lo mecánico: segmentar el dataset, renombrar variables al esquema de siglas de la
  propuesta, extraer tablas.
- `sonnet` — default.
- `opus` — lo genuinamente difícil: el motor de eventos, la validación contra M/M/c.

Ante la duda entre dos tiers, el más barato y escalar si falla; una tarea que ya se sabe difícil va al
de arriba de entrada.

## Pendientes / decisiones abiertas

- Origen de datos para `TFC` y `TRC` (el dataset del TP 4 no los tiene). La definición nueva de `TFC`
  condiciona qué dato buscar: el tiempo entre la puesta en servicio (reparación) y la falla siguiente
  del mismo cargador, no el tiempo entre fallas consecutivas.
- Serie histórica del parque EV/PHEV en CABA para ajustar la logística, y el % de usuarios que cargan en
  domicilio (se resta de la demanda).
- Segmentación del dataset por franja horaria y tipo de día para obtener las 6 FDPs de IA.
- Valores de `CC_MAX`, `CE_MAX`, `CPN`, `CEN`, `CCP`, tarifas pico/valle y costos de mantenimiento.
- Cantidad de notebooks: uno solo, o separar "análisis de datos y FDPs" de "motor + experimentación".

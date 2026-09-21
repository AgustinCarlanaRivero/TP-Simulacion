# Bibliografía

Copias locales de las fuentes que usa el TP, para que el repo sea autocontenido y las citas no
dependan de que un link siga vivo. Cada entrada indica qué se usa de cada fuente y desde dónde se la
cita.

| Archivo | Qué es | Dónde se usa |
|---|---|---|
| [`Paper_TP_4.pdf`](Paper_TP_4.pdf) | Nuestro paper del TP 4 | Base del TP final; regla de arrepentimiento vigente y resultados de referencia |
| [`IDEAS_2024_Chattopadhyay_Kar_arXiv_2403.06223.pdf`](IDEAS_2024_Chattopadhyay_Kar_arXiv_2403.06223.pdf) | Modelo de impaciencia en estaciones de carga | Núcleo del modelo de arrepentimiento |
| [`EAFO_Consumer_Monitor_2023_EU_Aggregated_Report.pdf`](EAFO_Consumer_Monitor_2023_EU_Aggregated_Report.pdf) | Encuesta europea a conductores de vehículos eléctricos | FDP de paciencia en valor absoluto |
| [`TimeAnxiety_2020_Alsabbagh_Wu_Ma_IEEE_TII.pdf`](TimeAnxiety_2020_Alsabbagh_Wu_Ma_IEEE_TII.pdf) | Gestión de carga con *time anxiety* | Perfiles de conductor y forma de la curva de impaciencia |
| [`sustainability-16-02536.pdf`](sustainability-16-02536.pdf) | Espera aceptable declarada por usuarios de EV en Japón | Respaldo académico de modelar la paciencia como variable aleatoria, y su discretización en tramos |
| [`sustainability-17-00336.pdf`](sustainability-17-00336.pdf) | Cola de estación de carga con *balking* y *reneging* | Forma funcional exponencial del arrepentimiento y su antecedente |
| [`2025-08-29-bchydro-public-ev-charging-service-rates-evaluation-report-year-1.pdf`](2025-08-29-bchydro-public-ev-charging-service-rates-evaluation-report-year-1.pdf) | Informe regulatorio anual de la red pública de carga de BC Hydro | Distribución de paciencia (`TMEU`), más datos operativos y económicos de una red real |

---

## Detalle

### `Paper_TP_4.pdf`

**Estudio de la eficiencia técnica y operativa en la infraestructura de carga de vehículos eléctricos
a través de la simulación de eventos discretos en la Ciudad Autónoma de Buenos Aires.**
Carlana Rivero, Loglen, Millán, Ojeda Cabrera. UTN — FRBA.

- §2.1: regla de arrepentimiento (93 % con un vehículo en el puesto, 100 % con dos o más).
- §2.3: ajuste de las FDP de `IA` (Landau) y `TC` (Gumbel R).
- §2.4: precio de energía y regresión energía–tiempo.
- §3: resultados por escenario, que son el baseline de comparación del TP final.

Citado desde [`../Arrepentimiento_Alternativas.md`](../Arrepentimiento_Alternativas.md) §3 y desde
[`../CLAUDE.md`](../CLAUDE.md).

### `IDEAS_2024_Chattopadhyay_Kar_arXiv_2403.06223.pdf`

**IDEAS: Information-Driven EV Admission in Charging Station Considering User Impatience to Improve
QoS and Station Utilization.** A. Chattopadhyay, S. Kar. Indian Institute of Technology, Delhi.
arXiv:2403.06223v1, 10 de marzo de 2024. Licencia CC BY-SA 4.0.

Es la fuente del 93 % que usa el TP 4 (Tabla II, caso `ObservationFC` con λ = 0,6). De acá salen:

- La separación entre balking forzado, balking voluntario y reneging (§III-A).
- El factor de impaciencia `z = 0,6`: la paciencia como fracción del tiempo de carga propio
  (ec. 5 y 9).
- Los estimadores de espera por perfil de usuario — optimista, estándar, pesimista (ec. 10–12).
- La cola que permite abandonar desde cualquier posición (Algoritmo 1).
- Los cuatro escenarios `BlockingFC`, `ObservationFC`, `InformedFC` e `Informed2PortCharge` (§V) y
  sus resultados (Tabla II).

### `EAFO_Consumer_Monitor_2023_EU_Aggregated_Report.pdf`

**Consumer Monitor 2023 — European Alternative Fuels Observatory, EU Aggregated Report.**
L. Vanhaverbeke, D. Verbist, G. Barrera (VUB-MOBI) y M. Csukas (FIER). Comisión Europea,
DG MOVE, junio de 2024. doi:10.2832/062076. Licencia CC BY 4.0.

- §3.4 y figura 10: espera declarada en puntos de carga públicos, que es la distribución de paciencia
  que usa el modelo (31 % no espera, 32 % hasta 15 min, 18 % hasta 30 min, 13 % hasta 1 h, 6 % más).
- §4: mismos porcentajes abiertos por país, que dan el rango del análisis de sensibilidad.

### `TimeAnxiety_2020_Alsabbagh_Wu_Ma_IEEE_TII.pdf`

**Distributed Electric Vehicles Charging Management Considering Time Anxiety and Customer Behaviors.**
A. Alsabbagh, B. Wu, C. Ma. *IEEE Transactions on Industrial Informatics*, 2020. Manuscrito final de
autor; el uso personal está permitido por IEEE.

- §III: concepto de *time anxiety* y los cuatro perfiles de conductor (NTAD, LTAD, MTAD, HTAD).
- Ecuaciones 11–13: las tres formas funcionales de cómo crece la impaciencia con el tiempo
  transcurrido (logarítmica, lineal y exponencial).

### `sustainability-16-02536.pdf`

**Modeling of the Acceptable Waiting Time for EV Charging in Japan.** U. e Hanni, T. Yamamoto,
T. Nakamura. *Sustainability* 2024, 16, 2536. MDPI, 20 de marzo de 2024. doi:10.3390/su16062536.
Licencia CC BY 4.0.

Encuesta de preferencias declaradas a 441 usuarios de BEV en Japón (relevamiento de noviembre de 2021,
regiones de Chubu y Kanto), modelada con un *generalized ordered logit*.

- §3 y tabla 1: el escenario SP define la espera aceptable como el tiempo hasta que se libera un
  cargador **cuando el usuario llega y está ocupado**, que es exactamente la situación que decide el
  arrepentimiento en nuestro modelo.
- Tabla 4: discretización de la paciencia en cuatro categorías ordinales — sin espera, 5 a <15 min,
  15 a <60 min, 60 a 90 min. Es la referencia para agrupar en tramos una distribución empírica de
  paciencia.
- §4.3 y figura 3: la moda de la espera tolerable es "no esperar nada", seguida de 5 min, en todas las
  ubicaciones y tanto para carga normal como rápida. Coincide en forma con la encuesta de BC Hydro.
- §4.1–4.2 y figuras 1–2: esperas medias y máximas efectivamente experimentadas. Alrededor del 40 % no
  esperó nada en comercios, tiendas de conveniencia y concesionarias, y entre 15 % y 30 % esperó de 5 a
  menos de 30 minutos.
- §5.2 y §6: la tolerancia varía con sexo, edad, ingreso, situación laboral y frecuencia de uso, y
  sobre todo con la **ubicación** del cargador. Es el argumento para tratar la paciencia como variable
  aleatoria y no como una constante.

Citado desde [`../Calibracion_Arrepentimiento_EV.md`](../Calibracion_Arrepentimiento_EV.md) §5 y
[`../Calibracion_Arrepentimiento_Revision.md`](../Calibracion_Arrepentimiento_Revision.md) §5.5.

### `sustainability-17-00336.pdf`

**Should Charging Stations Provide Service for Plug-In Hybrid Electric Vehicles During Holidays?**
T. Zhang, X. Li, Y. Zhang, C. Shu. *Sustainability* 2025, 17, 336. MDPI, 4 de enero de 2025.
doi:10.3390/su17010336. Licencia CC BY 4.0.

Modelo M/M/1 de una estación de carga que atiende EV y PHEV con clientes impacientes, resuelto por
proceso de nacimiento y muerte.

- §4.2: **confirma la forma funcional** que usa nuestra calibración. La probabilidad de ingresar con
  `n` autos en cola es `b_n = e^(−β·n)`, y la tasa media de abandono por unidad de tiempo es
  `b'_n = (1 − e^(−β·n))·(n − 1)`.
- La exponencial **no es original de este paper**: la toman de Gross, Shortle, Thompson y Harris,
  *Fundamentals of Queueing Theory* (Wiley, 2008), su referencia [30]. Ese es el antecedente a citar.
- §4.1: variante más simple, con probabilidad de ingreso constante mientras haya cola e independiente
  de su largo. Es el caso degenerado contra el que conviene contrastar el nuestro.
- §3 y §4.2: función de beneficio `r·λ_efectivo − c·Lq` — ingreso sobre los autos que **efectivamente**
  ingresan, menos un costo por auto y por unidad de tiempo en cola. Referencia de forma para `BM`.
- Tabla 1: relevamiento de qué estructura de cola usa cada trabajo del área (M/M/C, M/M/S/N, M/D/C,
  M/G/N/N, GI/GI/C). Respalda usar M/M/c como referencia de validación del motor.
- La comparación entre cola única y colas separadas del paper es **entre tipos de vehículo** (EV y
  PHEV), que nuestro modelo no distingue: con una sola clase de auto, la cola única FCFS de la
  propuesta es el único caso. No sirve como justificación de esa decisión.

Citado desde [`../Calibracion_Arrepentimiento_EV.md`](../Calibracion_Arrepentimiento_EV.md) §3 y
[`../Calibracion_Arrepentimiento_Revision.md`](../Calibracion_Arrepentimiento_Revision.md) §11.

### `2025-08-29-bchydro-public-ev-charging-service-rates-evaluation-report-year-1.pdf`

**Public Electric Vehicle Charging Service Rates — Evaluation Report for Year One (mayo 2024 a
abril 2025).** British Columbia Hydro and Power Authority, presentado ante la British Columbia
Utilities Commission el 29 de agosto de 2025. Es la versión pública: la información comercialmente
sensible aparece tachada como `xx` en varias tablas.

Informe regulatorio de la red pública de carga de BC Hydro: 591 puertos en 144 sitios, 85 % de carga
rápida, con un pico de 56 000 sesiones mensuales promedio.

#### Paciencia

Apéndice C, §*Willingness to wait to access a BC Hydro charger*. Quinta encuesta anual, noviembre
de 2024, 1 813 respuestas y base de 1 747 usuarios de carga pública (1 814 en 2023). Preguntas 11d
(zona urbana) y 12 (zona no urbana; el enunciado cambió en 2024, así que esa serie no se compara
contra 2023).

| Máximo que está dispuesto a esperar | Urbano 2024 | Urbano 2023 | No urbano 2024 |
|---|---:|---:|---:|
| Nada | 17 % | 18 % | 12 % |
| 5 min | 29 % | 23 % | 17 % |
| 10 min | 30 % | 32 % | 25 % |
| 15 min | 1 % | 2 % | 23 % |
| 20–30 min | 18 % | 20 % | 17 % |
| Más de 30 min | 5 % | 5 % | 6 % |

Los tramos salen del gráfico de barras apiladas, y los agregados que el informe publica en prosa
cierran exactamente con esa lectura: 46 % espera 5 min o menos en 2024 contra 41 % en 2023, 76 % espera
10 min o menos y 24 % acepta 15 min o más; en zona no urbana, 54 % y 46 % respectivamente.

**Corrección pendiente:** la tabla de
[`../Calibracion_Arrepentimiento_EV.md`](../Calibracion_Arrepentimiento_EV.md) §5 asigna 18 % a
"15 min", 5 % a "20–30 min" y 1 % a "más de 30 min": los tres últimos tramos están permutados. Los
agregados no cambian — por eso ningún chequeo lo detectó — pero la media sí: con los valores
representativos del apéndice A de
[`../Calibracion_Arrepentimiento_Revision.md`](../Calibracion_Arrepentimiento_Revision.md)
(0, 5, 10, 15, 30 y 120 min), `TMEU` pasa de 9,85 a 16,0 min.

**Salvedad:** la red de BC Hydro es 85 % carga rápida, así que esta es una distribución de paciencia
frente a carga rápida. Es la misma salvedad que ya anota
[`../Calibracion_Arrepentimiento_Revision.md`](../Calibracion_Arrepentimiento_Revision.md) §5.1, ahora
confirmada contra el informe.

#### Datos operativos y económicos

Todavía no están citados desde ningún documento del repo. Cubren varios de los pendientes de
[`../CLAUDE.md`](../CLAUDE.md).

- **`TFC`, `TRC` y `PDC`** (apéndice E): la red de carga rápida declara un *uptime* del **99 %** con
  soporte 24/7. Ancla la relación `TRC / (TFC + TRC) ≈ 1 %`, que es el dato que faltaba para las dos
  FDP de falla.
- **`RC` y `CCP`** (§4.3.4 y apéndice E): tarifa energética aprobada en 2024 de **36,09 ¢CAD/kWh** para
  carga rápida y 29,72 ¢/kWh para nivel 2; el informe pide llevar la rápida a 39,32 ¢/kWh desde abril
  de 2026. El costo de energía real del ejercicio fiscal 2025 (abril 2024 a marzo 2025) fue de
  **19,3 ¢/kWh despachado** contra 23,9 previstos, con un consumo auxiliar del sitio —iluminación y
  ventilación— del 10 % contra el 12 % supuesto. La tarifa es ≈ 1,9 veces el costo de energía.
- **Estructura por franja** (apéndice E): BC Hydro no tarifa la carga pública por franja horaria, pero
  su tarifa residencial opcional sí, con **descuento de 5 ¢/kWh de 23 a 7 y recargo de 5 ¢/kWh de 16
  a 21**. Sirve como orden de magnitud del diferencial pico/valle, no como cuadro tarifario aplicable.
- **`CC_MAX` y Análisis de Expansión** (§3.2.6): miden la congestión como la proporción de sesiones que
  arrancan **3 minutos o menos** después de que terminó la anterior en el mismo puerto, y reportan que
  baja a medida que sube la cantidad de puertos por sitio (tramos 1–2, 3–4, 5–7 y *hub* de 8 o más).
  Por eso abandonan la configuración de dos puertos. Los porcentajes están tachados; la conclusión no.
  Sus *hubs* van de 8 a 22 puertos.
- **Utilización por puerto** (§3.2.4): el modelo tarifario supone **30 % de utilización máxima** y, el
  primer año, 12 % en zona urbana y 2 % en zona no urbana.
- **Energía despachada por minuto** (tabla 7): 0,08 kWh/min a 7 kW, 0,60 a 50 kW, 0,87 a 90 kW y 1,33 a
  180 kW. La regresión energía–tiempo del TP 4 da 0,0859 kWh/min, o sea ≈ 5 kW: el dataset de
  California es carga de nivel 2, no rápida. (Esa regresión se rehace en el TP final, ver
  [`../CLAUDE.md`](../CLAUDE.md).)
- **Horizonte y `BM`** (§4.1 y §5.1): la relación ingreso/costo fue del **45 % en el primer año** contra
  un 98,7 % proyectado a diez años; los ingresos recién superan a los costos en 2030 y la
  subrecuperación acumulada 2025–2029 es de unos 66 millones de dólares. Es el respaldo empírico de por
  qué el horizonte de simulación tiene que cubrir varios años y de por qué el Análisis de Expansión no
  puede exigir recupero inmediato.
- **Tarifa por ocupación** (*idle fee*, §3.2.6): 40 ¢/min con 5 minutos de gracia; se aplicó al 1,8 % de
  las sesiones del año, con una duración media de 16 minutos, contra un supuesto original de 5 % de las
  sesiones y 5 minutos. No está en nuestro modelo: queda como mecanismo conocido y no adoptado.

Citado desde [`../Calibracion_Arrepentimiento_EV.md`](../Calibracion_Arrepentimiento_EV.md) §5 y
[`../Calibracion_Arrepentimiento_Revision.md`](../Calibracion_Arrepentimiento_Revision.md) §5.1.

---

## Fuentes citadas que no están acá

- **Dataset de sesiones de carga de California** — vive en Drive y lo monta el notebook; pesa
  demasiado para versionarlo. La ruta está en el [README](../README.md) del repositorio.
- **ACN-Data** (Lee, Li y Low, *e-Energy '19*) — no lo usamos de primera mano; entra citado a través
  de IDEAS, que lo usa para el perfil horario de demanda y la duración de las sesiones.
- **Cuadro tarifario de Edenor** — se cita por versión y fecha; cambia seguido, no tiene sentido
  congelarlo acá.
- **Gross, Shortle, Thompson y Harris, *Fundamentals of Queueing Theory* (Wiley, 2008)** — libro de
  texto, no lo versionamos. Es el origen real de la forma exponencial de balking que usa
  `sustainability-17-00336.pdf` §4.2.

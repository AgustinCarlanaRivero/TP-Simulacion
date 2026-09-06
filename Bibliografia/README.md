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

---

## Fuentes citadas que no están acá

- **Dataset de sesiones de carga de California** — vive en Drive y lo monta el notebook; pesa
  demasiado para versionarlo. La ruta está en el [README](../README.md) del repositorio.
- **ACN-Data** (Lee, Li y Low, *e-Energy '19*) — no lo usamos de primera mano; entra citado a través
  de IDEAS, que lo usa para el perfil horario de demanda y la duración de las sesiones.
- **Cuadro tarifario de Edenor** — se cita por versión y fecha; cambia seguido, no tiene sentido
  congelarlo acá.

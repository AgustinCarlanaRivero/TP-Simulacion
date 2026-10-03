# Bibliografía

Copias locales de las fuentes que usa el TP, para que el repo sea autocontenido y las citas no
dependan de que un link siga vivo. Cada entrada indica qué se usa de cada fuente y desde dónde se la
cita.

| Archivo | Qué es | Dónde se usa |
|---|---|---|
| [`Paper_TP_4.pdf`](Paper_TP_4.pdf) | Nuestro paper del TP 4 | Base del TP final |
| [`IDEAS_2024_Chattopadhyay_Kar_arXiv_2403.06223.pdf`](IDEAS_2024_Chattopadhyay_Kar_arXiv_2403.06223.pdf) | Modelo de impaciencia en estaciones de carga | Comparación de la paciencia propia (`PU = FI · TC`) contra la espera estimada |
| [`EAFO_Consumer_Monitor_2023_EU_Aggregated_Report.pdf`](EAFO_Consumer_Monitor_2023_EU_Aggregated_Report.pdf) | Encuesta europea a conductores de vehículos eléctricos | Calibración de `PUNEN` y `FI` |

---

## Detalle

### `Paper_TP_4.pdf`

**Estudio de la eficiencia técnica y operativa en la infraestructura de carga de vehículos eléctricos
a través de la simulación de eventos discretos en la Ciudad Autónoma de Buenos Aires.**
Carlana Rivero, Loglen, Millán, Ojeda Cabrera. UTN — FRBA.

- §2.1: regla de arrepentimiento del TP 4 (93 % con un vehículo en el puesto, 100 % con dos o más),
  reemplazada en el TP final por balking voluntario.
- §2.3: ajuste de las FDP de `IA` (Landau) y `TC` (Gumbel R).
- §2.4: precio de energía y regresión energía–tiempo.
- §3: resultados por escenario.

Citado desde [`../Propuesta_TP-FINAL.md`](../Propuesta_TP-FINAL.md) y [`../CLAUDE.md`](../CLAUDE.md).

### `IDEAS_2024_Chattopadhyay_Kar_arXiv_2403.06223.pdf`

**IDEAS: Information-Driven EV Admission in Charging Station Considering User Impatience to Improve
QoS and Station Utilization.** A. Chattopadhyay, S. Kar. Indian Institute of Technology, Delhi.
arXiv:2403.06223v1, 10 de marzo de 2024. Licencia CC BY-SA 4.0.

- Ecuación (9): la paciencia como fracción `z` del tiempo de carga del propio usuario, que es nuestro
  `FI`.
- Ecuación (5): la comparación de esa paciencia contra la espera estimada.

Citado desde [`../Propuesta_TP-FINAL.md`](../Propuesta_TP-FINAL.md), en la justificación de `EEU`.

### `EAFO_Consumer_Monitor_2023_EU_Aggregated_Report.pdf`

**Consumer Monitor 2023 — European Alternative Fuels Observatory, EU Aggregated Report.**
L. Vanhaverbeke, D. Verbist, G. Barrera (VUB-MOBI) y M. Csukas (FIER). Comisión Europea,
DG MOVE, junio de 2024. doi:10.2832/062076. Licencia CC BY 4.0.

- §3.4 y figura 10: espera declarada en puntos de carga públicos (31 % no espera, 32 % hasta 15 min,
  18 % hasta 30 min, 13 % hasta 1 h, 6 % más). De ahí salen `PUNEN = 31 %` y la media entre los que
  esperan (41,7 min) que ancla `FI`.

Citado desde [`../Propuesta_TP-FINAL.md`](../Propuesta_TP-FINAL.md), en los parámetros del
arrepentimiento.

---

## Fuentes citadas que no están acá

- **Knudsen (1972)**, *Econometrica* 40, 515–528 — artículo de revista, no lo versionamos. Es el
  origen de la regla de decisión de la cola observable con varios servidores que da `EEU`.
- **Dataset de sesiones de carga de California** — vive en Drive y lo monta el notebook; pesa
  demasiado para versionarlo. La ruta está en el [README](../README.md) del repositorio.
- **Cuadro tarifario de Edenor** — se cita por versión y fecha; cambia seguido, no tiene sentido
  congelarlo acá.

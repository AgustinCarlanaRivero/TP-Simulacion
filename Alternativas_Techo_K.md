# Techo del parque (`K`): alternativas y decisiones

Árbol de las alternativas evaluadas para estimar el techo `K` de la curva de crecimiento del parque de
enchufables de CABA (sección 2 del notebook), con sus pros y contras, y las decisiones de soporte que
afectan a varias ramas a la vez.

Los valores se calcularon con los datos de `Datos/` (IEA, *Global EV Outlook 2026*; IDECBA). Valores
comunes a todas las ramas:

| Dato | Valor |
|---|---|
| `T = 0` del motor | 1/7/2026 (cierre del 2.º trimestre de 2026) |
| `P(T=0)`: stock de enchufables de CABA que cargan en estaciones públicas | 1 688 vehículos |
| Cuota actual de CABA: % de enchufables entre los autos nuevos | 9,1 % |
| Fracción que carga en estaciones públicas | 29 % |
| Mercados maduros de referencia | Noruega, Suecia, Finlandia, Islandia, Dinamarca, Países Bajos, China |

**Estado actual del notebook:** rama **B1**, con mediana y sin sensibilidad. La fracción de automóviles
se aproxima con los patentamientos de toda la serie disponible, 2016 al 2.º trimestre de 2026 (73,3 %):
parque de automóviles de 1 230 424 y `K` = **217 662**. La velocidad de Gompertz se ajusta por
mínimos cuadrados en niveles, acotada a la máxima compatible con el techo de la cuota
(`c ≤ e · 61 % · ventas anuales · 29 % / K` = 0,183/año). Sin la cota daría 0,272/año y no pasaría
el control de 2.4 (ver la sección 3). Con la cota lo pasa, a cambio de implicar 6,3 % de enchufables
en `T = 0` contra el 9,1 % observado.

---

## 1. Árbol

```
K (techo del stock de enchufables de CABA que cargan en estaciones)
│
├── A) P(T=0) de CABA × mediana de [techo del país / stock del país en el cruce]
│   │
│   ├── A1) Techo de cada país a partir de su stock
│   │   ├── A11) Máximo histórico del stock
│   │   └── A12) Curva ajustada al stock (logística o Gompertz)   ← implementado hoy (logística)
│   │
│   └── A2) Techo de cada país = parque total del país × techo de su cuota de ventas
│       ├── A21) Techo de la cuota = máximo histórico
│       └── A22) Techo de la cuota = curva ajustada a la cuota (logística o Gompertz)
│
└── B) Parque total de autos de CABA × mediana del techo de la cuota de los países × 29 %
    ├── B1) Techo de la cuota = máximo histórico                  ← elección tentativa
    └── B2) Techo de la cuota = curva ajustada a la cuota (logística o Gompertz)
```

Las decisiones de soporte (sección 4) atraviesan el árbol:

| | Decisión | Ramas que afecta |
|---|---|---|
| A | Curva de proyección del stock de CABA: logística o Gompertz | Todas |
| B | Mediana o media geométrica | Rama A |
| C | Denominador estimado u observado | A12 |
| D | Sólo la mediana o también sensibilidad | Todas |
| E | Logística o Gompertz para estimar techos | A12, A22, B2 |
| F a I | Otras decisiones y supuestos (sección 5) | Varias |

**Cruce.** Es el instante en que la cuota de enchufables en las ventas de un país alcanzó el 9,1 % actual
de CABA. Es el punto equivalente, en ese país, a la etapa en que hoy está CABA.

**Techo de la cuota.** Es el porcentaje máximo de autos nuevos enchufables al que llega un país. En el
notebook se llama `S_MAX`.

---

## 2. Rama A: índice de crecimiento desde el cruce

`K = P(T=0) × mediana de (techo del país / stock del país en el cruce)`

Para cada país se calcula cuántas veces creció (o crecerá) su stock de enchufables desde que estaba en
la etapa actual de CABA hasta su techo. La mediana de esos cocientes se aplica al stock actual de CABA.

**Pros**
- No requiere datos de CABA más allá de su propia serie de stock.
- El cociente no tiene unidades, así que se traslada sin importar el tamaño de cada mercado.
- La fracción que carga en estaciones (29 %) queda incluida en `P(T=0)` y se mantiene en `K`.

**Contras**
- Es una analogía de toda la trayectoria posterior al cruce: supone que CABA va a recorrer un camino
  relativo parecido al del país típico.
- El resultado depende de cómo se estime el techo de cada país, que es el punto débil (ver A1 y A2).

**Control disponible.** En el cruce, el stock de cada país equivalía a entre 0,85 y 2,6 años de ventas
de enchufables, y el de CABA en `T = 0` equivale a 1,3. La "profundidad" del parque es comparable, lo
que respalda la analogía.

### A1) Techo de cada país a partir de su stock

**Problema común a A11 y A12.** El stock de enchufables no satura cuando la cuota de ventas satura:
sigue sumando lo que se vende hasta que se renueva todo el parque. Noruega, con 97 % de cuota, sigue
sumando unos 100 000 enchufables por año, y sólo el 36 % de su parque es enchufable. Una curva
ajustada al stock interpreta como saturación lo que en realidad es el paso de un crecimiento
exponencial a uno lineal.

#### A11) Máximo histórico del stock

El stock nunca baja, así que el máximo histórico es el último dato (2025).

| | Mediana | Media geométrica | Rango (mín.–máx.) |
|---|---|---|---|
| Cociente | 13,9 | 20,7 | 10,1 – 70,1 |
| `K` de CABA | **23 493** | 34 909 | 17 133 – 118 380 |

**Pros**
- Es lo más simple: no se ajusta ninguna curva.
- Se basa en un dato observado.

**Contras**
- **Es un piso por construcción.** Todos los stocks siguen creciendo, así que subestima el techo de
  todos los países, y más en los que están más lejos de saturar.
- Da el `K` más bajo de todas las ramas.

**Evaluación:** descartada como escenario central.

#### A12) Curva ajustada al stock

Es lo que hace hoy el notebook, con logística. Valores con denominador estimado (el de la curva en el
cruce) y observado (el stock interpolado del dato real), ver la decisión C:

| Curva | Denominador | Mediana | Media geom. | `K` (mediana) | Rango de `K` |
|---|---|---|---|---|---|
| Logística | Estimado (actual) | 33,3 | 28,5 | **56 294** | 24 970 – 96 855 |
| Logística | Observado | 37,1 | 34,1 | **62 698** | 25 234 – 133 986 |
| Gompertz | Estimado | 220,5 | 226,5 | 372 322 | 45 625 – 27,9 M |
| Gompertz | Observado | 121,7 | 206,3 | 205 490 | 39 460 – 18,3 M |

Techo de cada país con cada curva, contra lo observado:

| País | Stock 2025 | Logística | Gompertz | Parque total |
|---|---|---|---|---|
| Noruega | 1,10 M | 1,25 M (+13 %) | 1,74 M | 3,06 M |
| Suecia | 0,81 M | 0,98 M | 1,53 M | 5,06 M |
| Finlandia | 0,35 M | 0,45 M | 0,77 M | 2,92 M |
| Islandia | 64 000 | 86 000 | 188 000 | 256 000 |
| Dinamarca | 0,67 M | 1,52 M | 11,6 M | 2,91 M |
| Países Bajos | 1,20 M | 3,52 M | 1 025 M | 9,23 M |
| China | 44 M | 93 M | 1 511 M | 314 M |

**Pros**
- Con la logística, los errores estándar del techo son chicos (entre 1,8 % y 24 %).
- Ya está implementado.

**Contras**
- **Logística:** su simetría fuerza un techo cercano. En los países con la cuota cerca del máximo, como
  Noruega, pone el techo apenas por encima del último dato (+13 %), aunque el stock sigue sumando más
  de 100 000 vehículos por año. El error chico refleja la forma de la curva, no información de los
  datos.
- **Gompertz:** el techo queda indeterminado (errores de hasta 316 %) y en algunos casos es
  físicamente imposible: en Países Bajos y China supera el parque total del país.
- Requiere tres ajustes por país: logística y Gompertz sobre el stock, y logística sobre la cuota para
  el control de 2.4.

**Evaluación:** Gompertz, descartada. Logística, válida pero sesgada hacia abajo en los países con la
cuota cerca del máximo.

### A2) Techo de cada país = parque total del país × techo de su cuota de ventas

`techo del país = parque total del país (todos los autos) × techo de su cuota`

A largo plazo todo el parque se renueva, y la fracción enchufable del parque tiende a la cuota de
ventas. El parque de cada país sale de la misma base de la IEA: stock de enchufables / `EV stock share`
(proporción del parque que es enchufable).

**Qué supone la analogía.** El cociente de cada país se puede descomponer en factores. Es una
identidad que sirve para interpretar el método, no un paso del cálculo:

`techo / P(cruce)` = techo de la cuota / (profundidad × 9,1 % × renovación)

| Factor | Definición | Situación en CABA |
|---|---|---|
| Techo de la cuota | % máximo de enchufables en las ventas | **Supuesto**: igual a la mediana de los países |
| Profundidad | Stock / ventas anuales de enchufables en el cruce | **Controlado**: CABA 1,3, dentro del rango de los países |
| 9,1 % | Cuota en el cruce | Igual por definición del cruce |
| Renovación | Ventas totales de autos / parque total | **Supuesto**: hoy va de 2,5 % (Finlandia) a 7,8 % (China) |

La identidad es exacta con el parque del momento del cruce. El cálculo usa el parque de 2025, así que
es una aproximación.

**Relación con la rama B.** A2 equivale a B con el parque de CABA estimado por analogía en lugar de
observado.

**Pros**
- Corrige el problema de A1: el techo de cada país es el que alcanzaría cuando todo su parque se haya
  renovado, no donde frena una curva ajustada.
- No requiere datos de CABA más allá de su serie de stock.
- Sólo hace falta un ajuste por país (o ninguno, en A21).

**Contras**
- Suma la serie `EV stock share`, que la IEA redondea a dos cifras significativas (error del orden de
  4 % en el parque).
- Los supuestos de analogía son tres (techo de la cuota, renovación y profundidad), contra uno de la
  rama B.
- Más del doble de `K` que la rama A12 actual.
- Deja el control de 2.4 al límite (ver la sección 6).

#### A21) Techo de la cuota = máximo histórico

| | Mediana | Media geométrica | Rango |
|---|---|---|---|
| Cociente | 66,1 | 73,6 | 38,4 – 188,9 |
| `K` de CABA | **111 592** | 124 224 | 64 861 – 318 968 |

**Pros:** no ajusta ninguna curva.
**Contras:** en los países donde la cuota sigue subiendo (Dinamarca, Países Bajos, China), el máximo
es un piso.

#### A22) Techo de la cuota = curva ajustada a la cuota

| Curva | Mediana | Media geom. | `K` (mediana) | Rango de `K` |
|---|---|---|---|---|
| Logística | 77,8 | 78,0 | **131 396** | 71 605 – 326 502 |
| Gompertz | 97,4 | 86,3 | 164 508 | 82 563 – 328 833 |

**Pros:** extrapola la cuota de los países que todavía crecen.
**Contras:**
- Agrega un ajuste por país.
- En los países que siguen subiendo, el techo es una extrapolación.
- Con Gompertz, Noruega y Países Bajos quedan en la cota del 100 %.

---

## 3. Rama B: parque de CABA × techo de la cuota

`K = parque total de automóviles de CABA × mediana del techo de la cuota de los países × 29 %`

El parque total incluye todos los autos, enchufables o no. A largo plazo, la fracción enchufable del
parque de CABA tiende al techo de la cuota de ventas, y el 29 % deja sólo a los que cargan en
estaciones públicas.

**Dato requerido:** el parque de **automóviles** de CABA, sin comerciales livianos ni pesados,
**radicados en CABA** (el mismo criterio que los patentamientos de 2.1), con fecha lo más cercana
posible a mediados de 2026. Si sólo hay parque registrado y no parque activo, se usa ése y se lo
explicita: los registros suelen sobrestimar el parque porque muchas bajas nunca se informan. Fuentes
posibles: IDECBA o DNRPA.

**Dato disponible: `Datos/Parque Automotor CABA.xlsx`.** Es el cuadro TPTE_PA del IDECBA, sobre datos de
la DNRPA: parque automotor mensual de CABA y del país, de 2010 a octubre de 2026.
- **Valor a usar: junio de 2026, 1 679 722 vehículos.** Coincide con `T = 0`, aunque es provisorio
  (lo son todos los datos desde mayo de 2026).
- **Radicados en CABA:** sí. Son automotores inscriptos en los registros seccionales de CABA, el mismo
  criterio que los patentamientos de 2.1.
- **Activo o registrado:** según la ficha técnica, son automotores "en condiciones de circular", así
  que se excluyen las bajas registradas. Igual se basa en el registro: los autos que dejaron de
  circular sin darse de baja siguen contando.
- **Crecimiento:** el parque crece entre 1 % y 2 % por año (1,57 M en 2021, 1,68 M en 2026). El supuesto
  de parque constante es una aproximación.

**Aproximación de la fracción de automóviles.** El cuadro incluye **todos los automotores**
(automóviles, comerciales livianos y pesados), y la rama B necesita sólo automóviles. No hay un dato del
parque por tipo, así que se aproxima la fracción de automóviles con la de los patentamientos de CABA
(`Total Patentamientos CABA.xlsx`, columnas de automóviles y total):

| Año | 2020 | 2021 | 2022 | 2023 | 2024 | 2025 | 2026 (1.er sem.) | 2020–2026 |
|---|---|---|---|---|---|---|---|---|
| Automóviles / total patentado | 76,0 % | 69,0 % | 66,6 % | 64,6 % | 70,3 % | 74,3 % | 76,0 % | **70,7 %** |

`parque de automóviles ≈ 1 679 722 × 70,7 % ≈ 1 187 700`

**Adoptado en el notebook: toda la serie disponible, 2016 al 2.º trimestre de 2026, 73,3 %** (parque de
automóviles de 1 230 424, `K` = 217 662). El parque acumula ventas de muchos años, así que la ventana
más larga es la que mejor lo aproxima, y diluye la caída de 2021–2023 (64,6 % a 68,9 %). Los valores de
la tabla de abajo son con 70,7 %.

Supuesto a explicitar: la composición del parque por tipo de vehículo es la de los patentamientos. Es una aproximación, porque el parque acumula décadas de ventas con otra composición. El
rango anual (64,6 % a 76,0 %) da un parque de automóviles de entre 1,09 M y 1,28 M.

**Resultado con B1** (`K` = parque de automóviles × 61 % × 29 %), proyectado con Gompertz:

| Escenario | Parque | `K` | Crecimiento a 1 / 3 / 5 / 10 años | Cuota implícita máx. (actual / 2023–2024) | 95 % de `K` |
|---|---|---|---|---|---|
| **Central** (70,7 %, techo de la cuota 61 %) | 1 187 700 | **210 103** | ×3,2 / ×14,9 / ×36,4 / ×90,9 | **88 % / 132 %** | 2043 |
| Automóviles 64,6 % | 1 085 100 | 191 954 | ×3,2 / ×14,5 / ×35,0 / ×84,7 | 82 % / 122 % | 2043 |
| Automóviles 76,0 % | 1 276 600 | 225 829 | ×3,2 / ×15,1 / ×37,5 / ×96,2 | 93 % / 140 % | 2043 |
| Techo de la cuota 53 % | 1 187 700 | 182 548 | ×3,1 / ×14,4 / ×34,2 / ×81,5 | 79 % / 118 % | 2043 |
| Techo de la cuota 71 % | 1 187 700 | 244 546 | ×3,2 / ×15,4 / ×38,8 / ×102,2 | 100 % / 149 % | 2044 |
| Sin aproximar (todos los automotores) | 1 679 722 | 297 143 | ×3,2 / ×16,1 / ×42,1 / ×118,4 | 117 % / 175 % | 2045 |

**Consecuencia: B1 no pasa el control de 2.4 en ningún escenario.** Aun en el más bajo, la curva
necesita que el 79 % de los autos nuevos sea enchufable, por encima del techo de la cuota de los
mercados maduros (61 %). Con el mercado de 2023–2024 supera el 100 %, que es físicamente imposible.

El origen es un conflicto entre las dos fuentes de la curva. El techo estructural es razonable, pero a
ese techo se llega en lo que tarda en renovarse el parque: 1 187 700 / 82 677 ≈ **14 años de ventas**
al ritmo actual, contados después de que la cuota ya se estabilizó. Gompertz, con la velocidad
ajustada a la aceleración actual de CABA, intenta llegar antes. En Gompertz anclada, el cociente entre
la cuota máxima y la de `T = 0` es `K / (e · P(T=0) · ln(K / P(T=0)))` (9,8), y no depende de `c`:
ninguna velocidad reproduce el 9,1 % actual y respeta el 61 %.

**Resuelto en el notebook acotando la velocidad** (ver "Estado actual"). Se descartaron borrar o aflojar
el control y bajar `K`. Queda como alternativa más fiel pasar la "S" a la cuota (logística de la cuota
hasta 61 % y stock por acumulación), que cambia la curva de la propuesta.

**Pros**
- Es la más simple: una multiplicación.
- Un solo supuesto de analogía: que CABA llegue al techo de la cuota de los mercados maduros. El
  techo de la cuota es un porcentaje del mercado, la magnitud más comparable entre países.
- Usa el parque real de CABA en lugar de estimarlo por analogía (diferencia con A2).
- No hace falta el cruce para calcular `K`.
- Es compatible con la propuesta: "basándonos en las características de la misma para otros países".

**Contras**
- El dato disponible incluye todos los automotores: hay que aproximar la fracción de automóviles (ver
  abajo).
- Supone un parque de CABA constante en el horizonte.
- Hay que reescribir 2.3, incluida la frase "`K` no depende del parque total de CABA".
- Con un parque grande, el control de 2.4 puede quedar al límite (ver la sección 6).

**Control extra que permite:** `parque / ventas anuales de autos` da los años de renovación del
parque de CABA. Se puede contrastar con el año en que la curva llega al 95 % de `K`.

**Comparación con las otras ramas.** Con B1, `K` ≈ 0,1769 × parque. El parque con el que B1 iguala a
cada una de las otras alternativas:

| Iguala a | `K` | Parque de CABA equivalente |
|---|---|---|
| A12 logística (actual) | 56 294 | ~318 000 autos |
| A21 | 111 592 | ~631 000 autos |
| A22 logística | 131 396 | ~743 000 autos |

#### B1) Techo de la cuota = máximo histórico (elección tentativa)

| País | Máximo histórico | Último dato (2025) |
|---|---|---|
| Noruega | 97 % | 97 % |
| Dinamarca | 71 % | 71 % |
| Islandia | 71 % (2023) | 62 % |
| Suecia | 61 % | 61 % |
| Países Bajos | 58 % | 58 % |
| Finlandia | 57 % | 57 % |
| China | 53 % | 53 % |
| **Mediana** | **61 %** | **61 %** |

`K` = parque × 61 % × 29 % = **0,1769 × parque**.

**Pros**
- No ajusta ninguna curva.
- Se explica en una frase: el mayor porcentaje de enchufables en las ventas que alcanzó cada mercado
  maduro.
- No lo afecta un año malo, como la caída de Islandia a 43 % en 2024.
- El último dato da la misma mediana: el resultado no depende de esa elección.

**Contras**
- Un pico excepcional puede inflar el techo de un país: es el caso de Islandia en 2023. A la mediana
  no la afecta, porque Islandia queda por encima del valor central.
- En los países donde la cuota sigue subiendo, el máximo es un piso. Eso hace a `K` conservador.

#### B2) Techo de la cuota = curva ajustada a la cuota

| Curva | Mediana del techo | `K` |
|---|---|---|
| Logística (el `S_MAX` actual del notebook) | 63,7 % | 0,1846 × parque |
| Gompertz | 69,7 % | 0,2021 × parque |

**Pros:** extrapola la cuota de los países que todavía crecen.
**Contras:**
- Agrega un ajuste por país.
- Con logística, la diferencia con B1 es de 4 %. No justifica la complejidad extra.
- Con Gompertz, Noruega y Países Bajos quedan en la cota del 100 %, y el techo queda mal determinado.

---

## 4. Decisiones de soporte

### A) Curva de proyección del stock de CABA: logística o Gompertz

No es la estimación del techo: es la curva `P(t)` de CABA que se proyecta con el `K` elegido y de la que
sale el factor sobre el IA. **Ya está decidida (Gompertz) y actualizada en la propuesta.** Se mantiene
por cualquier `K`:

- **Logística:** con cualquier `K`, llega al 95 % del techo entre 2030 y 2032 y después queda plana. A
  10 años, la cuota implícita es nula. Supera la cota física (100 % de los autos nuevos enchufables)
  desde `K` ≈ 63 000, y el techo de la cuota de los países (61 %) desde `K` ≈ 25 000 en el mercado
  de 2023–2024.
- **Gompertz:** reproduce la cuota observada en `T = 0` y respeta el techo de la cuota hasta
  `K` ≈ 130 000 con el mercado actual, y hasta `K` ≈ 79 000 con el de 2023–2024.

### B) Mediana o media geométrica (rama A)

| Hoja | Mediana | Media geométrica | Media aritmética |
|---|---|---|---|
| A12 logística, denominador estimado | 33,3 | 28,5 | 31,9 |
| A21 | 66,1 | 73,6 | — |
| A22 logística | 77,8 | 78,0 | — |

- **Mediana:** sólo depende del orden, así que la protegen los extremos. Noruega es un caso de política
  excepcional, y en A12 Países Bajos tiene el techo peor determinado. Con siete valores, la mediana es
  el valor de un solo país.
- **Media geométrica:** es el promedio natural de cocientes, porque se combinan multiplicando. Pero la
  mueve cualquier país extremo.

**Recomendación:** mediana, con la justificación en una línea. La diferencia entre las dos es chica
frente a la dispersión entre países (decisión D). En la rama B no aplica: es un promedio de
porcentajes, y la media geométrica no aporta.

### C) Denominador estimado u observado (A12)

En A12, el stock del país en el cruce puede ser el que predice la curva ajustada (lo actual) o el
observado (interpolado de los datos). En algunos países la curva se aleja mucho del dato:

| País | Estimado (logística) | Observado | Diferencia |
|---|---|---|---|
| Noruega | 26 300 | 15 700 | +68 % |
| Dinamarca | 45 600 | 26 800 | +70 % |
| Países Bajos | 61 300 | 94 700 | −35 % |

Con el observado, la mediana pasa de 33,3 a 37,1, y `K` de 56 294 a 62 698.

**Recomendación:** observado. El estimado mezcla un valor ajustado con uno real. En A11, A21 y A22 no
hay curva sobre el stock, así que sólo existe el observado.

### D) Sólo la mediana o también análisis de sensibilidad

La dispersión entre países pesa mucho más que cualquier otra elección de la sección 2:

| Rama | Escenarios | `K` resultante |
|---|---|---|
| A12 | Mínimo / mediana / máximo de los cocientes | 24 970 / 56 294 / 96 855 (×4) |
| B1 | Mínimo (53 %, China) / mediana (61 %) / segundo mayor (71 %) | −13 % / central / +16 % |

En B1, el escenario alto usa el segundo mayor para dejar afuera a Noruega (97 %) como caso
excepcional.

**Sesgo de selección.** Los siete países son los que más rápido adoptaron el auto eléctrico, que es
justamente por qué ya maduraron. Tienen mayores ingresos, incentivos fuertes y más oferta de modelos
que Argentina. La mediana tiende a sobrestimar el techo de CABA, y no hay mercados maduros menos
favorables para compensarlo.

**Recomendación:** mediana como escenario central, más dos escenarios de sensibilidad (bajo y alto)
corridos en el barrido de la simulación.

### E) Logística o Gompertz para estimar techos (A12, A22, B2)

| Hoja | Logística | Gompertz | Recomendación |
|---|---|---|---|
| A12 (stock) | Determinada, pero sesgada hacia abajo | Indeterminada, techos imposibles | Logística |
| A22, B2 (cuota) | Forma adecuada: la cuota tiene techo natural (100 %) y se estabiliza | Dos países en la cota del 100 % | Logística |

En cualquier caso, si se elige el máximo histórico (A11, A21, B1), esta decisión no aplica.

---

## 5. Otras decisiones y supuestos (no estaban en la lista original)

### F) Estimación de la velocidad `c` de Gompertz (todas las ramas)

Con `K` fijo y la curva anclada a `P(T=0)`, queda un único parámetro libre.

| Método | `c` (1/año) | Crecimiento a 10 años | Comentario |
|---|---|---|---|
| Mínimos cuadrados en niveles (actual) | 0,361 | ×30,3 | Estándar en la literatura de difusión; usa los 26 trimestres |
| Dos puntos (`T = 0` y un año antes) | 0,351 | ×30,0 | Fórmula cerrada. Pero según el punto elegido va de 0,277 a 0,456 |
| Mínimos cuadrados en logaritmos | 0,196 | ×20,3 | Da el mismo peso a 2020–2021, cuando el stock era casi nulo |

**Recomendación:** mantener mínimos cuadrados en niveles. Los valores de la tabla son con
`K` = 56 294; con otro `K` se vuelve a ajustar `c`.

### G) Último dato o máximo histórico para el techo de la cuota (A21, B1)

**Decidido: máximo histórico.** Los dos dan la misma mediana (61 %), y conviene dejar esa
coincidencia anotada en el notebook.

### H) Muestra de países (todas las ramas)

Hoy son siete mercados maduros. Hay dos variantes posibles:
- excluir a Noruega del escenario central por excepcional (con la mediana, el efecto es casi nulo);
- ampliar la muestra a medida que otros países pasen su inflexión.

El sesgo de selección se describe en la decisión D.

### I) Supuestos comunes a explicitar

- **Fracción que carga en estaciones públicas:** 29 %, constante (supuesto 6 de 2.1).
- **Parque de CABA constante** en el horizonte (ramas A2 y B).
- **Autos patentados en CABA como base de la demanda.** En CABA también cargan autos del conurbano, y
  autos de CABA cargan afuera. Se supone que esos flujos se compensan, o que la demanda escala con el
  parque patentado en CABA. Hoy no figura en el notebook.
- **Nivel de ventas en el control de 2.4.** Se supone constante en el nivel de los últimos cuatro
  trimestres (82 677 autos por año), y se informa también con el de 2023–2024 (55 263). No entra en la
  curva ni en el motor: es sólo un control de plausibilidad.

---

## 6. Resumen de las hojas, proyectadas con Gompertz

Todas las hojas con mediana y denominador observado, salvo la de A12 marcada como actual. Cuota
implícita máxima: % de enchufables en los patentamientos de CABA que necesita la curva, con el mercado
actual / el de 2023–2024. Límites: techo de la cuota de los países, 61 %; cota física, 100 %.

| Hoja | `K` | Crecimiento a 1 / 3 / 5 / 10 años | Cuota implícita máx. | 95 % de `K` | Control |
|---|---|---|---|---|---|
| A11 | 23 493 | ×2,6 / ×7,2 / ×10,7 / ×13,6 | 17 % / 25 % | 2035 | Pasa |
| A12 logística, actual | 56 294 | ×2,9 / ×10,2 / ×18,7 / ×30,3 | 31 % / 47 % | 2038 | Pasa |
| A12 logística, observado | 62 698 | ×2,9 / ×10,6 / ×19,9 / ×33,4 | 34 % / 51 % | 2038 | Pasa |
| A12 Gompertz | 205 490 | ×3,2 / ×14,8 / ×36,0 / ×89,4 | 87 % / 129 % | 2043 | No pasa |
| A21 | 111 592 | ×3,1 / ×12,6 / ×27,1 / ×54,7 | 53 % / 79 % | 2040 | Pasa con el mercado actual; no pasa con el de 2023–2024 |
| A22 logística | 131 396 | ×3,1 / ×13,2 / ×29,3 / ×62,5 | 60 % / 90 % | 2041 | Justo con el mercado actual; no pasa con el de 2023–2024 |
| A22 Gompertz | 164 508 | ×3,1 / ×14,0 / ×32,6 / ×75,0 | 72 % / 108 % | 2042 | No pasa |
| B1 | 210 103 | ×3,2 / ×14,9 / ×36,4 / ×90,9 | 88 % / 132 % | 2043 | No pasa |
| B2 logística | 219 307 | Similar a B1 (`K` 4 % mayor) | — | — | No pasa |

---

## 7. Próximos pasos si se confirma B1

1. Resolver el conflicto con el control de 2.4 (sección 3, "Consecuencia"). El dato del parque ya está
   en `Datos/Parque Automotor CABA.xlsx`: se usa junio de 2026 por la fracción de automóviles aproximada
   con los patentamientos.
2. En los dos notebooks, en la misma celda y en el mismo commit:
   - **2.3:** el techo pasa a ser `parque × mediana del máximo histórico de la cuota × 29 %`. El
     cruce y los cocientes quedan como control o se eliminan, según decida el equipo. Hay que
     explicitar los supuestos de la sección 5 y el sesgo de selección.
   - **2.4:** rehacer la comparación y los controles con el `K` nuevo. Usar 61 % como techo de la
     cuota y agregar el control de los años de renovación del parque.
   - **2.5:** actualizar los valores del factor.
   - **Barrido:** sumar los escenarios de sensibilidad de la decisión D.
3. Actualizar en `CLAUDE.md` la descripción del techo `K` (de los países sale el techo de la cuota, no
   el `K`). La propuesta no necesita cambios.

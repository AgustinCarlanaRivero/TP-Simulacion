# Cómo modelar el arrepentimiento en el TP final — alternativas

> **Estado:** documento de trabajo para discutir entre los cuatro. No es una decisión tomada.
> Lo que elijamos hay que bajarlo a [Propuesta_TP-FINAL.md](Propuesta_TP-FINAL.md): siglas nuevas,
> tabla de eventos y TEF (ver [§9](#9-qué-hay-que-tocar-en-la-propuesta)).

---

## 1. Resumen

El arrepentimiento es la variable que más pesa en el TP final y hoy es la peor fundamentada del
modelo: la propuesta **no la define en ninguna parte** (aparece `CARRUM(i)` en la condición de
expansión, pero nunca se dice cómo se genera un arrepentido), y lo único que tenemos es la regla
hardcodeada del TP 4, que además **no se puede portar tal cual** al esquema de fila única.

Seis alternativas sobre la mesa, ordenadas de menos a más ambiciosa:

| # | Alternativa | Mecanismo | Esfuerzo | Veredicto |
|---|---|---|---|---|
| **A** | Umbral por longitud de cola (TP 4 corregido) | Balking | Bajo | Sirve sólo como **baseline** para comparar |
| **B** | Probabilidad continua en la longitud de cola | Balking | Bajo | Mejor que A, pero sin anclaje empírico |
| **C** | Paciencia empírica vs. espera estimada | Balking | Medio | **Recomendada** como núcleo |
| **D** | Paciencia como reloj: abandono en cola | Reneging | Medio-alto | **Recomendada**; habilita validar contra Erlang-A |
| **E** | Híbrido C + D con/sin información al usuario | Ambos | Alto | **Objetivo final**; agrega un eje experimental gratis |
| **F** | Población heterogénea de perfiles | Se monta sobre C/D/E | Medio | Opcional; explica el piso irreducible de pérdida |

**Recomendación:** ir a **E**, pero construido por etapas (A → D → C → E), porque cada etapa deja un
resultado presentable aunque nos quedemos sin tiempo. Detalle en [§7](#7-recomendación-y-plan-por-etapas).

---

## 2. Balking vs. reneging: los dos arrepentimientos

En teoría de colas son dos fenómenos distintos y el TP 4 los mezcló en uno solo:

- **Balking (no ingresar):** el auto llega, ve la estación, y decide no entrar. **No ocupa cola, no
  espera, no consume capacidad.** Es lo único que modela el TP 4.
- **Reneging (abandonar):** el auto entra, hace cola, espera, y en algún momento se va sin cargar.
  **Ocupó lugar en la cola durante todo ese tiempo** y degradó la espera de los que estaban atrás.

La diferencia no es académica: económicamente el cliente perdido es el mismo, pero **el impacto sobre
el sistema es opuesto**. El que hace balking descomprime; el que hace reneging congestiona y después
se va igual. Como nuestra condición de expansión se dispara con `CARRUM(i)`, la elección entre uno y
otro cambia la política de expansión que va a salir del modelo, no sólo el valor de `PARR`.

Además hay un tercer "no ingreso" que ya está en la propuesta y **no** es arrepentimiento: el filtro
`R ≤ PDCE/100`. Conviene dejarlo escrito para no contar dos veces lo mismo:

- **`PDCE`** = decisión **antes** de ver la estación (ubicación, marca, app, competencia).
- **Arrepentimiento** = decisión **después** de ver la cola.

Son ortogonales y así hay que implementarlos.

---

## 3. Qué hicimos en el TP 4 y por qué no se puede copiar

### 3.1. La regla

En el paper del TP 4 (§2.1) escribimos:

- puesto libre → ingresa;
- 1 vehículo en el puesto → abandona con probabilidad **93 %**;
- 2 o más vehículos → abandona con probabilidad **100 %**.

### 3.2. Tres problemas (uno de ellos, un bug)

**(a) El código no hace lo que dice el paper.** En `TP 4 Simu.ipynb`, `arrepentimiento()` evalúa
`NS[i]` *antes* de incrementarlo:

```python
if cant_autos <= 1:   return False          # 0 o 1 autos -> ingresa siempre
elif cant_autos <= 2: r = random.random()   # 2 autos     -> 93 % abandona
else:                 return True           # >=3 autos   -> 100 % abandona
```

O sea: el código es **un auto más permisivo** que la regla publicada. Los `PARR` del 53–60 % que
reportamos salen del código, no del paper. Antes de comparar cualquier resultado nuevo contra el
TP 4 hay que decidir cuál de las dos versiones es la válida y dejarlo dicho.

**(b) El umbral está sobre la cantidad de autos en el puesto, y eso con fila única no significa
nada.** En el TP 4 había *una cola por cargador*, entonces "2 autos en el puesto" = "1 cargando + 1
esperando". En el TP final hay **una sola cola por estación** con `CD(i)` cargadores: portar
`CA(i) ≥ 3 → 100 %` con `CD(i) = 4` haría que la estación rechace autos **teniendo cargadores
libres**. El umbral tiene que estar sobre la **cola**, no sobre el sistema:

```
q(i) = max(CA(i) - CD(i), 0)        # autos efectivamente esperando
```

y normalizado por cargador (`q(i)/CD(i)`) si queremos que la regla no se rompa cuando la estación
crece de 4 a `CC_MAX` cargadores. Una regla de umbral fijo hace que **expandir empeore
artificialmente el arrepentimiento relativo**, que es justo la decisión que el modelo tiene que tomar.

**(c) Ignora el tiempo, que es la variable de decisión real.** Un auto no decide por "cuántos hay
adelante" sino por "cuánto voy a esperar". Con `TC` ~ Gumbel(84,39; 60,19) — media ≈ 119 min sin
truncar — un solo auto adelante ya son **dos horas** de espera. La regla por longitud de cola no
distingue entre esperar detrás de un auto que arranca y uno que está por terminar, y sobre todo **no
reacciona a las fallas ni al mantenimiento**: si se cae un cargador, la espera se duplica y la regla
por longitud no se entera. Como `TMP` es una de nuestras tres variables de control, esto es grave:
con un modelo de balking por longitud, **`TMP` casi no puede mover `PARR`**, y el análisis de
sensibilidad de mantenimiento se queda sin efecto que medir.

---

## 4. Qué aportan las fuentes

| Fuente | Qué nos da | Cómo se usa |
|---|---|---|
| **EAFO Consumer Monitor 2023** (Comisión Europea) | Distribución empírica de espera tolerada por conductores de BEV | Es la **FDP de paciencia**. Único dato empírico y citable que tenemos |
| **IDEAS** (Chattopadhyay & Kar, arXiv 2403.06223) | Distingue balking y reneging; perfiles optimista/estándar/pesimista; informar la espera real reduce el reneging hasta **94 %** | Justifica separar los dos mecanismos y habilita el escenario "con/sin información" |
| **Alsabbagh, Wu & Ma (IEEE TII, 2020)** | "Time anxiety": la impaciencia **crece con el tiempo transcurrido**, con cuatro perfiles de conductor (NTAD/LTAD/MTAD/HTAD) y tres formas funcionales (log, lineal, exponencial) | Justifica una tasa de abandono **creciente en el tiempo de espera** y la mezcla de perfiles |
| **Paper del TP 4** | La regla del 93 % y los resultados de referencia | Baseline de comparación |

### 4.1. El dato de EAFO (el más importante)

Pregunta a conductores de BEV europeos: *cuál fue la espera más larga en un punto de carga público*.

| Respuesta | % |
|---|---|
| Nunca espero; si está ocupado me voy sin cargar | **31 %** |
| 15 minutos o menos | **32 %** |
| Más de 15 y hasta 30 minutos | **18 %** |
| Más de 30 minutos y hasta 1 hora | **13 %** |
| Más de 1 hora | **6 %** |

Tres lecturas que hay que hacer explícitas si usamos esto:

1. **El 31 % es un piso duro de balking.** No es la cola de una distribución continua: es un
   segmento que se va *siempre*, tenga la estación 4 o 40 cargadores. Ninguna expansión lo recupera.
   Esto solo ya cambia la conclusión económica del TP.
2. **Está censurada a derecha.** La pregunta es "cuánto esperaste", no "cuánto tolerarías": quien
   nunca se encontró con una cola larga reporta poco aunque su paciencia sea alta. Es decir, la
   distribución **subestima** la paciencia real. Es un supuesto a declarar, no a esconder.
3. **Es Europa, no CABA.** Mismo problema que ya asumimos con el dataset de California. El rango
   entre países del "15 minutos o menos" va de **10 % (Lituania) a 42 % (Francia)**: sirve como
   rango de sensibilidad razonable en vez de inventar uno.

---

## 5. Las alternativas

Todas asumen fila única FCFS por estación y `q(i) = max(CA(i) − CD(i), 0)`.

---

### A. Umbral por longitud de cola (TP 4 corregido y parametrizado)

**Idea.** Lo mismo que el TP 4, pero con el umbral sobre la cola y sin el 100 % duro:

```
P(arrepentirse | q) = 0            si q < Q_MIN
                    = PARRB        si Q_MIN <= q < Q_MAX
                    = 1            si q >= Q_MAX
```

con `Q_MIN`, `Q_MAX` expresados en autos en cola **por cargador**.

- **Datos que necesita:** ninguno nuevo. `PARRB`, `Q_MIN`, `Q_MAX` son supuestos.
- **Cambios en la propuesta:** ninguno estructural. No agrega eventos ni entradas a la TEF.
- **Pros:** trivial de implementar; comparable con el TP 4; barrer `PARRB` es un análisis de
  sensibilidad legítimo.
- **Contras:** los tres problemas de [§3.2](#32-tres-problemas-uno-de-ellos-un-bug) siguen ahí salvo el (b);
  no reacciona a `TMP` ni a las fallas; los valores no tienen respaldo empírico propio.
- **Cuándo conviene:** como **baseline** contra el que mostrar que el modelo nuevo aporta algo. Vale
  la pena dejarlo implementado aunque adoptemos otro.

---

### B. Probabilidad continua en la longitud de cola

**Idea.** Reemplazar el escalón por una función suave, que es la forma estándar de balking en la
literatura de colas:

```
P(arrepentirse | q) = 1 - e^(-BETA * q / CD(i))
```

- **Datos que necesita:** un solo parámetro `BETA`, calibrable para que `PARR` a `CD = 4` caiga cerca
  del 53–60 % del TP 4 (así queda anclado a algo, aunque sea a nuestro propio resultado anterior).
- **Cambios en la propuesta:** ninguno estructural.
- **Pros:** un parámetro, sin discontinuidades; escala solo con `CD(i)`; el barrido de `BETA` da una
  curva de sensibilidad limpia.
- **Contras:** `BETA` no sale de ningún dato; sigue ignorando el tiempo y por lo tanto las fallas y
  `TMP`. Es "A pero prolijo", no un modelo mejor.

---

### C. Paciencia empírica vs. espera estimada — *recomendada como núcleo*

**Idea.** Cada auto que llega sortea su tolerancia `TMEU` de la distribución de EAFO, estima cuánto
va a esperar, y se arrepiente si la espera estimada supera su tolerancia.

```
TMEU     ~ F_paciencia          # 31 % en 0; el resto por tramos (EAFO)
W_est(i)  = (q(i) + 1) / CD(i) * E[TC]
se arrepiente  <=>  W_est(i) > TMEU
```

- **Datos que necesita:** la tabla de EAFO + una decisión sobre cómo repartir dentro de cada tramo
  (uniforme es lo más honesto; exponencial por tramos si queremos suavidad) y un tope para el
  "más de 1 hora".
- **Cambios en la propuesta:** una FDP nueva (`TMEU`) en Datos. **No agrega eventos ni TEF.**
- **Pros:**
  - Es el único que se apoya en un dato empírico publicado y citable.
  - Un solo mecanismo produce el piso del 31 % *y* la sensibilidad a la congestión.
  - **Reacciona a la capacidad y a las fallas**: si cae un cargador, `CD(i)` baja, `W_est` sube y
    `PARR` sube. Recién acá `TMP` tiene un canal por el que afectar el arrepentimiento.
  - `PARR` queda en las mismas unidades que `PEC`, que es lo que el informe compara.
- **Contras:**
  - Hay que asumir cómo estima el usuario la espera. `W_est` ignora el tiempo residual del que ya
    está cargando (sobreestima un poco). Es un supuesto a declarar.
  - El dato es europeo y está censurado a derecha ([§4.1](#41-el-dato-de-eafo-el-más-importante)).
  - Sigue siendo **sólo balking**: nadie abandona después de haber esperado, y por lo tanto la cola
    nunca se descomprime sola.
  - Ojo con la interacción con `TC`: con media ≈ 119 min y paciencias de 15–30 min, **cualquier**
    cola genera arrepentimiento casi total. El modelo va a ser durísimo con la congestión. Puede ser
    correcto, pero hay que anticiparlo y explicarlo, no descubrirlo en los resultados.

---

### D. Paciencia como reloj: abandono en cola (reneging) — *recomendada*

**Idea.** El auto entra y hace cola, pero con vencimiento: si no empezó a cargar antes de `TMEU`, se va.

```
al ingresar a la cola:  TAB(i)(k) = T + TMEU_k
TEF:                    TPAB(i) = min_k TAB(i)(k)
evento nuevo:           "Arrepentimiento de auto en cola (i)"
```

- **Datos que necesita:** la misma FDP `TMEU` de la alternativa C.
- **Cambios en la propuesta:** **una fila nueva en la tabla de eventos y una entrada nueva en la
  TEF.** Además `CA(i)` deja de alcanzar como estado: hay que llevar la cola con los vencimientos.
- **Pros:**
  - Es el modelo canónico de la literatura (**M/M/c+G**), y con paciencia exponencial es **Erlang-A**,
    que tiene fórmula cerrada para probabilidad de abandono y espera media. Eso nos da un **test de
    validación del motor** que hoy no tenemos: corrida degenerada (una estación, sin fallas, sin
    expansión, IA/TC/paciencia exponenciales) contra la fórmula. Es exactamente lo que pide la regla
    de "Verificación del motor" de `CLAUDE.md`, y es el argumento más fuerte a favor de esta opción.
  - `PARR` y `PEC` quedan consistentes: el que abandona **efectivamente esperó**.
  - Reacciona a fallas, a `TMP` y a la expansión por el canal correcto (el tiempo real de espera, no
    una estimación).
- **Contras:**
  - Más máquina: estado por auto en cola, más eventos por corrida, motor más lento.
  - **Problema fino de formalismo:** cuando un auto empieza a cargar hay que *cancelar* su
    vencimiento, y la metodología evento a evento de la cátedra no tiene cancelación de eventos.
    Dos salidas, ambas defendibles, pero hay que elegir una y dejarla escrita:
    1. **Recalcular** `TPAB(i) = min` sobre la cola cada vez que la cola cambia (TEF limpia, un poco
       más de cómputo). **Preferida.**
    2. Dejar el evento agendado y, al dispararse, verificar si el auto sigue en cola ("evento
       fantasma"). Más rápido, pero mete eventos que no son eventos.
  - Sin balking, todos entran: el 31 % de EAFO que se va sin siquiera hacer cola no queda
    representado (se representa como abandono instantáneo, `TMEU = 0`, lo cual es aceptable pero hay
    que decirlo).

---

### E. Híbrido: balking + reneging, con y sin información — *objetivo final*

**Idea.** Las dos decisiones con una sola paciencia, que es lo que hace IDEAS:

1. **Al llegar:** estima `W_est` y hace balking si `W_est > TMEU`.
2. **Si entra:** conserva la paciencia y abandona si la espera real supera `TMEU`.

Y arriba de eso, una variable de control nueva `IEC` (Información de Espera al Cliente):

- `IEC = 0`: el usuario estima a ojo con la cola visible (`W_est` como en C, sesgado).
- `IEC = 1`: la estación publica la espera real (app/cartel), `W_est` ≈ espera verdadera.

- **Pros:**
  - El más realista y el que mejor cierra con la bibliografía.
  - **Regala un cuarto eje experimental que no cuesta plata:** informar la espera es una política
    operativa gratis frente a construir un cargador (`CPN`) o una estación (`CEN`). Si reproducimos
    aunque sea parcialmente el resultado de IDEAS (hasta 94 % menos reneging), es la conclusión más
    interesante que puede tener el TP: *hay una palanca de eficiencia que no es capital*.
  - Permite descomponer `PARR` en `PARRB` (balking) y `PARRR` (reneging), que es información útil
    para la decisión de expansión: el balking se cura con capacidad, parte del reneging se cura con
    información.
- **Contras:**
  - Riesgo real de **doble conteo** de la impaciencia si no se cuida que la paciencia sea *la misma*
    variable en las dos decisiones.
  - Más parámetros, más difícil de explicar en el informe, más superficie donde el motor puede estar
    mal sin tirar excepción.
  - `PARR` deja de ser un número y pasa a ser dos; hay que rehacer la definición de la variable de
    resultado en la propuesta.

---

### F. Población heterogénea de perfiles (se monta sobre C, D o E)

**Idea.** En vez de una única `TMEU`, una mezcla de perfiles, como los tres de IDEAS
(pesimista/estándar/optimista) o los cuatro de Alsabbagh (NTAD/LTAD/MTAD/HTAD). Cada perfil tiene su
paciencia **y su sesgo de percepción**: el optimista subestima la espera y entra, el pesimista la
sobreestima y se va.

Del paper de "time anxiety" se puede tomar directo la forma funcional de cómo crece la impaciencia
con el tiempo normalizado de espera `B ∈ [0,1]`: **logarítmica** (aguanta bien, se impacienta al
final), **lineal** (proporcional) y **exponencial** (aguanta poco, se va temprano).

- **Pros:** captura que el 31 % de EAFO es un segmento y no una cola; permite decir con números
  *"este pedazo de la demanda se pierde igual, no lo compres con `CPN`"*, que es un insumo directo
  para el análisis económico; barato de agregar si ya está C o D.
- **Contras:** la mezcla de perfiles no la tenemos medida para CABA (la inventamos); sobreparametriza
  un modelo que ya tiene tres variables de control; puede volver el informe ilegible.
- **Cuándo conviene:** al final, si sobra tiempo, y sólo con dos o tres perfiles.

---

### Descartada: arrepentimiento con reintento en otra estación

El auto que se arrepiente en `i` busca la estación `j`. Es más realista en una red, pero **contradice
la decisión ya tomada** de que cada estación tiene su propio flujo de arribos y que `PDCE` modela la
competencia. Implementarlo obliga a un modelo espacial y a reescribir media propuesta. Lo dejamos
anotado como limitación en la discusión del informe, no como modelo.

---

## 6. Tabla comparativa

| Criterio | A | B | C | D | E | F |
|---|---|---|---|---|---|---|
| Mecanismo | balking | balking | balking | reneging | ambos | modificador |
| Respaldo empírico | ninguno | ninguno | **EAFO** | **EAFO** | EAFO + IDEAS | IDEAS + TII |
| Reacciona a fallas / `TMP` | no | no | sí | **sí (real)** | **sí** | — |
| Escala al crecer `CC(i)` | con cuidado | sí | sí | sí | sí | — |
| Agrega eventos / TEF | no | no | no | **sí** | **sí** | no |
| Permite validar vs. Erlang-A | no | no | no | **sí** | sí | no |
| Costo de implementación | bajo | bajo | medio | medio-alto | alto | bajo |
| Riesgo de motor mal sin darnos cuenta | bajo | bajo | medio | medio | **alto** | medio |

---

## 7. Recomendación y plan por etapas

Cada etapa deja algo presentable; si nos quedamos sin tiempo, cortamos donde estemos.

1. **Etapa 0 — Baseline (A).** Portar la regla del TP 4 con el umbral corregido sobre `q(i)`. Sirve
   para el "antes y después" del informe. *Verificable:* `PARR` con `CD = 4` en el orden del 53–60 %
   del TP 4.
2. **Etapa 1 — Reneging con paciencia exponencial (D).** Implementar el evento de abandono y la
   entrada `TPAB(i)` en la TEF. *Verificable:* la corrida degenerada reproduce **Erlang-A** dentro de
   la tolerancia fijada de antemano. **Esta es la etapa que más valor agrega**, porque es la única
   que nos da una vara externa para saber si el motor está bien.
3. **Etapa 2 — Paciencia empírica (C sobre D).** Cambiar la exponencial por la distribución de EAFO y
   agregar el balking por `W_est`. *Verificable:* con capacidad sobrada, `PARR → ~31 %` (el piso de
   EAFO) y no a 0.
4. **Etapa 3 — Información (E).** Agregar `IEC` como cuarta variable de control y correr el escenario
   con y sin información. *Verificable:* `PARRR` cae con `IEC = 1` y `PARRB` no empeora.
5. **Etapa 4 — Perfiles (F).** Sólo si sobra tiempo.

---

## 8. Decisiones que tenemos que cerrar entre los cuatro

1. **¿Qué versión del TP 4 es la válida?** ¿La del paper (93 % con 1 auto) o la del código (93 % con
   2)? Sin esto, cualquier comparación con el TP 4 es inválida.
2. **Denominador de `PARR`.** ¿Arrepentidos sobre los arribos que pasaron el filtro `PDCE`, o sobre
   todos los arribos generados? Propuesta: **sobre los que pasaron `PDCE`**, porque los otros nunca
   consideraron la estación. Hay que dejarlo escrito en la propuesta.
3. **`PARR` por franja horaria.** Ya está definida como variable de resultado por franja: hay que
   decidir si un auto que llega en la franja 2 y abandona en la 3 cuenta en la franja de llegada o en
   la de abandono. Propuesta: **franja de llegada**.
4. **Falla del cargador mientras un auto está cargando** (pendiente ya listado en `CLAUDE.md`). Con
   reneging la respuesta natural es: **vuelve a la cabecera de la cola con su paciencia remanente**;
   si se le vence, cuenta como arrepentido. La opción "se pierde" es el caso particular de paciencia
   remanente igual a cero. Sin modelo de paciencia esta decisión es arbitraria; con D o E sale sola.
5. **Tope del tramo "más de 1 hora"** de EAFO: hay que elegir un valor (¿2 h? ¿3 h?) y justificarlo.
6. **Cancelación de eventos** (si vamos a D o E): recálculo del mínimo vs. evento fantasma.
   Propuesta: **recálculo**.
7. **Verificar la Figura 1 del TP 4 contra el paper IDEAS.** El 93 % lo tomamos del valor de
   convergencia de la curva `ObservationFC` con λ = 0,6. Conviene que alguien lo confirme leyendo el
   paper, porque de ahí sale todo el baseline.

---

## 9. Qué hay que tocar en la propuesta

Si adoptamos **D** o **E**, estos son los cambios mínimos a
[Propuesta_TP-FINAL.md](Propuesta_TP-FINAL.md):

**Datos (FDP nueva)**

| Sigla | Significado |
|---|---|
| `TMEU` | Tiempo Máximo de Espera tolerado por el Usuario (min) |

**Control (sólo si vamos a E)**

| Sigla | Significado |
|---|---|
| `IEC` | Información de Espera al Cliente (0 = no se informa, 1 = se informa la espera real) |

**Resultado (desdoblar `PARR`)**

| Sigla | Significado |
|---|---|
| `PARRB` | Porcentaje de arrepentimiento por no ingresar (balking) |
| `PARRR` | Porcentaje de arrepentimiento por abandono de cola (reneging) |

**Estado / auxiliares**

| Sigla | Significado |
|---|---|
| `TAB(i)(k)` | Instante de abandono del k-ésimo auto en la cola de la estación (i) |

**TEF**

| Sigla | Significado |
|---|---|
| `TPAB(i)` | Tiempo de Próximo ABandono en la estación (i) = `min_k TAB(i)(k)` |

**Tabla de eventos — fila nueva**

| Evento | E. F. no C. | E. F. C. | Condición |
|---|---|---|---|
| Ingreso de auto a una estación (i) | Ingreso (i) | Arrepentimiento en cola (i) | `R ≤ PDCE/100 && CA(i) ≥ CD(i) && TMEU > 0` |
| Arrepentimiento de auto en cola (i) | – | – | – |

---

## 10. Fuentes

1. **EAFO Consumer Monitor 2023 — European Aggregated Report.** Vanhaverbeke, Verbist, Barrera
   (VUB-MOBI) y Csukas (FIER), Comisión Europea, DG MOVE, junio 2024. doi:10.2832/062076.
   Sección 3.4 y figura 10: esperas tolerada y declarada en puntos de carga públicos.
2. **IDEAS: Information-Driven EV Admission in Charging Station Considering User Impatience to
   Improve QoS and Station Utilization.** A. Chattopadhyay, S. Kar, IIT Delhi, arXiv:2403.06223
   (marzo 2024). Balking y reneging, perfiles de optimismo, efecto de informar la espera.
3. **Distributed Electric Vehicles Charging Management Considering Time Anxiety and Customer
   Behaviors.** A. Alsabbagh, B. Wu, C. Ma, *IEEE Transactions on Industrial Informatics*, 2020.
   Concepto de *time anxiety*, cuatro perfiles de conductor y sus formas funcionales (ec. 11–13).
4. **Paper del TP 4** (Carlana Rivero, Loglen, Millán, Ojeda Cabrera, UTN-FRBA). §2.1: regla de
   arrepentimiento vigente; §3: resultados de referencia.

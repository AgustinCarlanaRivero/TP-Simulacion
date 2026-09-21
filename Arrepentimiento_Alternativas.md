# Cómo modelar el arrepentimiento en el TP final — alternativas

> **Estado:** documento de trabajo para discutir entre los cuatro. No es una decisión tomada.
> Lo que elijamos hay que bajarlo a [Propuesta_TP-FINAL.md](Propuesta_TP-FINAL.md): siglas nuevas,
> tabla de eventos y TEF (ver [§11](#11-qué-hay-que-tocar-en-la-propuesta)).
>
> **Actualización — alineado con la revisión de la calibración.** Este documento se ajustó a
> [Calibracion_Arrepentimiento_Revision.md](Calibracion_Arrepentimiento_Revision.md). La decisión del
> conductor se evalúa contra la longitud de la cola, así que cae el eje `IEC` de la alternativa E; el
> mínimo `min(FI·TC, P_EAFO)` de la alternativa G queda corregido; BC Hydro pasa a ser la fuente de
> paciencia absoluta y EAFO queda como validación externa, y se suma la alternativa **I**, que es la
> calibración corregida. El anclaje de `FI` a esas dos fuentes también se corrigió: hay que tomar la
> media entre los que esperan, no la media total, porque el segmento que no espera ya lo modela `PNE`
> y contarlo dos veces achica la paciencia por debajo de lo que dicen las encuestas.

---

## 1. Resumen

El arrepentimiento es la variable que más pesa en el TP final y hoy es la peor fundamentada del
modelo: la propuesta **no la define en ninguna parte** (aparece `CARRUM(i)` en la condición de
expansión, pero nunca se dice cómo se genera un arrepentido), y lo único que tenemos es la regla
hardcodeada del TP 4, que además **no se puede portar tal cual** al esquema de fila única — y que,
como se ve en [§3.3](#33-el-93--del-tp-4-es-una-lectura-equivocada-del-paper), está apoyada en una
lectura equivocada de la fuente.

Nueve alternativas sobre la mesa, ordenadas de menos a más ambiciosa:

| # | Alternativa | Mecanismo | Esfuerzo | Veredicto |
|---|---|---|---|---|
| **A** | Umbral por longitud de cola (TP 4 corregido) | Balking voluntario | Bajo | Sirve sólo como **baseline** para comparar |
| **B** | Probabilidad continua en la longitud de cola | Balking voluntario | Bajo | Mejor que A, pero sin anclaje empírico |
| **H** | Cola finita: se llena y no entra nadie más | Balking **forzado** | Muy bajo | **Hacerlo sí o sí**, es independiente del resto |
| **C** | Paciencia empírica vs. espera estimada | Balking voluntario | Medio | Es el mecanismo del núcleo; con paciencia absoluta degenera |
| **G** | Paciencia proporcional al tiempo de carga propio | Paciencia | Bajo | Da la escala del núcleo; sola, sobrestima la paciencia |
| **I** | Paciencia relativa con piso de no-espera (calibración corregida) | Balking voluntario | Medio | **Recomendada** como núcleo: C con la escala de G |
| **D** | Paciencia como reloj: abandono en cola | Reneging | Medio-alto | **Recomendada**; habilita validar contra Erlang-A |
| **E** | Híbrido H + I + D: las tres decisiones con una sola paciencia | Todos | Alto | **Objetivo final** |
| **F** | Población heterogénea de perfiles | Se monta sobre C/D/E | Medio | Opcional; con fórmulas tomadas de la bibliografía |

**Recomendación:** ir a **E**, construido por etapas (**H → A → D → I**; con I montada sobre las
anteriores, el modelo ya es E), porque cada etapa deja
un resultado presentable aunque nos quedemos sin tiempo. Detalle en
[§9](#9-recomendación-y-plan-por-etapas).

---

## 2. Los tres arrepentimientos

En teoría de colas no hay uno sino tres fenómenos, y el TP 4 los mezcló en uno solo. IDEAS los separa
explícitamente y conviene que nosotros también:

- **Balking forzado (no hay lugar):** el auto llega, la capacidad de espera está llena y no puede
  entrar aunque quiera. No es una decisión del usuario: es una restricción física de la estación.
- **Balking voluntario (no vale la pena):** hay lugar, pero el auto ve la cola, estima la espera y
  decide no entrar. **No ocupa cola, no espera, no consume capacidad.** Es lo único que modela el TP 4.
- **Reneging (abandono):** el auto entra, hace cola, espera, y en algún momento se va sin cargar.
  **Ocupó lugar durante todo ese tiempo** y degradó la espera de los que estaban atrás.

La diferencia no es académica: económicamente el cliente perdido es el mismo, pero **el impacto sobre
el sistema es opuesto**. El que hace balking descomprime; el que hace reneging congestiona y después
se va igual. Como nuestra condición de expansión se dispara con `CARRUM(i)`, la elección entre uno y
otro cambia la política de expansión que va a salir del modelo, no sólo el valor de `PARR`.

Además hay un cuarto "no ingreso" que ya está en la propuesta y **no** es arrepentimiento: el filtro
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

**(a) El código no hace lo que dice el paper.** Las dos versiones de la regla se evalúan sobre los
autos **ya presentes** en el puesto — `NS[i]` incluye al que está cargando, igual que el `CA(i)` del
TP final — pero los umbrales no coinciden:

| Autos ya presentes en el puesto | Paper | Código |
|---|---|---|
| 0 | ingresa | ingresa |
| 1 | **93 % abandona** | **ingresa** |
| 2 | 100 % abandona | 93 % abandona |
| ≥ 3 | 100 % abandona | 100 % abandona |

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
truncar — un solo auto adelante ya son **dos horas** de espera. Un umbral fijo sobre la cola **no
reacciona a las fallas ni al mantenimiento**: si se cae un cargador, la espera se duplica y el umbral
no se entera. Como `TMP` es una de nuestras tres variables de control, esto es grave:
con un umbral fijo sobre la cola, **`TMP` casi no puede mover `PARR`**, y el análisis de
sensibilidad de mantenimiento se queda sin efecto que medir.

El reproche es al **umbral fijo** sobre `q(i)`, no a decidir por longitud de cola. Una regla que
traduce la cola a minutos con `CD(i)` adentro —`W(q) = (q(i) + 1) / CD(i) · E[TC]`, alternativas C e
I— sí reacciona: cuando se cae un cargador, la misma cola da más minutos y `PARR` sube sin recalibrar
nada. Y que la regla no distinga entre esperar detrás de un auto que arranca y uno que está por
terminar **no es un defecto**: el conductor tampoco lo ve desde el auto, así que el modelo no tiene
que verlo
([revisión §6](Calibracion_Arrepentimiento_Revision.md#6-la-premisa-corregida-el-que-no-ve-el-tiempo-es-el-conductor)).

### 3.3. El 93 % del TP 4 es una lectura equivocada del paper

Es el problema de fondo de la regla vigente, y es más grave que los tres anteriores.

En el TP 4 dijimos que tomamos *"el valor de convergencia de la curva ObservationFC con λ = 0,6"*.
Ese valor existe y es **93,14 %** (Tabla II del paper). Pero **no es lo que creímos que era**:

| Lo que dice IDEAS | Lo que escribimos en el TP 4 |
|---|---|
| 93,14 % es el **porcentaje del total de arribos que hacen balking** a lo largo de toda la corrida | Lo usamos como **probabilidad condicional de abandonar dado que hay 1 auto en el puesto** |
| Es una estación de **un solo cargador** con **cola finita** | Lo aplicamos a puestos con cola propia y después a fila única |
| Está **dominado por el balking forzado** (la cola llena), no por la decisión del usuario — el paper lo dice explícitamente | Lo interpretamos como decisión voluntaria del usuario |
| El escenario es de saturación extrema: con λ = 0,6 arribos/min y ≈ 56 min de carga, ρ ≈ 34. Incluso el caso "hora valle" (λ = 0,1) tiene ρ ≈ 5,6 | Lo tomamos como un parámetro de comportamiento general |

Son dos objetos distintos. **El 93 % no tiene respaldo empírico en el uso que le dimos**, y el
53–60 % de `PARR` del TP 4 no es comparable con el 93 % de IDEAS.

Consecuencia práctica: de IDEAS podemos tomar prestadas **las direcciones de los efectos y las
fórmulas de comportamiento**, pero **no los niveles** (93 %, 3,9 %, etc.), porque salen de una
estación mucho más saturada que la nuestra.

---

## 4. Qué aportan las fuentes

| Fuente | Qué nos da | Cómo se usa |
|---|---|---|
| **EAFO Consumer Monitor 2023** (Comisión Europea) | Distribución empírica de espera tolerada por conductores de BEV | **Validación externa** y extremo del barrido de I: su media entre los que esperan ancla `FI = 0,34` y su 31 % es el techo del piso de no-espera |
| **BC Hydro, informe del primer año** (2025) | Espera máxima que el usuario **está dispuesto** a tolerar, zona urbana y no urbana | **Fuente de paciencia absoluta**; por ser 85 % carga rápida, es cota impaciente: ancla `FI = 0,16` (media entre los que esperan) y el piso de 12–17 % |
| **Hanni, Yamamoto & Nakamura** (*Sustainability*, 2024) | Espera aceptable **con el cargador ocupado al llegar**, separada en carga normal y rápida | Fija la **dirección y la forma** de la paciencia, no sus niveles |
| **IDEAS** (Chattopadhyay & Kar, arXiv 2403.06223) | Balking forzado / voluntario / reneging; **paciencia proporcional al tiempo de carga propio** (`z = 0,6`); estimadores de espera por perfil; efecto de informar la espera | Núcleo del modelo de comportamiento y escalera de escenarios |
| **Alsabbagh, Wu & Ma** (IEEE TII, 2020) | *Time anxiety*: la impaciencia crece con el tiempo transcurrido; cuatro perfiles y tres formas funcionales (log, lineal, exponencial) | Forma de la curva de impaciencia y mezcla de perfiles |
| **ACN-Data** (vía IDEAS, figs. 3 y 4) | Duración de carga más frecuente **entre 100 y 200 min**; horas activas 07–23 con picos **09–10 y 14–19** | **Corrobora dos supuestos nuestros**: el `TC` medio ≈ 119 min y el corte de franjas 08–13 / 13–20 / 20–08 |
| **Paper del TP 4** | La regla del 93 % y los resultados de referencia | Baseline de comparación |

### 4.1. El dato de EAFO: paciencia en valor absoluto

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
   distribución **subestima** la paciencia real. Es un supuesto a declarar, no a esconder. BC Hydro
   pregunta directamente cuánto se toleraría y corrige esta debilidad
   ([§4.4](#44-las-fuentes-nuevas-bc-hydro-y-hanni-et-al)): por eso EAFO pasa a ser validación
   externa y no insumo.
3. **Es Europa, no CABA.** Mismo problema que ya asumimos con el dataset de California. El rango
   entre países del "15 minutos o menos" va de **10 % (Lituania) a 42 % (Francia)**: sirve como
   rango de sensibilidad razonable en vez de inventar uno.

### 4.2. El dato de IDEAS: paciencia relativa a la propia necesidad

IDEAS **no** define la paciencia en minutos absolutos. La define como una fracción del tiempo que el
propio usuario necesita cargar:

```
T_Impaciencia(i) = z · T(i, SoC_llegada, 80 %)          con z = 0,6
```

`z` es el **Factor de Impaciencia**: la fracción del tiempo de carga que el usuario está dispuesto a
esperar antes de impacientarse. La idea de fondo es sólida y muy portable: **el que necesita cargar
mucho tolera esperar más**; el que venía a enchufar diez minutos no hace cola.

Y el usuario abandona cuando el tiempo ya esperado supera ese umbral (ec. 13 del paper):

```
[T - T_ingreso(i)] > T_Impaciencia(i)   ->   se va sin cargar
```

**Los dos datos se contradicen, y por mucho.** Con nuestro `TC` (media ≈ 119 min, mediana ≈ 107 min),
`z · TC` da paciencias de **~64 a ~71 min**, contra los **~15 min** que sugiere la mediana de EAFO.
Un factor 4. No es un detalle: con los cuatro cargadores iniciales ocupados y la cola vacía, las
tablas absolutas dan un `PARR` de 81 % (EAFO) a 95 % (BC Hydro), y `FI·TC` da 16 %
([revisión §3.1](Calibracion_Arrepentimiento_Revision.md#31-los-números-con-nuestro-tc-no-con-el-del-ejemplo)).
Hanni et al. (2024) muestra que ninguno de los dos extremos aguanta
([§4.4](#44-las-fuentes-nuevas-bc-hydro-y-hanni-et-al)). La propuesta para resolverlo está en
[§5.I](#i-paciencia-relativa-con-piso-de-no-espera-calibración-corregida--recomendada-como-núcleo).

### 4.3. Los resultados de IDEAS (Tabla II) y sus tres contra-intuiciones

| λ | ¿Informa la espera? | Balking % | Reneging % | Servidos % (sobre los que hicieron cola) |
|---|---|---|---|---|
| 0,1 | No | 94,01 | 3,67 | 33,06 |
| 0,1 | **Sí** | 97,33 | **0,31** | **77,19** |
| 0,6 | No | 93,14 | 3,94 | 37,05 |
| 0,6 | **Sí** | 97,09 | **0,35** | **74,57** |

Tres resultados que valen para nosotros aunque los niveles no sean transferibles:

1. **Informar la espera aumenta el balking y casi elimina el reneging.** Al informar, el balking sube
   (93,1 → 97,1 %) pero el reneging cae **–91 %** (3,94 → 0,35 %) y **el porcentaje de servidos se
   duplica** (37,1 → 74,6 %). La información no retiene más clientes: **los ordena**. El que iba a
   ocupar un lugar para irse igual, ahora directamente no entra, y el lugar queda para alguien que sí
   va a cargar. En nuestro modelo este escenario no se corre: el conductor decide con lo que ve desde
   el auto, y la espera informada queda como limitación declarada
   ([§6](#6-aparte-dos-palancas-que-quedan-fuera-de-alcance)).
2. **Menos balking no es mejor.** El caso sin decisión del usuario (`BlockingFC`) tiene menos balking
   pero más reneging y menos servidos. Textual: *"los escenarios con menor porcentaje de balking no
   necesariamente aseguran mayor porcentaje de tráfico servido"*. Bajar `PARR` no puede ser el
   objetivo del TP por sí solo.
3. **El throughput es la métrica equivocada bajo impaciencia.** Como los usuarios de carga corta son
   los menos pacientes, la cola se auto-selecciona hacia usuarios de carga larga: el throughput puede
   verse bien mientras el servicio es malo. Esto tiene consecuencias directas sobre nuestro `BM` y
   nuestro `ECP` — ver [§7](#7-efectos-de-segundo-orden-que-hay-que-anticipar).

### 4.4. Las fuentes nuevas: BC Hydro y Hanni et al.

**BC Hydro** pregunta lo que el modelo necesita —*cuánto estás dispuesto a esperar*—, no cuánto
esperaste, así que no arrastra la censura de EAFO. Tabla urbana de la encuesta de noviembre de 2024
(pregunta 11d), con los tramos corregidos contra el informe
([revisión §11](Calibracion_Arrepentimiento_Revision.md#11-sobre-las-fuentes)):

| Máximo que está dispuesto a esperar | % |
|---|---:|
| Nada | 17 % |
| 5 min | 29 % |
| 10 min | 30 % |
| 15 min | 1 % |
| 20–30 min | 18 % |
| Más de 30 min | 5 % |

La media es 16,0 min, y en zona no urbana el piso baja a 12 %. La salvedad es la tecnología: 499 de
sus 591 puertos son de carga rápida, así que es paciencia medida frente a sesiones cortas. Sirve como
**cota impaciente**, no como valor central para un dataset de carga de destino como el nuestro.

**Hanni et al. (2024)** es la fuente mejor alineada con nuestro punto de decisión: pregunta por la
espera tolerable *cuando el cargador está ocupado al llegar*, y separa carga normal de rápida. Tres
resultados:

1. **Con carga normal la paciencia sube, pero poco**: del orden de ×1,6 en odds, no un factor de
   cuatro. Ninguno de los dos extremos aguanta: la tabla absoluta de carga rápida subestima la
   paciencia en un contexto de carga de destino, y `FI·TC` la sobrestima.
2. **La moda es "no esperar nada"** en todas las ubicaciones y para los dos tipos de carga: el
   segmento que no espera existe también en carga normal.
3. **La tolerancia declarada queda por debajo de la espera que efectivamente aguantaron**, así que las
   tres tablas empujan `PARR` hacia arriba por construcción. Es un argumento más para tratar el piso
   como rango y no como punto.

No publica las proporciones por tramo —están sólo en un gráfico—, así que fija dirección y forma, no
números. El detalle está en la
[revisión §5.5](Calibracion_Arrepentimiento_Revision.md#55-hanni-et-al-2024-no-es-una-fuente-complementaria).

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
  no reacciona a `TMP` ni a las fallas; y el 93 % que parecía respaldarlo no lo respalda
  ([§3.3](#33-el-93--del-tp-4-es-una-lectura-equivocada-del-paper)).
- **Cuándo conviene:** como **baseline** contra el que mostrar que el modelo nuevo aporta algo. Vale
  la pena dejarlo implementado aunque adoptemos otro.

---

### B. Probabilidad continua en la longitud de cola

**Idea.** Reemplazar el escalón por una función suave, que es la forma estándar de balking voluntario
en la literatura de colas, y es la que usa IDEAS (ec. 4, tomada de Zhang et al. 2020), con un
parámetro `σ` que gradúa la sensibilidad a la longitud de la cola:

```
P(arrepentirse | q) = 1 - e^(-BETA * q / CD(i))
```

> **Ojo con la fórmula del paper.** IDEAS la escribe como `P_VB = e^(-(1-w)σ)` con `w ≥ 1`, que da
> `e^((w-1)σ) ≥ 1`: no está acotada a [0,1] y para cola > 1 da probabilidad mayor que uno. Es una
> errata. La lectura sensata es la de arriba (`1 - e^(-σ(w-1))`) o `e^(-σ(k-w))`. Si citamos la
> fórmula en el informe, hay que citarla corregida y decir por qué.

- **Datos que necesita:** un solo parámetro `BETA`, calibrable para que `PARR` a `CD = 4` caiga cerca
  del 53–60 % del TP 4 (así queda anclado a algo, aunque sea a nuestro propio resultado anterior).
- **Cambios en la propuesta:** ninguno estructural.
- **Pros:** un parámetro, sin discontinuidades; escala solo con `CD(i)`; el barrido de `BETA` da una
  curva de sensibilidad limpia; y tiene cita bibliográfica.
- **Contras:** `BETA` no sale de ningún dato; sigue ignorando el tiempo y por lo tanto las fallas y
  `TMP`. Es "A pero prolijo", no un modelo mejor. Además vale 0 en `q = 0`: con todos los cargadores
  ocupados y la cola vacía —el estado congestionado más frecuente— no se arrepiente nadie, y eso choca
  con el piso empírico de no-espera. El agujero es de la forma funcional misma (`b_0 = e^0 = 1` en
  Zhang et al. 2025), no de cómo se la adapte
  ([revisión §4.4](Calibracion_Arrepentimiento_Revision.md#44-el-piso-en-q--0-no-es-un-caso-de-borde)).

---

### H. Cola finita: capacidad física de espera (balking forzado)

**Idea.** La estación tiene lugar para `CEM(i)` autos esperando. Si la cola está llena, el que llega
no entra, decida lo que decida:

```
si  q(i) >= CEM(i)   ->   balking forzado (no ingresa, no hace cola)
```

Es el mecanismo que **domina los resultados de IDEAS** y hoy nuestro modelo no lo tiene: tal como
está la propuesta, la cola de una estación puede crecer sin límite, cosa que en una playa de
estacionamiento no pasa.

- **Datos que necesita:** un valor de `CEM(i)`. Se puede atar a `CC(i)` (p. ej. 2 lugares de espera
  por cargador) para que escale con la expansión, que es coherente con `CC_MAX` como límite de
  espacio físico.
- **Cambios en la propuesta:** una constante y una condición más en el evento de ingreso. **No agrega
  eventos ni TEF.**
- **Pros:**
  - Es el más barato de todos y es **ortogonal a las demás alternativas**: se puede combinar con
    cualquiera.
  - Acota la cola, con lo cual el motor no puede divergir en los escenarios saturados (que van a
    aparecer sí o sí con el crecimiento logístico).
  - Es **físicamente honesto** y engancha con el argumento de espacio del TP: no tiene sentido
    modelar `CC_MAX` por espacio físico y a la vez suponer cola infinita.
  - Separa `PARR` en una parte que la expansión **sí** puede curar (forzado: falta lugar) y otra que
    no (voluntario: no vale la pena esperar). Eso es información directa para la decisión de expansión.
- **Contras:**
  - Un parámetro más sin dato local (aunque acotado por sentido común: una playa no tiene 30 lugares
    de espera).
  - Si `CEM(i)` queda chico, **domina todo lo demás** y el modelo de paciencia deja de verse. Hay que
    verificar que no tape a los otros mecanismos antes de sacar conclusiones.

---

### C. Paciencia empírica vs. espera estimada

**Idea.** Cada auto que llega sortea su tolerancia `TMEU`, estima cuánto va a esperar, y se arrepiente
si la espera estimada supera su tolerancia.

```
TMEU     ~ F_paciencia          # 17 % en 0; el resto por tramos (BC Hydro, §4.4)
W_est(i)  = (q(i) + 1) / CD(i) * E[TC]
se arrepiente  <=>  W_est(i) > TMEU
```

**El estimador conviene tomarlo de IDEAS.** El paper no usa `E[TC]` (la media poblacional): el
usuario **proyecta su propia necesidad sobre los que tiene adelante**. Es más plausible y para
nosotros es gratis, porque ya generamos `TC` por auto:

```
W_est(i) = f * TC_propio * N / CD(i)
```

donde `N` es la cantidad de autos que el usuario "ve" y `f` un factor de perfil (ver
[alternativa F](#f-población-heterogénea-de-perfiles-se-monta-sobre-c-d-o-e)).

> **Consecuencia de implementación:** si la paciencia o la estimación dependen de `TC`, hay que
> **generar `TC` en el evento de arribo y no al empezar a cargar**, como hace hoy el motor del TP 4.
> Es un cambio chico pero hay que hacerlo antes, no después.

- **Datos que necesita:** la tabla de BC Hydro con los tramos corregidos
  ([§4.4](#44-las-fuentes-nuevas-bc-hydro-y-hanni-et-al)) + una decisión sobre cómo repartir dentro
  de cada tramo (uniforme es lo más honesto) y un tope para el "más de 30 min". EAFO queda como
  validación externa.
- **Cambios en la propuesta:** una FDP nueva (`TMEU`) en Datos. **No agrega eventos ni TEF.**
- **Pros:**
  - Se apoya en un dato empírico publicado y citable.
  - Un solo mecanismo produce el piso de no-espera (17 % en BC Hydro, 31 % en EAFO) *y* la
    sensibilidad a la congestión.
  - **Reacciona a la capacidad y a las fallas**: si cae un cargador, `CD(i)` baja, `W_est` sube y
    `PARR` sube. Recién acá `TMP` tiene un canal por el que afectar el arrepentimiento.
  - `PARR` queda en las mismas unidades que `PEC`, que es lo que el informe compara.
- **Contras:**
  - Hay que asumir cómo estima el usuario la espera. `W_est` ignora el tiempo residual del que ya
    está cargando y sobreestima la espera: con nuestra Gumbel, `W(0)` da 30,4 min contra ≈ 21 min
    descontando el remanente de equilibrio. **No es un defecto a corregir**: el conductor tampoco ve
    el remanente, así que ese sesgo es conducta modelada, y la versión corregida queda como cota de
    sensibilidad. En su forma con `E[TC]`, además, `W_est` es la espera condicional exacta de una
    M/M/c, y conviene citarla así
    ([revisión §5.3](Calibracion_Arrepentimiento_Revision.md#53-wq-tiene-mejor-pedigrí-del-que-el-documento-le-atribuye)).
  - BC Hydro es 85 % carga rápida: su tabla es una cota impaciente, no un valor central para carga
    de destino ([§4.4](#44-las-fuentes-nuevas-bc-hydro-y-hanni-et-al)).
  - Sigue siendo **sólo balking**: nadie abandona después de haber esperado, y por lo tanto la cola
    nunca se descomprime sola.
  - Con `TC` medio ≈ 121 min y paciencias absolutas de 15–30 min, el modelo degenera: con los cuatro
    cargadores ocupados y **la cola vacía**, `PARR` ya da 95 % con BC Hydro y 81 % con EAFO. La
    estación pasa a comportarse como un sistema de pérdida, `CARRUM(i)` dispara la expansión todos
    los meses y `TMP` se queda sin efecto que mover
    ([revisión §3](Calibracion_Arrepentimiento_Revision.md#3-el-problema-de-fondo-la-escala)). Es el
    problema que corrige la alternativa
    [I](#i-paciencia-relativa-con-piso-de-no-espera-calibración-corregida--recomendada-como-núcleo).

---

### G. Paciencia proporcional al tiempo de carga propio

**Idea.** En vez de (o además de) sortear una paciencia absoluta, derivarla de la necesidad del
propio usuario, como hace IDEAS:

```
TMEU_k = FI * TC_k                  con FI = 0,6 (Factor de Impaciencia)
```

**Cómo se combina con la paciencia absoluta** ([§4.2](#42-el-dato-de-ideas-paciencia-relativa-a-la-propia-necesidad)).
El mínimo `min(FI·TC_k, P_k)` no resuelve la contradicción: se queda con la paciencia chica, que es
justamente la que hace degenerar el modelo
([revisión §9](Calibracion_Arrepentimiento_Revision.md#9-qué-proponemos-hacer)). La combinación que sí
la resuelve es la alternativa
[I](#i-paciencia-relativa-con-piso-de-no-espera-calibración-corregida--recomendada-como-núcleo): la
forma relativa como modelo, con las tablas absolutas traducidas a `FI` como extremos del barrido y el
segmento que no espera tratado aparte.

- **Datos que necesita:** ninguno nuevo. `TC` ya lo tenemos ajustado; `FI` es un parámetro con valor
  de referencia publicado (0,6).
- **Cambios en la propuesta:** una constante `FI`. Se monta sobre C, D o E sin cambiar su estructura.
- **Pros:**
  - **Costo cero en datos** y con respaldo bibliográfico.
  - Genera heterogeneidad de paciencia **automáticamente**, porque `TC` ya es aleatorio: no hace
    falta inventar una mezcla de perfiles.
  - Acopla la impaciencia con la energía cargada, que es lo que después factura: es coherente con el
    resto del modelo económico.
  - `FI` es una variable de sensibilidad interpretable ("¿qué pasa si los usuarios se vuelven un 30 %
    más impacientes?").
- **Contras:**
  - `z = 0,6` sale de un solo paper y de otro contexto (autopista en India, cargadores de 50 kW).
  - En IDEAS, `T(i, SoC, 80 %)` es el tiempo hasta el 80 % de carga; nuestro `TC` es la sesión
    completa del dataset. Aplicar `FI` sobre `TC` **sobrestima** la paciencia. Hay que declararlo, o
    corregir con un factor.
  - Hanni et al. (2024) confirma la dirección con datos: con carga normal la paciencia sube, pero
    mucho menos que proporcionalmente al tiempo de carga
    ([§4.4](#44-las-fuentes-nuevas-bc-hydro-y-hanni-et-al)). Con `FI = 0,6`, G sola es la cota
    paciente, no el valor central.
  - Sola, no reproduce el segmento que se va sin esperar (17 % en BC Hydro, 31 % en EAFO): **por eso
    la alternativa I le suma un piso explícito**.

---

### I. Paciencia relativa con piso de no-espera (calibración corregida) — *recomendada como núcleo*

**Origen.** Es lo que queda de [Calibracion_Arrepentimiento_EV.md](Calibracion_Arrepentimiento_EV.md)
después de la [revisión](Calibracion_Arrepentimiento_Revision.md). Aquella proponía la forma
exponencial de B calibrada con la cadena de C —`q → W(q) → P(TMEU < W(q))`, con un ajuste de `α`
encima— y BC Hydro como fuente de paciencia. La revisión saca el ajuste de `α`, porque
`P(TMEU < W(q))` ya es el modelo exacto y comprimirlo en una exponencial sólo agrega error
([revisión §4](Calibracion_Arrepentimiento_Revision.md#4-defectos-concretos-de-la-formulación)), y
resuelve la escala, que con nuestro `TC` era lo que hacía degenerar todo. Lo que queda es C con la
escala de G y el segmento que no espera explícito.

**Idea.** Cada auto trae su `TC` desde el arribo y sortea a qué segmento pertenece:

```
con probabilidad PNE:  TMEU_k = 0                   # no espera nunca
si no:                 TMEU_k = FI * TC_k           # paciencia relativa (G)

W_est(i) = (q(i) + 1) / CD(i) * E[TC]               # lo que el conductor infiere desde el auto
se arrepiente  <=>  CA(i) >= CD(i)  &&  W_est(i) > TMEU_k     # CA(i) sin contar al que llega
```

- **La espera se estima con la longitud de la cola — decisión cerrada.** El conductor ve cuántos
  autos esperan y cuántos cargadores hay en servicio; no ve cuánto le falta a cada carga en curso.
  Los remanentes de la TEF quedan fuera del modelo de decisión aunque el motor los tenga: usarlos
  sería simular otro sistema
  ([revisión §6](Calibracion_Arrepentimiento_Revision.md#6-la-premisa-corregida-el-que-no-ve-el-tiempo-es-el-conductor)).
- **El estimador es el ingenuo**, sin descontar el remanente de la carga en curso, porque es lo que
  hace el conductor. La versión con el remanente de equilibrio de la Gumbel (84,4 min en lugar de
  121,4) queda como cota de sensibilidad.
- **`E[TC]` y no el `TC` propio en `W_est`.** Si las dos puntas usaran `TC_k`, se cancelaría: la regla
  quedaría `(q(i) + 1) / CD(i) > FI`, un umbral fijo —la alternativa A con otro nombre— y se perdería
  la heterogeneidad que aporta G.
- **`FI` se barre entre tres anclajes empíricos**: 0,16 (media de BC Hydro entre los que esperan,
  19,3 min sobre 121,4), 0,34 (media de EAFO entre los que esperan, 41,7 min) y 0,60 (IDEAS). Se
  excluyen los ceros de cada media porque ese segmento ya lo modela `PNE` aparte. Las tablas absolutas
  dejan de ser una alternativa a la forma relativa y pasan a ser los extremos impacientes de su
  barrido, que es la lectura que sostiene Hanni et al.
  ([§4.4](#44-las-fuentes-nuevas-bc-hydro-y-hanni-et-al)). Es un anclaje de **medias**, no una
  equivalencia de distribuciones.
- **`PNE` se barre entre 12 % y 31 %**: BC Hydro no urbano, BC Hydro urbano (17 %) y EAFO. Escrito
  así es un **modelo de mezcla**, que es una construcción estándar, y tapa el agujero que la
  exponencial de B tiene en `q = 0`
  ([revisión §4.4](Calibracion_Arrepentimiento_Revision.md#44-el-piso-en-q--0-no-es-un-caso-de-borde)).
- **`TMEU` se sortea por auto**, no como moneda por arribo: el segmento que no espera queda como
  propiedad del auto, y así se lo puede seguir en los resultados.

- **Datos que necesita:** ninguno nuevo. `TC` ya está ajustado, y los anclajes de `FI` y `PNE` salen
  de BC Hydro (tabla corregida), EAFO e IDEAS.
- **Cambios en la propuesta:** dos constantes (`FI`, `PNE`) y la fila del ingreso desdoblada
  ([revisión §7.7](Calibracion_Arrepentimiento_Revision.md#77-qué-habría-que-tocar-en-la-propuesta)).
  **No agrega eventos ni TEF.**
- **Pros:**
  - Produce un gradiente utilizable. Con `FI = 0,6` y `CD = 4`, de los que no pertenecen al segmento
    que no espera se arrepiente el 16 % con la cola vacía, el 46 % con un auto adelante y el 72 % con
    dos
    ([revisión §3.1](Calibracion_Arrepentimiento_Revision.md#31-los-números-con-nuestro-tc-no-con-el-del-ejemplo)).
    Con las tablas absolutas, en cambio, se va casi todo el mundo desde `q = 0`.
  - Reacciona a `CD(i)`: una falla, un mantenimiento o una expansión mueven `PARR` sin recalibrar
    nada. `TMP` conserva el canal por el que afectar el arrepentimiento.
  - Cada parámetro tiene un rango con respaldo empírico en lugar de un valor inventado.
  - La decisión es instantánea en el arribo: no hay ambigüedad de franja horaria (decisión 4 de
    [§10](#10-decisiones-que-tenemos-que-cerrar-entre-los-cuatro)).
- **Contras:**
  - Ningún punto del barrido es *el* valor: `FI·TC` sobrestima la paciencia y las tablas absolutas,
    medidas frente a carga rápida, la subestiman. El resultado es un rango, y el criterio de éxito no
    es que `PARR` se mueva poco sino que **la conclusión económica no se dé vuelta entre las cotas**.
  - Anclar `FI` por la media no reproduce la forma de las tablas absolutas.
  - Es sólo balking: no descomprime la cola (hace falta H) y no tiene fórmula cerrada contra la cual
    validar. Aunque el modelo final no lleve reneging, D tiene que quedar implementada en modo
    degenerado para contrastar el motor contra Erlang-A
    ([revisión §7.3](Calibracion_Arrepentimiento_Revision.md#73-se-pierde-la-única-validación-externa-disponible)).
  - Como la paciencia correlaciona con `TC`, arrastra el sesgo de selección sobre `ECP` de
    [§7](#7-efectos-de-segundo-orden-que-hay-que-anticipar).

---

### D. Paciencia como reloj: abandono en cola (reneging) — *recomendada*

**Idea.** El auto entra y hace cola, pero con vencimiento: si no empezó a cargar antes de `TMEU`, se va.

```
al ingresar a la cola:  TAB(i)(k) = T + TMEU_k
TEF:                    TPAB(i) = min_k TAB(i)(k)
evento nuevo:           "Arrepentimiento de auto en cola (i)"
```

- **Datos que necesita:** la misma `TMEU` de C, G o I.
- **Cambios en la propuesta:** **una fila nueva en la tabla de eventos y una entrada nueva en la
  TEF.** Además `CA(i)` deja de alcanzar como estado: hay que llevar la cola con los vencimientos.
- **Pros:**
  - Es el modelo canónico de la literatura (**M/M/c+G**), y con paciencia exponencial es **Erlang-A**,
    que tiene fórmula cerrada para probabilidad de abandono y espera media. Eso nos da un **test de
    validación del motor** que hoy no tenemos: corrida degenerada (una estación, sin fallas, sin
    expansión, IA/TC/paciencia exponenciales) contra la fórmula. Es exactamente lo que pide la regla
    de "Verificación del motor" de `CLAUDE.md`, y es el argumento más fuerte a favor de esta opción.
    IDEAS usa la misma familia: tasa de abandono `r_k = (k − c)·θ` en un M/M/c/K, con `θ` el
    tiempo de impaciencia exponencial.
  - **El abandono desde cualquier posición de la cola es un punto que IDEAS defiende
    explícitamente** como mejora sobre la literatura previa, que sólo deja abandonar desde la cabecera.
    Nuestra formulación (vencimiento por auto) ya lo hace: conviene decirlo en el informe, es un punto
    a favor del modelo.
  - `PARR` y `PEC` quedan consistentes: el que abandona **efectivamente esperó**.
  - No choca con la decisión de evaluar contra la longitud de la cola: el conductor no ve el tiempo
    **remanente**, pero el tiempo **transcurrido** sí lo vive, así que el reloj de paciencia es
    legítimo
    ([revisión §6](Calibracion_Arrepentimiento_Revision.md#6-la-premisa-corregida-el-que-no-ve-el-tiempo-es-el-conductor)).
  - Reacciona a fallas, a `TMP` y a la expansión por el canal correcto (el tiempo real de espera).
- **Contras:**
  - Más máquina: estado por auto en cola, más eventos por corrida, motor más lento.
  - **Problema fino de formalismo:** cuando un auto empieza a cargar hay que *cancelar* su
    vencimiento, y la metodología evento a evento de la cátedra no tiene cancelación de eventos.
    Dos salidas, ambas defendibles, pero hay que elegir una y dejarla escrita:
    1. **Recalcular** `TPAB(i) = min` sobre la cola cada vez que la cola cambia (TEF limpia, un poco
       más de cómputo). **Preferida.**
    2. Dejar el evento agendado y, al dispararse, verificar si el auto sigue en cola ("evento
       fantasma"). Más rápido, pero mete eventos que no son eventos.
  - Sin balking, todos entran: el 31 % de EAFO que se va sin siquiera hacer cola se representa como
    abandono instantáneo (`TMEU = 0`), lo cual es aceptable pero hay que decirlo.
  - **No validar contra las ecuaciones (7) y (8) de IDEAS.** Suman una probabilidad (`P_r`) a un
    número de clientes y a un tiempo: no cierran dimensionalmente. La vara correcta es Erlang-A.

---

### E. Híbrido: balking forzado, balking voluntario y abandono con una sola paciencia — *objetivo final*

**Idea.** Las tres decisiones con una sola paciencia, que es lo que hace IDEAS:

1. **Al llegar, ¿hay lugar?** Si `q(i) ≥ CEM(i)` → balking forzado (alternativa H).
2. **Si hay lugar, ¿vale la pena?** Estima `W_est` y hace balking voluntario si `W_est > TMEU`.
3. **Si entra:** conserva la paciencia y abandona si la espera real supera `TMEU`.

Con la paciencia y el estimador de la alternativa I, E es **H + I + D**; sin reneging, se reduce a
H + I. No lleva eje de información al cliente: la decisión se toma contra la longitud de la cola, y
la espera informada queda fuera de alcance ([§6](#6-aparte-dos-palancas-que-quedan-fuera-de-alcance)).

- **Pros:**
  - El más realista y el que mejor cierra con la bibliografía.
  - Permite descomponer `PARR` en `PARRF` (forzado), `PARRB` (voluntario) y `PARRR` (abandono), que
    es información directa para la expansión: el forzado se cura con capacidad, el voluntario sólo
    en parte.
- **Contras:**
  - Riesgo real de **doble conteo** de la impaciencia si no se cuida que la paciencia sea *la misma*
    variable en las tres decisiones.
  - Más parámetros, más difícil de explicar en el informe, más superficie donde el motor puede estar
    mal sin tirar excepción.
  - `PARR` deja de ser un número y pasa a ser tres; hay que rehacer la definición de la variable de
    resultado en la propuesta.
  - Con reneging vuelve la ambigüedad de franja horaria (decisión 4 de
    [§10](#10-decisiones-que-tenemos-que-cerrar-entre-los-cuatro)).

---

### F. Población heterogénea de perfiles (se monta sobre C, D o E)

**Idea.** En vez de una única `TMEU`, una mezcla de perfiles con distinta paciencia **y distinto sesgo
de percepción**: el optimista subestima la espera y entra, el pesimista la sobreestima y se va.

IDEAS da las tres fórmulas ya escritas (sus ec. 10–12), y son simples de portar. Todas usan el
tiempo de carga **del que llega** como proxy del de los que están adelante; lo único que cambia es
qué SoC supone y a cuántos autos mira:

| Perfil | Estimador de IDEAS | Traducción a nuestro modelo |
|---|---|---|
| **Pesimista** | `T(i, 5 %, 80 %) · N_sys(t)` | Supone el peor caso de carga y mira **todos** los autos de la estación |
| **Estándar** | `T(i, SoC, 80 %) · N_cola(t)` | Usa su propio tiempo de carga y mira **sólo la cola** |
| **Optimista** | `T(i, 60 %, 80 %) · N_cola(t)` | Supone cargas cortas y mira sólo la cola |

Es decir, `W_est = f · TC_propio · N / CD(i)` con `f > 1` (pesimista), `f = 1` (estándar) y `f < 1`
(optimista), y `N` = autos en la estación o en la cola según el perfil.

Del paper de *time anxiety* se puede tomar además la forma funcional de cómo crece la impaciencia con
el tiempo normalizado de espera: **logarítmica** (aguanta bien, se impacienta al final), **lineal**
(proporcional) y **exponencial** (aguanta poco, se va temprano).

- **Pros:** captura que el 31 % de EAFO es un segmento y no una cola; permite decir con números
  *"este pedazo de la demanda se pierde igual, no lo compres con `CPN`"*; las fórmulas están
  publicadas y no inventadas, y con `f` como único parámetro por perfil es barato.
- **Contras:** la mezcla de perfiles no la tenemos medida para CABA (la inventamos); sobreparametriza
  un modelo que ya tiene tres variables de control; puede volver el informe ilegible. Además, si ya
  usamos G, buena parte de la heterogeneidad ya la genera la aleatoriedad de `TC`.
- **Cuándo conviene:** al final, si sobra tiempo, y sólo con dos o tres perfiles.

---

### Descartada: arrepentimiento con reintento en otra estación

El auto que se arrepiente en `i` busca la estación `j`. Es más realista en una red, pero **contradice
la decisión ya tomada** de que cada estación tiene su propio flujo de arribos y que `PDCE` modela la
competencia. Implementarlo obliga a un modelo espacial y a reescribir media propuesta. Lo dejamos
anotado como limitación en la discusión del informe, no como modelo.

---

## 6. Aparte: dos palancas que quedan fuera de alcance

Dos políticas operativas que muestra la bibliografía y que el modelo deja afuera. Las dos van a la
discusión del informe como limitación declarada y línea futura.

**Informar la espera.** Una red puede publicar en su app una estimación de la espera, e IDEAS muestra
que eso ordena la demanda: el balking sube, el reneging cae −91 % y el porcentaje de servidos se
duplica ([§4.3](#43-los-resultados-de-ideas-tabla-ii-y-sus-tres-contra-intuiciones)). Nuestro
conductor decide con lo que ve desde el auto
([revisión §6](Calibracion_Arrepentimiento_Revision.md#6-la-premisa-corregida-el-que-no-ve-el-tiempo-es-el-conductor)),
así que modelar esa política queda fuera de alcance. Lo que se deja afuera es la política, no su
existencia: el tiempo exacto no lo puede publicar nadie —depende de los `TC` de los autos que todavía
están en cola—, pero una estimación sí.

**Cargador de dos modos.** IDEAS propone además un **cargador de dos modos y dos bocas**: carga rápida
hasta el 80 % de SoC y, pasado ese punto, la boca conmuta a modo lento y **libera la potencia rápida
para otro auto en la segunda boca**. El fundamento es la curva de carga de litio: pasar de 80 % a
100 % tarda casi lo mismo que llegar de 0 % a 80 %, y los usuarios cargan de más por ansiedad de
autonomía. El resultado que reportan es +5 % de disponibilidad de carga rápida y +14 % de throughput
en hora pico.

Para nosotros esto **no es un modelo de arrepentimiento**, pero es una palanca en el mismo eje
que la información: aumenta la capacidad efectiva **sin pagar `CPN` ni `CEN`**, y por lo tanto compite
directamente con la condición de expansión. Traducido a nuestro modelo sería que un cargador se libera
al llegar al 80 % de la carga, o sea un `TC` efectivo menor al observado.

**Recomendación:** dejarlo fuera del alcance y mencionarlo en la discusión del informe como línea
futura. Meterlo obligaría a partir `TC` en dos tramos (rápido y lento) y a rehacer la regresión
energía–tiempo, que hoy es lineal sobre la sesión completa. Es más trabajo del que parece y no es el
tema del TP.

---

## 7. Efectos de segundo orden que hay que anticipar

Cualquier modelo de paciencia (C, D, E, G, I) mete tres efectos que **no son bugs** pero que van a
aparecer en los resultados y conviene tener explicados de antemano:

1. **Sesgo de selección sobre `ECP`.** Si la paciencia crece con el tiempo de carga (alternativas G e I),
   los autos que se quedan son sistemáticamente los de carga larga. Entonces **`ECP` deja de ser la
   media de la distribución de energía y pasa a ser la media sobre los atendidos, sesgada hacia
   arriba**. Y `ECP` está en las dos condiciones de expansión de la propuesta. Hay que calcularlo
   sobre los atendidos, no sobre la FDP teórica, y decirlo.
2. **El throughput miente.** Como los de carga corta son los que se van, la estación puede atender
   pocos autos y facturar bien, o muchos y facturar mal. La métrica correcta bajo impaciencia es el
   **porcentaje de autos atendidos** y el beneficio, no la cantidad por unidad de tiempo.
3. **`PARR` deja de ser "cuanto más bajo mejor".** Con reneging, menos balking puede significar más
   abandono y menos atendidos: es el caso `BlockingFC` de IDEAS
   ([§4.3](#43-los-resultados-de-ideas-tabla-ii-y-sus-tres-contra-intuiciones)). Si el informe concluye a partir de `PARR` sola, va a concluir mal. La conclusión
   tiene que salir de `BM` + porcentaje de atendidos, con `PARR` desagregado como diagnóstico.

Y un detalle de implementación que hay que resolver antes de escribir el motor:

4. **`TC` se genera en el arribo, no al empezar a cargar.** Si la paciencia (G, I) o la estimación de
   espera (C, F) dependen de `TC`, el auto tiene que traer su `TC` desde que llega. Es un cambio chico
   respecto del motor del TP 4, pero cambia dónde se consume el número aleatorio y por lo tanto los
   resultados: hay que hacerlo antes de fijar la semilla de las corridas del informe.

---

## 8. Tabla comparativa

| Criterio | A | B | H | C | G | I | D | E | F |
|---|---|---|---|---|---|---|---|---|---|
| Mecanismo | balk. vol. | balk. vol. | **balk. forzado** | balk. vol. | paciencia | balk. vol. | reneging | todos | modificador |
| Respaldo empírico | ninguno | fórmula citada | físico | **BC Hydro** (EAFO valida) | **IDEAS (z=0,6)** | **BC Hydro, EAFO, IDEAS, Hanni** | EAFO/IDEAS | todas | IDEAS + TII |
| Reacciona a fallas / `TMP` | no | no | sí | sí | sí | sí | **sí (real)** | **sí** | — |
| Escala al crecer `CC(i)` | con cuidado | sí | sí (vía `CEM`) | sí | sí | sí | sí | sí | — |
| Agrega eventos / TEF | no | no | no | no | no | no | **sí** | **sí** | no |
| Permite validar vs. Erlang-A | no | no | no | no | no | no | **sí** | sí | no |
| Datos nuevos que hace falta conseguir | — | — | — | tabla BC Hydro | — | — | — | — | mezcla |
| Costo de implementación | bajo | bajo | **muy bajo** | medio | **bajo** | medio | medio-alto | alto | bajo |
| Riesgo de motor mal sin darnos cuenta | bajo | bajo | bajo | medio | bajo | medio | medio | **alto** | medio |

---

## 9. Recomendación y plan por etapas

Cada etapa deja algo presentable; si nos quedamos sin tiempo, cortamos donde estemos.

0. **Etapa 0 — Cola finita (H)**. Un parámetro `CEM(i)` y una condición. Es media hora de trabajo,
   acota la cola y evita que los escenarios saturados del final de la curva logística den números
   absurdos. *Verificable:* `q(i) ≤ CEM(i)` en toda la corrida (assert), y el conteo de balking
   forzado es > 0 en hora pico.
1. **Etapa 1 — Baseline (A).** Portar la regla del TP 4 con el umbral corregido sobre `q(i)`, para el
   "antes y después" del informe. *Verificable:* `PARR` con `CD = 4` en el orden del 53–60 % del TP 4.
2. **Etapa 2 — Reneging con paciencia exponencial (D).** Implementar el evento de abandono y la
   entrada `TPAB(i)` en la TEF. *Verificable:* la corrida degenerada reproduce **Erlang-A** dentro de
   la tolerancia fijada de antemano. **Esta es la etapa que más valor agrega**, porque es la única
   que nos da una vara externa para saber si el motor está bien. Queda implementada aunque el modelo
   final no lleve reneging.
3. **Etapa 3 — Paciencia realista (I, sobre D si va el reneging).** Cambiar la exponencial por la
   `TMEU` de la alternativa I —`0` con probabilidad `PNE`, `FI·TC` si no— y agregar el balking
   voluntario por `W_est`. Con esto el modelo ya es **E**. *Verificable:* entre los arribos que
   encuentran los cuatro cargadores ocupados y la cola vacía, la fracción que se arrepiente es
   `PNE + (1 − PNE) · 15,9 %` con `FI = 0,6`
   ([revisión §3.1](Calibracion_Arrepentimiento_Revision.md#31-los-números-con-nuestro-tc-no-con-el-del-ejemplo)),
   y nunca baja de `PNE`; y `PARR` sube al subir `TMP` (mantenimientos más espaciados → más
   cargadores fallados → menos `CD(i)` → más espera estimada).
4. **Etapa 4 — Perfiles (F).** Sólo si sobra tiempo.

---

## 10. Decisiones que tenemos que cerrar entre los cuatro

A éstas se suman las cinco de la
[revisión §10](Calibracion_Arrepentimiento_Revision.md#10-decisiones-a-cerrar-entre-los-cuatro):
escala de la paciencia, estimador de espera (cerrada: longitud de cola), piso del segmento que no
espera, reparto dentro de los tramos y si el arrepentido entra a `CA(i)`. Las que se superponen con
las de acá están actualizadas en su lugar.

1. **¿Qué versión del TP 4 es la válida?** ¿La del paper (93 % con 1 auto) o la del código (93 % con
   2)? Sin esto, cualquier comparación con el TP 4 es inválida. Y hay que decidir **cómo lo contamos
   en el informe del TP final**, dado que el 93 % está además mal interpretado
   ([§3.3](#33-el-93--del-tp-4-es-una-lectura-equivocada-del-paper)). Propuesta: decirlo, es un
   hallazgo del trabajo y queda mejor que dejarlo pasar.
2. **¿Paciencia absoluta, relativa (`FI·TC`) o una combinación?** Es la decisión de modelado más
   importante: con los cuatro cargadores ocupados y la cola vacía, `PARR` va de 16 % a 95 % según la
   escala. El mínimo de las dos no sirve: se queda con la paciencia chica, que es la que degenera.
   Propuesta: **la alternativa I**, con `FI` barrido entre 0,16, 0,34 y 0,60 (decisión 1 de la
   revisión).
3. **Denominador de `PARR`.** ¿Arrepentidos sobre los arribos que pasaron el filtro `PDCE`, o sobre
   todos los arribos generados? Propuesta: **sobre los que pasaron `PDCE`**, con el arrepentido fuera
   de `CA(i)`, de modo que atendidos + arrepentidos + perdidos por falla cierren el 100 % (decisión 5
   de la revisión). Y agregar la métrica de IDEAS: **porcentaje de atendidos sobre los que
   efectivamente hicieron cola**, que con reneging es la que mide el servicio.
4. **`PARR` por franja horaria — sólo si hay reneging (D).** Un auto que llega en la franja 2 y
   abandona en la 3, ¿en cuál cuenta? Propuesta: **franja de llegada**. Con balking puro no se
   plantea: la decisión es instantánea en el arribo.
5. **Falla del cargador mientras un auto está cargando** — **cerrada**: el auto **se pierde**, se va
   a la competencia. Es el caso particular de paciencia remanente igual a cero, así que la decisión
   vale cualquiera sea el modelo de arrepentimiento que se adopte y no queda atada a esta discusión.
   No cuenta como arrepentido: va a `CAPF(i)`, fuera de `CARRUM(i)` y de `PARR`. El detalle está en
   [Propuesta_TP-FINAL.md](Propuesta_TP-FINAL.md).
6. **Valor de `CEM(i)`.** ¿Fijo o proporcional a `CC(i)`? Propuesta: **2 lugares de espera por
   cargador**, para que escale con la expansión y siga siendo coherente con el argumento de espacio
   físico de `CC_MAX`.
7. **Tope del último tramo abierto** —"más de 30 min" en BC Hydro, "más de 1 hora" en EAFO— y reparto
   dentro de cada tramo. Propuesta: uniforme dentro del tramo y el tope justificado en una celda del
   notebook; Hanni et al. cierra sus opciones en "90 min o más", mejor anclaje que los 120 min que
   veníamos suponiendo. Baja de prioridad: el reparto mueve `PARR` a lo sumo 6,5 puntos (decisión 4
   de la revisión).
8. **Cancelación de eventos** (si vamos a D o E): recálculo del mínimo vs. evento fantasma.
   Propuesta: **recálculo**.
9. **¿`ECP` sobre los atendidos o sobre la FDP teórica?** Propuesta: **sobre los atendidos**, por
   el sesgo de selección de [§7](#7-efectos-de-segundo-orden-que-hay-que-anticipar). Afecta las dos
   condiciones de expansión.

---

## 11. Qué hay que tocar en la propuesta

Si adoptamos **E** —o sea **H** + **I** + **D**—, estos son los cambios mínimos a
[Propuesta_TP-FINAL.md](Propuesta_TP-FINAL.md). Si no va el reneging (H + I), sobran `PARRR`, `TAB`,
`TPAB` y el evento de abandono, y alcanza con desdoblar la fila del ingreso como en la
[revisión §7.7](Calibracion_Arrepentimiento_Revision.md#77-qué-habría-que-tocar-en-la-propuesta).

**Datos (FDP nueva)**

| Sigla | Significado |
|---|---|
| `TMEU` | Tiempo Máximo de Espera tolerado por el Usuario (min) |

**Resultado (desdoblar `PARR`)**

| Sigla | Significado |
|---|---|
| `PARRF` | Porcentaje de arrepentimiento forzado (no había lugar de espera) |
| `PARRB` | Porcentaje de arrepentimiento por no ingresar teniendo lugar (balking voluntario) |
| `PARRR` | Porcentaje de arrepentimiento por abandono de cola (reneging) |
| `PAA` | Porcentaje de Autos Atendidos sobre los que ingresaron a la cola |

**Estado / auxiliares**

| Sigla | Significado |
|---|---|
| `TAB(i)(k)` | Instante de abandono del k-ésimo auto en la cola de la estación (i) |

**Constantes**

| Sigla | Significado |
|---|---|
| `FI` | Factor de Impaciencia: fracción del tiempo de carga propio que el usuario tolera esperar (barrido 0,16–0,60; referencia de IDEAS: 0,6) |
| `PNE` | Porcentaje de usuarios que No Esperan: si encuentran todos los cargadores ocupados, se van (barrido 12–31 %) |
| `CEM` | Capacidad de Espera Máxima por estación (lugares de espera, atada a `CC(i)`) |

**TEF**

| Sigla | Significado |
|---|---|
| `TPAB(i)` | Tiempo de Próximo ABandono en la estación (i) = `min_k TAB(i)(k)` |

**Tabla de eventos — fila nueva y condición ampliada**

| Evento | E. F. no C. | E. F. C. | Condición |
|---|---|---|---|
| Ingreso de auto a una estación (i) | Ingreso (i) | Carga en cargador (i)(j) | `R ≤ PDCE/100 && CA(i) < CD(i)` |
| Ingreso de auto a una estación (i) | Ingreso (i) | Arrepentimiento en cola (i) | `R ≤ PDCE/100 && CA(i) ≥ CD(i) && q(i) < CEM(i) && W_est ≤ TMEU` |
| Arrepentimiento de auto en cola (i) | – | – | – |

---

## 12. Fuentes

Los PDF están en [`Bibliografia/`](Bibliografia/), con su índice.

1. **EAFO Consumer Monitor 2023 — European Aggregated Report.** Vanhaverbeke, Verbist, Barrera
   (VUB-MOBI) y Csukas (FIER), Comisión Europea, DG MOVE, junio 2024. doi:10.2832/062076
   ([PDF](Bibliografia/EAFO_Consumer_Monitor_2023_EU_Aggregated_Report.pdf)).
   Sección 3.4 y figura 10: esperas tolerada y declarada en puntos de carga públicos.
2. **IDEAS: Information-Driven EV Admission in Charging Station Considering User Impatience to
   Improve QoS and Station Utilization.** A. Chattopadhyay, S. Kar, IIT Delhi, arXiv:2403.06223v1
   (10 de marzo de 2024)
   ([PDF](Bibliografia/IDEAS_2024_Chattopadhyay_Kar_arXiv_2403.06223.pdf)). Balking
   forzado/voluntario y reneging (§III-A); factor de impaciencia `z = 0,6` (ec. 5 y 9); estimadores por perfil (ec. 10–12); cola con abandono desde cualquier
   posición (Algoritmo 1); los cuatro casos `BlockingFC` / `ObservationFC` / `InformedFC` /
   `Informed2PortCharge` (§V); resultados en la Tabla II.
3. **Distributed Electric Vehicles Charging Management Considering Time Anxiety and Customer
   Behaviors.** A. Alsabbagh, B. Wu, C. Ma, *IEEE Transactions on Industrial Informatics*, 2020
   ([PDF](Bibliografia/TimeAnxiety_2020_Alsabbagh_Wu_Ma_IEEE_TII.pdf)).
   Concepto de *time anxiety*, cuatro perfiles de conductor y sus formas funcionales (ec. 11–13).
4. **ACN-Data: Analysis and Applications of an Open EV Charging Dataset.** Z. J. Lee, T. Li, S. H. Low,
   *e-Energy '19*. Citado por IDEAS (figs. 3 y 4) para la duración de carga y el perfil horario de
   demanda; corrobora nuestro `TC` y nuestro corte de franjas.
5. **Paper del TP 4** (Carlana Rivero, Loglen, Millán, Ojeda Cabrera, UTN-FRBA)
   ([PDF](Bibliografia/Paper_TP_4.pdf)). §2.1: regla de arrepentimiento vigente; §3: resultados de
   referencia.
6. **Public Electric Vehicle Charging Service Rates — Evaluation Report for Year One.** British
   Columbia Hydro and Power Authority, presentado ante la British Columbia Utilities Commission el 29
   de agosto de 2025
   ([PDF](Bibliografia/2025-08-29-bchydro-public-ev-charging-service-rates-evaluation-report-year-1.pdf)).
   Apéndice C: disposición a esperar, encuesta de noviembre de 2024, preguntas 11d (urbano) y 12 (no
   urbano).
7. **Modeling of the Acceptable Waiting Time for EV Charging in Japan.** U. e Hanni, T. Yamamoto,
   T. Nakamura, *Sustainability* 2024, 16, 2536. doi:10.3390/su16062536
   ([PDF](Bibliografia/sustainability-16-02536.pdf)). §4.3 y figura 3: espera aceptable por ubicación,
   para carga normal y rápida.
8. **Should Charging Stations Provide Service for Plug-In Hybrid Electric Vehicles During Holidays?**
   T. Zhang, X. Li, Y. Zhang, C. Shu, *Sustainability* 2025, 17, 336. doi:10.3390/su17010336
   ([PDF](Bibliografia/sustainability-17-00336.pdf)). §4.2: forma exponencial de balking, tomada de
   Gross, Shortle, Thompson y Harris, *Fundamentals of Queueing Theory* (Wiley, 2008).

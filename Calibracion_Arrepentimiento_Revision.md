# Revisión de la propuesta de calibración del arrepentimiento

> **Estado:** revisión técnica de [Calibracion_Arrepentimiento_EV.md](Calibracion_Arrepentimiento_EV.md),
> pedida para evaluar su viabilidad antes de bajarla al modelo. No es una decisión tomada: las
> decisiones que quedan abiertas están en [§10](#10-decisiones-a-cerrar-entre-los-cuatro) y hay que
> cerrarlas entre los cuatro.
>
> Documentos relacionados: [Propuesta_TP-FINAL.md](Propuesta_TP-FINAL.md) (fuente de verdad del
> modelo) y [Arrepentimiento_Alternativas.md](Arrepentimiento_Alternativas.md) (las ocho alternativas
> y el plan por etapas).

---

## 1. Veredicto

La cadena de calibración es correcta en su lógica, pero **el último paso sobra y los dos anteriores,
con el `TC` que ajustamos en el TP 4, producen un modelo degenerado**.

| | |
|---|---|
| **Sirve como** | fuente de la FDP de paciencia (`TMEU`) y como disciplina de trazabilidad |
| **No sirve como** | formulación principal tal como está escrita |
| **Lo que la tumba** | la escala: con `E[TC] ≈ 121 min`, cualquier tabla de paciencia absoluta da `PARR ≥ 81 %` apenas se ocupan los cargadores ([§3](#3-el-problema-de-fondo-la-escala)) |
| **Lo que hay que borrar** | el ajuste de `α` ([§4.1](#41-ajustar-α-destruye-información-y-no-agrega-nada)): es una compresión con pérdida de una función que ya tenemos exacta |
| **Lo que hay que rescatar** | BC Hydro como fuente de `TMEU`, EAFO como validación externa, y el planteo de tipo de cargador de su §12 — que no es un ítem a verificar, es la restricción que decide todo |

---

## 2. Dónde encaja respecto de lo que ya teníamos

El documento fue escrito sin haber leído `Arrepentimiento_Alternativas.md`, y eso explica casi todos
sus puntos ciegos. Traducido a la nomenclatura que ya veníamos usando, lo que propone es:

- **forma funcional** = alternativa **B** (probabilidad continua en la longitud de cola);
- **calibración** = alternativa **C** (paciencia empírica contra espera estimada);
- **fuente de paciencia** = una fuente nueva (BC Hydro) en lugar de EAFO.

O sea: no es una novena alternativa, es B calibrada con la cadena de C. Eso está bien —es
exactamente el hueco que B tenía, que su `BETA` no salía de ningún dato— pero implica que **seis de
sus diez preguntas de §13 ya estaban contestadas**, y que el plan por etapas (`H → A → D → G+C → E`)
sigue siendo el marco donde esto se inserta, no algo que esta propuesta reemplace.

Lo que sí es aporte nuevo y hay que conservar está en [§5](#5-lo-que-hay-que-conservar).

---

## 3. El problema de fondo: la escala

### 3.1. Los números con nuestro `TC`, no con el del ejemplo

Corrimos la cadena `q → W(q) → P(TMEU < W(q))` con el `TC` que ajustamos en el TP 4
—`gumbel_r(loc=84,39; scale=60,19)` truncada a `[0,1; 1375,92]`, **`E[TC] = 121,4 min`**— y con las
dos tablas de paciencia disponibles, repartiendo uniforme dentro de cada tramo. El script está en el
[apéndice A](#apéndice-a-script-de-verificación).

**`CD = 4` cargadores (la estación inicial):**

| q | W(q) | PARR (BC Hydro) | PARR (EAFO) |
|---:|---:|---:|---:|
| 0 | 30,4 min | **99,0 %** | 81,2 % |
| 1 | 60,7 min | 99,3 % | 94,0 % |
| 2 | 91,1 min | 99,7 % | 95,6 % |
| 3 | 121,4 min | 100,0 % | 97,1 % |
| 4 | 151,8 min | 100,0 % | 98,6 % |

**`CD = 8` cargadores (una estación ya expandida):**

| q | W(q) | PARR (BC Hydro) | PARR (EAFO) |
|---:|---:|---:|---:|
| 0 | 15,2 min | 94,1 % | 63,2 % |
| 1 | 30,4 min | 99,0 % | 81,2 % |
| 2 | 45,5 min | 99,2 % | 87,7 % |

La conclusión no depende de la fuente de paciencia ni del tamaño de la estación: **el que llega y no
encuentra cargador libre se va, casi siempre**.

### 3.2. Por qué el ejemplo del documento no lo muestra

El ejemplo ilustrativo de su §4 y §6 (46 % / 76 % / 94 %) sale de suponer `E[TC] = 30 min`. Verificamos
que esos números son exactamente los que da la tabla de BC Hydro con ese tiempo de carga, así que el
procedimiento está bien aplicado — pero **30 minutos es carga rápida DC, y nuestro dataset tiene
mediana ≈ 107 min y media ≈ 121 min**. Son cuatro veces. El propio documento advierte en §6 que los
números son ilustrativos y hay que recalcularlos; el punto es que al recalcularlos el modelo no se
ajusta, se rompe.

Invirtiendo la cuenta, para que la cadena devuelva un piso razonable en `q = 0` con `CD = 4` haría
falta `W(0) ≈ 2,4 min`, o sea **`E[TC] ≈ 10 min`**. No hay valor de `α` que arregle esto, porque el
problema está aguas arriba del ajuste.

### 3.3. Qué le pasa al modelo si lo adoptamos igual

Si `PARR ≈ 95 %` en cuanto se ocupan los cargadores, la estación deja de ser una cola y pasa a
comportarse como un **sistema de pérdida** (M/M/c/c). Las consecuencias caen sobre variables que ya
están en la propuesta:

1. **`PEC(i) → 0` y `PPS(i) → TC`.** La cola prácticamente no se forma, y dos de nuestras siete
   variables de resultado dejan de tener variabilidad que medir.
2. **`CARRUM(i)` se dispara.** Como acumula todo lo que rebota, la condición
   `CARRUM(i)·(RC·ECP − CCP) > CPN` se cumple todos los meses y la red expande hasta `CC_MAX` y
   `CE_MAX` lo más rápido que le permitan `TIC`, `TCE` y las guardas `TPIC(i) = HV` / `TPCE = HV`.
   El Análisis de Expansión deja de ser una decisión económica y pasa a ser un cronograma.
3. **`TMP` se queda sin efecto que mover.** Con `PARR` clavado cerca del techo, sacar o poner
   cargadores fuera de servicio casi no lo corre: el análisis de sensibilidad de mantenimiento, que
   es una de nuestras tres variables de control, se queda sin resultado.

Es decir: el modelo corre, no tira ninguna excepción y devuelve números plausibles y equivocados.
Es el escenario que la regla de **Verificación del motor** de [CLAUDE.md](CLAUDE.md) anticipa.

---

## 4. Defectos concretos de la formulación

### 4.1. Ajustar `α` destruye información y no agrega nada

`P(ARR|q) = P(TMEU < W(q))` **ya es el modelo**, y es exacto. Ajustarle encima `1 − e^{−αq}` es una
compresión con pérdida de una función que ya está cerrada: el paso 4 de su cadena sólo puede agregar
error.

Implementarlo directo, además, no es más caro: se sortea `TMEU` por auto al arribo y se compara
contra `W`. Eso cuesta un número aleatorio, el mismo que costaría evaluar la exponencial, y viene con
dos ventajas:

- **conserva la lectura de segmento.** Con `P(ARR|q)` como moneda por arribo, el 17–31 % que no espera
  es un 17–31 % *distinto en cada arribo*. Con `TMEU` sorteado por auto, es una propiedad del auto.
  Para el promedio da igual, pero la conclusión económica del TP —*"este pedazo de la demanda se
  pierde igual, no lo comprás con `CPN`"*— sólo se sostiene con la segunda lectura;
- **reacciona a `CD(i)` y a las fallas** sin recalibrar nada, que es justo lo que `α` no hace
  ([§4.2](#42-α-no-es-una-constante)).

Su propia pregunta 8 apunta a esto. La respuesta es sí: la escalonada empírica directa, sin ajuste.

### 4.2. `α` no es una constante

`W(q)` depende de `CD(i)` y de `E[TC]`. Si calibramos `α` para `CD = 4` y la estación expande a
`CC_MAX`, o si se caen dos cargadores, la curva verdadera se mueve y un `α` fijo no se entera. Se ve
en las dos tablas de [§3.1](#31-los-números-con-nuestro-tc-no-con-el-del-ejemplo): pasar de `CD = 4` a
`CD = 8` corre la curva un lugar entero de `q`.

O sea que la forma cerrada se rompe **exactamente en los dos regímenes que el TP estudia**: la
expansión dinámica y el efecto de `TMP` sobre la disponibilidad. Meter `q/CD(i)` en el exponente
—como hace la alternativa B— parchea la expansión, pero no las fallas ni el `TC`.

Su pregunta 5 (un `α` o varios según cantidad de cargadores) tiene entonces una tercera respuesta: si
`α` tiene que depender de `CD`, ya no es un modelo de un parámetro, es el modelo de paciencia con
pasos de más.

### 4.3. El ajuste por mínimos cuadrados es degenerado

`1 − e^{−αq}` vale exactamente 0 en `q = 0`, y el objetivo empírico ahí es 99 %. Ese residuo no lo
absorbe ningún `α`. Lo corrimos sobre `q = 0..4` con BC Hydro y `CD = 4`:

```
alpha* = 5,016        SSE total = 0,9802
    de los cuales el punto q=0 aporta 0,9802   (todo)
curva ajustada:  q=0: 0 %   q=1: 99 %   q=2: 100 %  q=3: 100 %  q=4: 100 %
objetivo      :  q=0: 99 %  q=1: 99 %   q=2: 100 %  q=3: 100 %  q=4: 100 %
```

El óptimo es `α ≈ 5`, que traducido significa *"todos se arrepienten desde `q = 1`"*: el ajuste hace
lo único que puede, que es tirar la exponencial contra la pared para tapar los puntos que sí puede
tapar. Y la fórmula de un solo punto, `α = −ln(1−p)/q`, directamente no está definida en `q = 0`.

### 4.4. El piso en `q = 0` no es un caso de borde

En nuestro modelo el auto hace cola recién cuando `CA(i) > CD(i)`, así que **"cargadores todos
ocupados, cola vacía"** es el estado congestionado más frecuente, y es precisamente donde
`1 − e^{−αq}` dice que no se arrepiente nadie. Es una incompatibilidad estructural con la tabla de
eventos de la propuesta, no un detalle de precisión.

La variante de su §9, `P(ARR|q) = 1 − 0,69·e^{−αq}`, lo arregla, y es bastante más defendible de lo
que el propio documento cree. No es un híbrido armado entre dos fuentes: es un **modelo de mezcla**
—con probabilidad `p₀` el conductor es del segmento que no espera nunca, y si no, se comporta según la
exponencial— que es una construcción estándar y perfectamente citable. Escrita así no necesita
disculpa metodológica. (Lo que sigue sin arreglar es la escala de [§3](#3-el-problema-de-fondo-la-escala):
con nuestro `TC`, el 69 % restante también se va casi entero.)

---

## 5. Lo que hay que conservar

### 5.1. BC Hydro es mejor fuente que EAFO para `TMEU`

Es el aporte real del documento y conviene decirlo con todas las letras.
`Arrepentimiento_Alternativas.md` §4.1 deja anotado que EAFO está **censurada a derecha**: pregunta
*cuánto esperaste*, no *cuánto tolerarías*, así que el que nunca se cruzó con una cola larga reporta
poco aunque sea paciente, y la distribución subestima la paciencia real. BC Hydro pregunta
directamente *"how long are you willing to wait"*, que es la variable que el modelo necesita. Corrige
una debilidad que teníamos declarada como supuesto.

**Salvedad, y es grande:** la red pública de BC Hydro es mayoritariamente carga rápida, o sea que esa
paciencia está medida en el mismo contexto de sesiones de 30 minutos del ejemplo. Sirve como **cota
impaciente**, no como valor central para un dataset de carga de destino.

### 5.2. EAFO como validación externa y no como insumo

Su §8 y §10 proponen no forzar la calibración con el 31 % de EAFO y usarlo para contrastar el orden de
magnitud. Es correcto y coincide con lo que ya habíamos decidido. Mejor todavía: los dos valores del
piso —**17 % (BC Hydro) y 31 % (EAFO)**— dan un rango de sensibilidad con respaldo empírico, en lugar
de uno inventado.

### 5.3. `W(q)` tiene mejor pedigrí del que el documento le atribuye

Su §4 presenta `W(q) ≈ (q+1)/CD · E[TC]` como "hipótesis de simplificación" y su §11 la marca como el
riesgo principal. En realidad **es la espera condicional exacta de una M/M/c**: con los `c` servidores
ocupados, las salidas ocurren a tasa `c·μ`, el que llega con `q` adelante espera `q+1` salidas, y eso
da `(q+1)·E[S]/c`. Conviene citarla así, con nombre, en vez de disculparse por ella.

Es aproximada sólo porque nuestro `TC` es Gumbel y no exponencial, y **el sesgo es hacia arriba**: con
`CV = 0,625 < 1`, el remanente de equilibrio de una carga en curso tiene media
`E[TC²]/(2·E[TC]) = 84,4 min`, bastante menos que los 121,4 min que el estimador supone. Corrigiendo
por eso, `W(0)` baja de 30,4 a ≈ 21 min — y `PARR` en `q = 0` sigue dando 96 % con BC Hydro. La
conclusión de [§3](#3-el-problema-de-fondo-la-escala) es robusta a la corrección.

### 5.4. La disciplina de trazabilidad

Su pregunta 10 —documentar por separado dato observado, parámetro calibrado, hipótesis de
simplificación y validación externa— hay que adoptarla tal cual para todo el TP, no sólo para el
arrepentimiento. Es exactamente lo que pide CLAUDE.md sobre supuestos explicitados.

---

## 6. La premisa falsa, que es la que abre la mejor salida

El documento arranca (§1) y cierra (§11) sobre la base de que *"el dato disponible es la cantidad de
vehículos en cola, no el tiempo de espera real"*, y de ahí sale toda la necesidad de aproximar.

**En nuestro motor eso no es cierto: los `TPC(i)(j)` están en la TEF.** El instante en que se libera
cada cargador ocupado es un dato exacto del estado, no algo que haya que estimar. La espera real del
auto que llega a la posición `q+1` es el `(q+1)`-ésimo estadístico de orden de los fines de carga
pendientes (encadenando los `TC` de los que ya están en cola, si `q+1 > CD(i)`, cosa que también
tenemos si generamos `TC` en el arribo — ver [§7.4](#74-tc-se-sortea-en-el-arribo-no-al-empezar-a-cargar)).

Eso contesta su pregunta 2 y hace dos cosas:

1. **Elimina el riesgo principal que el documento declara.** No hace falta que `α` absorba el error de
   la traducción `q → W`, porque no hace falta la traducción.
2. **Convierte el problema en un eje experimental.** La diferencia entre lo que el usuario estima
   mirando la cola (AWT, *Assumed Wait Time*) y lo que la estación puede publicar con los remanentes
   reales (EWT, *Estimated Wait Time*) es la variable de control `IEC` de la alternativa E de
   `Arrepentimiento_Alternativas.md`. Es una palanca de eficiencia **que no cuesta capital**, frente a
   `CPN` y `CEN`, y es la conclusión más interesante que puede tener el TP.

---

## 7. Riesgos de integración con el modelo ya decidido

Puntos que el documento no toca y que hay que resolver sí o sí antes de escribir el motor.

### 7.1. Ortogonalidad con `PDCE` y doble conteo

El 31 % de EAFO responde *"si está ocupado me voy sin cargar"*. Nuestro `PDCE` ya saca de la demanda a
los que eligen otra estación. Son decisiones distintas —`PDCE` es **antes** de ver la estación,
el arrepentimiento es **después** de ver la cola— y hay que implementarlas así, pero el informe tiene
que decirlo explícitamente o el lector va a leer que contamos dos veces la misma fuga. Si `PARR` se
mide sobre los arribos que pasaron `PDCE`, el piso empírico aplica sobre ese subconjunto.

### 7.2. Balking puro no descomprime la cola

Con sólo balking, nadie abandona después de haber entrado: el auto que entró cuando `W` era tolerable
se queda aunque después se caiga un cargador y su espera se duplique. `PEC(i)` puede crecer sin que
nada la corrija, y **la cola no tiene tope físico**, porque la propuesta todavía no incorpora la cola
finita (alternativa H). Conviene meter H igual: es media hora de trabajo y es ortogonal a esta
discusión.

### 7.3. Se pierde la única validación externa disponible

Un modelo de balking sin abandono no tiene fórmula cerrada contra la cual contrastar. La alternativa D
(reneging con paciencia exponencial) sí: la corrida degenerada se compara contra **Erlang-A**. Es el
único chequeo del motor que teníamos identificado que no consiste en mirar el propio motor. Si
adoptamos balking puro, hay que dejar igual el modo degenerado con paciencia exponencial
implementado, sólo para validar.

### 7.4. `TC` se sortea en el arribo, no al empezar a cargar

Si la estimación de espera o la paciencia dependen del `TC` propio, el auto tiene que traerlo desde
que llega. El motor del TP 4 lo sortea al empezar a cargar. Es un cambio chico, pero **cambia dónde se
consume el número aleatorio**, y por lo tanto los resultados: hay que hacerlo antes de fijar la semilla
de las corridas del informe.

### 7.5. `ECP` queda sesgado por selección

Si la paciencia termina correlacionando con `TC` (ya sea por la alternativa G o porque la espera
estimada usa el `TC` propio), los autos que se quedan son sistemáticamente los de carga larga, y `ECP`
deja de ser la media de la FDP de energía. `ECP` está en **las dos condiciones de expansión**. Hay que
calcularlo sobre los atendidos y decirlo.

### 7.6. Un problema que el balking puro sí resuelve

A favor: como la decisión es instantánea en el arribo, **no hay ambigüedad de franja horaria**. La
decisión 4 de `Arrepentimiento_Alternativas.md` §10 (un auto que llega en una franja y abandona en
otra, ¿dónde cuenta?) desaparece. Con reneging habría que resolverla.

### 7.7. Qué habría que tocar en la propuesta

Mínimo, si adoptamos un balking por paciencia:

**Datos (FDP nueva)**

| Sigla | Significado |
|---|---|
| `TMEU` | Tiempo Máximo de Espera tolerado por el Usuario (min) |

**Tabla de eventos — la fila del ingreso se desdobla**

| Evento | E. F. no C. | E. F. C. | Condición |
|---|---|---|---|
| Ingreso de auto a una estación (i) | Ingreso (i) | Carga en cargador (i)(j) | `R < PDCE/100 && CA(i) ≤ CD(i)` |
| Ingreso de auto a una estación (i) | Ingreso (i) | — (se arrepiente) | `R < PDCE/100 && CA(i) > CD(i) && W_est > TMEU` |

No agrega eventos ni entradas a la TEF, que es su principal virtud de implementación.

**Contabilidad.** La propuesta hoy exige que atendidos + arrepentidos + perdidos por falla cierren el
100 % de los ingresos. Hay que definir si el auto que se arrepiente **entra** a `CA(i)` y sale, o
nunca entra (esto último es lo correcto: no ocupa lugar ni tiempo), y si cuenta como "ingreso" a los
efectos del denominador de `PARR(i)` y `PAPF(i)`.

---

## 8. Respuestas a las diez preguntas del documento

| # | Pregunta | Respuesta |
|---:|---|---|
| 1 | ¿Es válida `W(q) ≈ (q+1)/CD · E[TC]`? | Sí, y con mejor fundamento del que le atribuye: es la espera condicional exacta de una M/M/c. Sesgada hacia arriba con Gumbel ([§5.3](#53-wq-tiene-mejor-pedigrí-del-que-el-documento-le-atribuye)). |
| 2 | ¿Hay mejor forma de traducir cola a espera? | Sí: no traducir. Los remanentes están en la TEF ([§6](#6-la-premisa-falsa-que-es-la-que-abre-la-mejor-salida)). |
| 3 | ¿BC Hydro es comparable con nuestro dataset? | En tipo de pregunta sí, y es mejor que EAFO. En tecnología de carga no: es red mayoritariamente rápida ([§5.1](#51-bc-hydro-es-mejor-fuente-que-eafo-para-tmeu)). |
| 4 | ¿La exponencial es buena forma funcional? | Es razonable y estándar en colas con balking, **muy anterior a Zhang 2025** (IDEAS ya la toma de un Zhang et al. de 2020). Citar el antecedente, no un paper de 2025 como origen. Pero acá el problema no es la forma: es el piso en `q = 0` y la dependencia de `CD` ([§4.2](#42-α-no-es-una-constante), [§4.4](#44-el-piso-en-q--0-no-es-un-caso-de-borde)). |
| 5 | ¿Un `α` o varios según cargadores? | Ninguno de los dos: si `α` depende de `CD`, es el modelo de paciencia con pasos de más. |
| 6 | ¿EAFO sólo validación o también base? | Sólo validación, como propone. Y el par 17 %–31 % como rango de sensibilidad del piso. |
| 7 | ¿Hay literatura que estime `P(ARR|q)` directamente para EV? | En lo que tenemos en `Bibliografia/`, no: hay formas funcionales (Zhang, IDEAS) y encuestas de paciencia, pero no una estimación empírica directa. Las estimaciones empíricas de balking por longitud de cola que conocemos vienen de call centers y retail, no de carga EV. No lo damos por cerrado: hace falta una búsqueda dedicada. |
| 8 | ¿Reemplazar la exponencial por la escalonada empírica? | **Sí. Es el punto central** ([§4.1](#41-ajustar-α-destruye-información-y-no-agrega-nada)). |
| 9 | ¿Qué análisis de sensibilidad? | Barrer el piso entre 17 % y 31 %, y el parámetro de escala de la paciencia entre la cota absoluta y la relativa (`FI·TC`). El criterio de éxito no es que `PARR` se mueva poco, sino que **la conclusión económica no se dé vuelta** entre las cotas. |
| 10 | ¿Cómo documentar dato / parámetro / hipótesis / validación? | Su propio esquema, adoptado para todo el TP ([§5.4](#54-la-disciplina-de-trazabilidad)). |

---

## 9. Qué proponemos hacer

1. **Reescribir el documento como fuente de `TMEU`**, no como formulación principal. Ahí aporta, y
   aporta bien: nos da una distribución de paciencia mejor fundada que la que teníamos.
2. **Sacar el paso de ajuste de `α`.** Sortear `TMEU` por auto en el arribo y comparar contra la
   espera. Mismo costo, sin error de ajuste, y reacciona a `CD(i)` y a las fallas.
3. **Resolver la escala antes que cualquier otra cosa**, porque es lo que decide si el modelo sirve.
   Dos salidas, y hay que elegir una:
   - **paciencia relativa al `TC` propio** (`TMEU = FI·TC`, IDEAS, `FI = 0,6`): da `TMEU` medio
     ≈ 73 min contra `W(0)` ≈ 30 min, o sea un gradiente sano —se entra con la cola vacía, se duda en
     `q = 1`, se abandona en `q = 2`—; o
   - **paciencia absoluta con una tabla de un contexto de carga comparable** al del dataset (carga de
     destino / Level 2), que hay que salir a buscar.

   Lo que **no** resuelve la degeneración es el `min()` de las dos que proponía
   `Arrepentimiento_Alternativas.md` §5.G: se queda con la chica, que es justamente la que rompe. Ese
   punto de aquel documento queda corregido por esta revisión.
4. **Usar el estimador exacto de espera** (los remanentes de la TEF) y dejar el estimador por
   longitud de cola como el escenario "sin información", que es el eje experimental gratis
   ([§6](#6-la-premisa-falsa-que-es-la-que-abre-la-mejor-salida)).
5. **Bajar los dos PDF nuevos a `Bibliografia/`** con su entrada en el índice, como los otros cuatro.

Nada de esto es caro ahora: el motor del TP final todavía no está escrito.

---

## 10. Decisiones a cerrar entre los cuatro

Se suman a las nueve de `Arrepentimiento_Alternativas.md` §10.

1. **Escala de la paciencia:** ¿relativa (`FI·TC`) o absoluta con fuente nueva? Es la decisión que
   define si el TP tiene resultados o no. Propuesta: **relativa**, con la absoluta como cota
   impaciente del análisis de sensibilidad.
2. **`W_est` exacto (remanentes de la TEF) o aproximado (longitud de cola)?** Propuesta: **los dos**,
   como escenarios de la variable `IEC`.
3. **Piso del segmento que no espera:** 17 % (BC Hydro), 31 % (EAFO) o barrido entre ambos.
   Propuesta: **barrido**.
4. **Reparto dentro de cada tramo de la tabla de paciencia** y **tope del último tramo abierto**
   ("más de 30 min" en BC Hydro, "más de 1 hora" en EAFO). Propuesta: uniforme dentro del tramo, y el
   tope elegido y justificado en una celda markdown del notebook.
5. **¿El arrepentido entra a `CA(i)`?** Propuesta: **no entra**, y se define explícitamente el
   denominador de `PARR(i)` para que atendidos + arrepentidos + perdidos por falla cierren el 100 %.

---

## 11. Sobre las fuentes

**No verificamos** el contenido de las dos fuentes nuevas: ni Zhang et al. (2025) ni el informe de
BC Hydro están en `Bibliografia/`, y esta revisión se hizo sobre lo que el documento reporta de ellas.
Antes de adoptar cualquier número hay que bajar los PDF y chequear, como mínimo:

- que `b_n = e^{−αn}` sea efectivamente la forma que usa Zhang et al. (2025) en su §4.2, y de dónde la
  toman ellos (la exponencial de balking es anterior y más general que ese paper);
- la tabla de BC Hydro: año del relevamiento, si la pregunta es sobre carga rápida o mixta, tamaño de
  muestra, y si el "urban area" del enunciado es comparable con CABA.

Lo que sí verificamos con el material propio, y está reproducido en el
[apéndice A](#apéndice-a-script-de-verificación): `E[TC]`, la tabla `q → W(q) → PARR`, el ajuste
degenerado de `α` y el remanente de equilibrio de la Gumbel.

---

## Apéndice A: script de verificación

Python puro, sin dependencias. Reproduce todos los números de [§3](#3-el-problema-de-fondo-la-escala),
[§4.3](#43-el-ajuste-por-mínimos-cuadrados-es-degenerado) y
[§5.3](#53-wq-tiene-mejor-pedigrí-del-que-el-documento-le-atribuye).

```python
import math

# TC del TP 4: gumbel_r(loc=84,39; scale=60,19) truncada a [0,1; 1375,92]
loc, sc, lo, hi = 84.39, 60.19, 0.1, 1375.92
cdf = lambda x: math.exp(-math.exp(-(x - loc) / sc))
pdf = lambda x: math.exp(-(x - loc) / sc - math.exp(-(x - loc) / sc)) / sc

Z = cdf(hi) - cdf(lo)
N = 2_000_000
h = (hi - lo) / N
m1 = sum((lo + (k + .5) * h) * pdf(lo + (k + .5) * h) for k in range(N)) * h / Z
m2 = sum((lo + (k + .5) * h) ** 2 * pdf(lo + (k + .5) * h) for k in range(N)) * h / Z
ETC = m1                                        # 121,4 min
print(f"E[TC]={m1:.1f}  CV={math.sqrt(m2 - m1**2) / m1:.3f}")
print(f"remanente de equilibrio = {m2 / (2 * m1):.1f} min")   # 84,4 min

# Distribuciones de paciencia, como (tope del tramo, proporcion)
BC   = [(0, .17), (5, .29), (10, .30), (15, .18), (30, .05), (120, .01)]  # BC Hydro
EAFO = [(0, .31), (15, .32), (30, .18), (60, .13), (180, .06)]            # EAFO 2023

def P_balk(W, tabla):
    """P(TMEU < W), reparto uniforme dentro de cada tramo."""
    acc, prev = 0.0, 0.0
    for top, p in tabla:
        if W >= top:
            acc += p
        elif W > prev:
            acc += p * (W - prev) / (top - prev)
            break
        else:
            break
        prev = top
    return min(acc, 1.0)

W = lambda q, CD: (q + 1) / CD * ETC

for CD in (4, 8):
    print(f"\n--- CD={CD} ---")
    for q in range(5):
        w = W(q, CD)
        print(f"q={q}  W={w:6.1f} min   BC={P_balk(w, BC):6.1%}   EAFO={P_balk(w, EAFO):6.1%}")

# Ajuste de alpha por minimos cuadrados (BC Hydro, CD=4, q=0..4)
pts = [(q, P_balk(W(q, 4), BC)) for q in range(5)]
sse, a = min((sum((p - (1 - math.exp(-a * q))) ** 2 for q, p in pts), a)
             for a in (i / 10000 for i in range(1, 200001)))
print(f"\nalpha*={a:.3f}  SSE={sse:.4f}  (el punto q=0 aporta {pts[0][1]**2:.4f})")
```

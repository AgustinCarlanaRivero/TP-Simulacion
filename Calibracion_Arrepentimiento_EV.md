# Propuesta para calibrar el arrepentimiento en la simulación de estaciones de carga EV

## 1. Objetivo

El TP necesita modelar el **arrepentimiento al llegar a una estación de carga** (*balking*): un vehículo llega, observa la congestión y decide si entra al sistema o se retira sin cargar.

La principal restricción del modelo es que, durante la simulación, el dato disponible es principalmente la **cantidad de vehículos en el sistema / cola**, no el tiempo de espera real que le queda a cada vehículo.

Por eso se busca construir una función del tipo:

\[
P(ARR \mid q)
\]

donde:

- `ARR`: arrepentimiento / balking.
- `q`: cantidad de vehículos esperando en cola.

La idea es evitar elegir porcentajes arbitrarios por cantidad de vehículos y, en cambio, **calibrar una función de arrepentimiento usando evidencia externa y datos propios del TP**.

---

## 2. Función propuesta por literatura específica de EV

Zhang et al. (2025), en un modelo de estaciones de carga con *balking* y *reneging*, representan la probabilidad de que un vehículo ingrese a la cola como:

\[
b_n=e^{-\alpha n}
\]

donde:

- \(n\): longitud de la cola.
- \(\alpha\): parámetro de sensibilidad o impaciencia del conductor.
- \(b_n\): probabilidad de que el conductor entre al sistema.

Por complemento:

\[
P(ARR \mid q)=1-e^{-\alpha q}
\]

Interpretación:

- si \(\alpha\) es bajo, los conductores son relativamente pacientes;
- si \(\alpha\) es alto, la probabilidad de arrepentimiento aumenta rápidamente al crecer la cola;
- cuando aumenta \(q\), aumenta \(P(ARR)\).

### Importante

El paper **justifica la forma funcional**, pero no proporciona un valor empírico universal de \(\alpha\) calibrado para todos los usuarios de EV.

Por lo tanto, copiar un valor arbitrario de \(\alpha\) sería metodológicamente débil.

La propuesta es **calibrarlo**.

Fuente:

Zhang et al. (2025). *Should Charging Stations Provide Service for Plug-In Hybrid Electric Vehicles During Holidays?* Sustainability, 17(1), 336.

https://www.mdpi.com/2071-1050/17/1/336

Revisar especialmente la sección:

**4.2. Queuing System with Balking and Reneging**

---

## 3. Idea general de calibración

La cadena propuesta es:

\[
\boxed{
\text{Tiempo de carga del dataset}
\rightarrow
W(q)
\rightarrow
\text{Distribución empírica de paciencia}
\rightarrow
P(ARR\mid q)
\rightarrow
\alpha
}
\]

Es decir:

1. A partir de la cantidad de vehículos en cola \(q\), estimar un tiempo de espera representativo.
2. Utilizar una distribución empírica de tolerancia a la espera para estimar qué proporción de conductores abandonaría ante esa espera.
3. Obtener varios puntos \((q, PARR)\).
4. Ajustar \(\alpha\) para que la función exponencial de Zhang reproduzca lo mejor posible esos puntos.

De esta forma, el parámetro \(\alpha\) deja de ser arbitrario.

---

## 4. Paso 1 — Relacionar cola con espera aproximada

Supongamos:

- \(q\): cantidad de vehículos esperando.
- \(CD\): cantidad de cargadores disponibles / operativos.
- \(E[TC]\): tiempo medio de carga obtenido del dataset utilizado por el TP.

Una aproximación posible, cuando todos los cargadores están ocupados, es:

\[
W(q)\approx \frac{q+1}{CD}E[TC]
\]

La interpretación es que el nuevo vehículo debe esperar aproximadamente una fracción del tiempo medio de servicio según cuántos vehículos tenga delante y cuántos puestos operativos existan.

### Ejemplo ilustrativo

Si:

\[
E[TC]=30\text{ minutos}
\]

y:

\[
CD=4
\]

entonces:

\[
q=0 \Rightarrow W\approx7.5\text{ min}
\]

\[
q=1 \Rightarrow W\approx15\text{ min}
\]

\[
q=2 \Rightarrow W\approx22.5\text{ min}
\]

### Advertencia metodológica

Esta fórmula **no proviene directamente de Zhang**.

Es una **hipótesis de simplificación / aproximación estadística** para traducir la información disponible en el TP (`q`) a una magnitud comparable con estudios de paciencia expresados en minutos.

Debe validarse si resulta consistente con:

- la disciplina de cola utilizada;
- cantidad de cargadores;
- distribución real de tiempos de carga;
- arquitectura del modelo Evento a Evento.

Si el simulador dispone de una estimación mejor de espera, debería preferirse esa alternativa.

---

## 5. Paso 2 — Obtener una distribución empírica de paciencia

Una fuente potencial es BC Hydro, que preguntó directamente a usuarios de cargadores públicos:

> “How long are you willing to wait to get access to an available BC Hydro charger in an urban area?”

En su relevamiento 2024 de usuarios de carga pública, para zonas urbanas reporta categorías de tolerancia aproximadas:

| Máximo tiempo dispuesto a esperar | Proporción |
|---|---:|
| Nada | 17% |
| 5 min | 29% |
| 10 min | 30% |
| 15 min | 18% |
| 20–30 min | 5% |
| Más de 30 min | 1% |

Esto permite construir una distribución acumulada de conductores que **ya no tolerarían** determinado tiempo de espera.

Ejemplo conceptual:

- si la espera supera 0 minutos, existe un grupo que no acepta esperar nada;
- si supera 5 minutos, se acumulan quienes toleraban 0 y hasta 5 minutos;
- si supera 10 minutos, se agregan quienes toleraban hasta 10 minutos;
- etc.

Fuente:

BC Hydro. *Public EV Charging Service Rates Evaluation Report*.

https://www.bchydro.com/content/dam/BCHydro/customer-portal/documents/corporate/regulatory-planning-documents/regulatory-filings/reports/2025-08-29-bchydro-public-ev-charging-service-rates-evaluation-report-year-1.pdf

### Fuente académica complementaria

Hanni, Yamamoto & Nakamura (2024) estudian explícitamente el concepto de **acceptable waiting time** para carga EV y utilizan intervalos de tolerancia desde “no esperar” hasta esperas prolongadas.

Fuente:

https://www.mdpi.com/2071-1050/16/6/2536

Esta fuente puede utilizarse para justificar conceptualmente que la paciencia del conductor se modele como una variable aleatoria.

---

## 6. Paso 3 — Construir puntos \(q \rightarrow P(ARR)\)

Para cada valor relevante de \(q\):

1. calcular \(W(q)\);
2. consultar la distribución de paciencia;
3. estimar qué proporción de usuarios tendría una tolerancia menor a esa espera.

Formalmente:

\[
P(ARR\mid q)
=
P(TMEU<W(q))
\]

donde:

- `TMEU`: tiempo máximo que un usuario está dispuesto a esperar;
- `W(q)`: espera aproximada asociada a una cola de longitud \(q\).

Esto genera puntos como:

\[
(q_1,p_1),(q_2,p_2),...,(q_k,p_k)
\]

### Ejemplo puramente ilustrativo

Si la combinación de tiempo de carga y cantidad de cargadores generara:

| q | W(q) | PARR derivado de paciencia |
|---:|---:|---:|
| 0 | 7.5 min | ~46% |
| 1 | 15 min | ~76% |
| 2 | 22.5 min | ~94% |

estos valores podrían utilizarse como objetivos para la calibración.

**Estos números son sólo un ejemplo del procedimiento y no deben incorporarse al TP sin recalcularlos con los parámetros reales del modelo.**

---

## 7. Paso 4 — Calibrar \(\alpha\)

La función objetivo es:

\[
P(ARR\mid q)=1-e^{-\alpha q}
\]

Si se tuviera un único punto confiable \((q,p)\), podría despejarse:

\[
\boxed{
\alpha=-\frac{\ln(1-p)}{q}
}
\]

Sin embargo, metodológicamente es mejor construir varios puntos y obtener un único \(\alpha\) que minimice el error total.

Por ejemplo:

\[
\alpha^*
=
\arg\min_{\alpha}
\sum_q
\left[
p_q-\left(1-e^{-\alpha q}\right)
\right]^2
\]

De esta manera:

- la **forma de la función** viene de Zhang et al.;
- los **niveles de paciencia** vienen de evidencia empírica;
- la relación entre cola y tiempo de espera utiliza datos propios del TP;
- \(\alpha\) es un parámetro **calibrado**, no elegido arbitrariamente.

---

## 8. Papel del dato de EAFO del 31%

EAFO Consumer Monitor 2023 reporta que aproximadamente:

\[
31\%
\]

de los conductores BEV encuestados declara que, cuando encuentra un punto público ocupado, **no espera y se va sin recargar**.

Fuente:

European Alternative Fuels Observatory (EAFO), *Consumer Monitor 2023 – EU Aggregated Report*.

https://alternative-fuels-observatory.ec.europa.eu/system/files/documents/2024-06/EU%20Aggregated%20Report%202023_0.pdf

### Recomendación

No utilizar necesariamente ese 31% para forzar la calibración de \(\alpha\), porque:

- EAFO pregunta por comportamiento ante un punto ocupado;
- BC Hydro / Hanni miden tolerancia en tiempo;
- son conceptos relacionados, pero no idénticos.

Es más limpio utilizar el 31% de EAFO como **validación externa**.

Por ejemplo:

> Luego de calibrar el modelo mediante paciencia y longitud de cola, verificar si el porcentaje de balking observado en situaciones de ocupación resulta razonablemente compatible con el orden de magnitud reportado por EAFO.

---

## 9. Posible variante: incorporar el 31% como nivel base

Otra alternativa exploratoria sería:

\[
P(ARR\mid q)
=
1-(1-0.31)e^{-\alpha q}
\]

equivalente a:

\[
P(ARR\mid q)
=
1-0.69e^{-\alpha q}
\]

Interpretación:

- con \(q=0\), si todos los cargadores están ocupados pero todavía no hay vehículos esperando:

\[
PARR=31\%
\]

- cuando aumenta \(q\), la probabilidad aumenta progresivamente hacia 100%.

### Advertencia muy importante

Esta función combinada **no aparece explícitamente en Zhang ni en EAFO**.

Es una **hipótesis de simplificación construida combinando**:

- el valor base de EAFO;
- la forma exponencial de Zhang.

Por lo tanto, si el TP exige máxima trazabilidad, conviene tratarla como una alternativa o análisis de sensibilidad y no como la formulación principal salvo que se justifique expresamente.

---

## 10. Alternativa metodológicamente más conservadora

La opción más defendible sería:

### Modelo principal

\[
P(ARR\mid q)=1-e^{-\alpha q}
\]

con \(\alpha\) calibrado mediante:

\[
q
\rightarrow
W(q)
\rightarrow
P(TMEU<W(q))
\]

### Validación externa

Comparar el resultado con:

- EAFO: 31% que no espera cuando encuentra el punto ocupado;
- otros estudios sobre acceptable waiting time;
- análisis de sensibilidad sobre \(\alpha\).

Esto evita mezclar directamente encuestas diferentes dentro de una única fórmula.

---

## 11. Riesgo principal de la calibración

El punto más delicado es la traducción:

\[
q\rightarrow W(q)
\]

porque el TP no conoce necesariamente el tiempo residual exacto de cada vehículo que está cargando.

Si esa traducción es demasiado simplificada, el \(\alpha\) calibrado absorberá errores de esa aproximación.

Por eso se recomienda:

1. utilizar el \(E[TC]\) obtenido del dataset real;
2. recalcular la calibración para distintos valores de `CD`;
3. hacer análisis de sensibilidad;
4. verificar si la curva resultante produce comportamientos razonables;
5. evitar presentar \(\alpha\) como un parámetro universal de conducta humana.

---

## 12. Cuestión adicional a verificar: tipo de cargador

Antes de utilizar estudios de tolerancia, debería revisarse el dataset del TP:

- `Port Type`
- `Plug Type`
- `Model Number`

La razón es que la tolerancia a la espera puede ser diferente entre:

- carga Level 2 / destination charging;
- DC fast charging;
- estaciones urbanas;
- estaciones de ruta.

Si el dataset del TP representa principalmente un tipo de carga distinto del estudiado en BC Hydro o Hanni, debería:

- buscarse una fuente más específica; o
- declararse la transferencia como hipótesis;
- realizar análisis de sensibilidad.

---

## 13. Recomendación final para evaluación por otro agente

Evaluar si resulta metodológicamente válido adoptar como modelo principal:

\[
\boxed{
P(ARR\mid q)=1-e^{-\alpha q}
}
\]

con:

\[
\boxed{
\alpha
\text{ calibrado a partir de }
q\rightarrow W(q)\rightarrow P(TMEU<W(q))
}
\]

### Preguntas específicas para revisar

1. ¿Es válida la aproximación:

\[
W(q)\approx\frac{q+1}{CD}E[TC]
\]

para este sistema con múltiples cargadores y una cola única?

2. ¿Existe una mejor forma de traducir longitud de cola a espera sin conocer tiempos residuales?

3. ¿BC Hydro es una fuente suficientemente comparable con el tipo de infraestructura representado por el dataset del TP?

4. ¿La función exponencial de Zhang puede considerarse una buena forma funcional para este caso?

5. ¿Conviene calibrar un único \(\alpha\) o diferentes \(\alpha\) según cantidad de cargadores / tipo de estación?

6. ¿Debería utilizarse EAFO sólo para validación o también como condición base?

7. ¿Existe literatura empírica que estime directamente una relación:

\[
P(ARR\mid q)
\]

para estaciones de carga EV?

8. ¿Conviene reemplazar la función exponencial por una función empírica escalonada obtenida directamente de la distribución de paciencia?

9. ¿Qué análisis de sensibilidad debería realizarse para demostrar que los resultados del TP no dependen excesivamente de la elección de \(\alpha\)?

10. ¿Cómo debería documentarse la diferencia entre:
   - datos observados,
   - parámetros calibrados,
   - hipótesis de simplificación,
   - validación externa?

---

## 14. Resumen ejecutivo

El TP necesita que el arrepentimiento dependa de una variable disponible durante la simulación: la **cantidad de vehículos en cola**.

Existe literatura EV que propone:

\[
P(ARR\mid q)=1-e^{-\alpha q}
\]

pero no aporta un valor universal de \(\alpha\).

La propuesta es calibrar \(\alpha\) mediante:

\[
\boxed{
\text{longitud de cola}
\rightarrow
\text{espera aproximada}
\rightarrow
\text{paciencia empírica}
\rightarrow
PARR
\rightarrow
\alpha
}
\]

De esta forma:

- la estructura matemática proviene de literatura específica de EV;
- la paciencia proviene de encuestas reales;
- los tiempos de carga provienen del dataset del TP;
- el parámetro final es calibrado y no arbitrario.

EAFO (31%) se recomienda principalmente como **fuente de validación externa**, no necesariamente como parte de la calibración principal.

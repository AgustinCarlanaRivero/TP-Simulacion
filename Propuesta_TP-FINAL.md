# Propuesta de Trabajo Práctico Final — Simulación

**UTN – FRBA** · Ingeniería en Sistemas de Información
**Cátedra:** Simulación
**Docentes:** Ing. Gladys Alfiero, Ing. Erica M. Milin, Ing. Silvia Quiroga

> Propuesta aprobada. Es la fuente de verdad del modelo: variables, eventos y condiciones. Si se
> corrige el modelo, se corrige acá.

---

## Título del trabajo

> Estudio de la eficiencia técnica, operativa y financiera en la infraestructura de una estación de
> carga de vehículos eléctricos a través de la simulación de eventos discretos en la Ciudad Autónoma
> de Buenos Aires

---

## Descripción de la simulación a realizar

El crecimiento sostenido de la movilidad eléctrica en Argentina, y especialmente en la Ciudad Autónoma
de Buenos Aires, plantea el desafío estratégico de dimensionar adecuadamente la infraestructura de
carga de vehículos eléctricos. El presente trabajo propone la simulación del funcionamiento de
estaciones de carga utilizando la metodología de simulación Evento a Evento. El objetivo es analizar
la eficiencia técnica, operativa y financiera de la infraestructura para encontrar un equilibrio
óptimo: determinar la política de expansión y los valores de las variables de control que eviten la
congestión y el arrepentimiento de los usuarios por falta de capacidad, sin incurrir en un
sobredimensionamiento que derive en recursos inactivos ni en costos de mantenimiento y pérdidas por
indisponibilidad técnica evitables.

**Características importantes:**

- **Comportamiento dinámico de la demanda:** Se contabilizan únicamente los vehículos eléctricos e
  híbridos recargables por enchufe, y se le resta el porcentaje de usuarios que cargan en su
  domicilio. A corto plazo, la variabilidad de la demanda se gestiona mediante distintas franjas
  horarias para días hábiles y fines de semana. A largo plazo, el crecimiento se modela con una curva
  logística, en forma de "S".
- **Estructura dinámica de costos:** El costo de la energía fluctúa dependiendo de las tarifas por
  franja horaria (picos y valles). A esto se le suman los costos variables de mantenimiento, la
  amortización por instalación de nueva infraestructura, y los costos directos de inversión requeridos
  para agregar cada nuevo puesto de carga o construir una nueva estación.
- **Expansión dinámica de la infraestructura:** El sistema permite agregar nuevos puestos y estaciones
  de forma dinámica durante la simulación. La decisión se basa en la conveniencia económica: se
  incorpora un nuevo cargador si la pérdida por demanda rechazada (falta de capacidad) resulta mayor
  al costo de instalarlo, o bien, se construye una nueva estación si la ganancia potencial por captar
  una mayor porción de la demanda supera los costos de agregar dicha estación.
- **Variables de control estratégico:** El modelo permite configurar distintos escenarios para buscar
  el equilibrio técnico, operativo y financiero mediante tres variables clave:
  - **Recaudación por Carga (RC):** Medida en $/kWh. Define el precio de venta de la energía al
    consumidor, permitiendo buscar el valor óptimo que maximice el beneficio financiero frente a los
    costos fluctuantes de la estación.
  - **Tiempo entre Mantenimientos Preventivos (TMP):** Medido en días. Establece la frecuencia con la
    que se revisan los cargadores. Variar el TMP permite analizar el balance entre los costos de
    realizar el mantenimiento y las pérdidas económicas por indisponibilidad técnica.
  - **Porcentaje de Demanda a Capturar por Estación (PDCE):** Determina qué porción del mercado total
    es absorbida por cada estación. Sirve para simular cómo se distribuyen los vehículos en la red y
    evaluar si capturar un mayor porcentaje justifica económicamente la construcción de nuevas
    estaciones ante el crecimiento del parque automotor eléctrico. También tiene en consideración que
    una parte de la demanda será captada por la competencia.

---

## ¿Qué complejidad que extienda los casos vistos durante las clases tiene la simulación propuesta?

Este modelo plantea varias complejidades, las cuales se detallan a continuación:

- **Variabilidad de la demanda mediante franjas horarias:** El flujo de clientes no será constante. La
  operatoria se dividirá entre días de semana y fines de semana, y a su vez, estará segmentada en
  distintas franjas horarias. Cada una de estas franjas y tipos de día (semana o fin de semana) tendrá
  asignada una función de densidad de probabilidad (FDP) de intervalos de arribos (IA) diferente,
  simulando los picos y valles de demanda reales.
- **Crecimiento de la demanda a largo plazo:** La demanda real no solo varía mediante franjas
  horarias, también varía a largo plazo, y sobre todo en un país donde la incorporación de autos
  eléctricos es reciente. En otros países bajo situaciones similares, se ha observado que por un
  tiempo la demanda crece de forma acelerada, hasta llegar a un punto de inflexión a partir del cual
  crece de forma desacelerada. Este comportamiento se describe con una curva logística, la cual tiene
  forma de "S" (`P(t) = K / (1 + A · e^(−r·t))`). Es por eso que ajustaremos dicha curva al historial
  en CABA (basándonos en las características de la misma para otros países), y obtendremos de ella un
  Factor (variable en el tiempo) por el cual multiplicaremos nuestros Intervalos entre Arribos, para
  que disminuyan en el periodo de aceleración y aumenten en el período de desaceleración.
- **Múltiples estaciones y cargadores:** El sistema comienza con 1 estación y 4 cargadores, pero la
  cantidad de cargadores puede ampliarse a medida que aumenta la demanda, así como pueden aumentar la
  cantidad de estaciones cuando convenga capturar una mayor porción de la demanda. Cada 1 mes, se
  realiza un análisis de expansión, en el que se evalúan ambos casos en función de su conveniencia
  económica: si la demanda perdida por falta de capacidad representa una pérdida mayor al costo de
  incorporar un nuevo cargador, se agrega un puesto; y si la demanda que se podría captar en una nueva
  estación representa una ganancia mayor al costo de construirla, se agrega una nueva estación. Ambas
  expansiones tendrán un límite, dado por el espacio físico (CC_MAX y CE_MAX). Cada estación tiene su
  propio flujo de arribos: al ocurrir un arribo en la estación (i) se evalúa contra el PDCE si el
  vehículo efectivamente ingresa, de modo que cada estación captura ese porcentaje de la demanda total
  y el resto queda en manos de la competencia. Los vehículos que ingresan son atendidos por los
  cargadores de esa estación, que atienden por una única cola FCFS. Como cada estación modela sus
  arribos por separado, la red en conjunto
  capta un PDCE * CE % de la demanda, que es justamente lo que acota la condición de construcción de
  una nueva estación. De esta forma, la cantidad de estaciones y puestos evoluciona durante la
  simulación según la demanda y la conveniencia económica de ampliar la infraestructura.
- **Falla y mantenimiento de cargadores:** Los cargadores no poseen disponibilidad permanente. Estos
  pueden fallar de manera aleatoria respecto a una FDP, y cuando sucede permanecen fuera de servicio
  una cantidad de tiempo aleatoria que está dada por otra FDP (hasta completar su reparación). Además,
  todos los cargadores reciben mantenimiento preventivo cada cierta cantidad de días (nuestra variable
  de control "TMP"). Estos mantenimientos hacen que se vuelva a calcular el momento de la próxima
  falla de cada uno. De esta forma, simular con distintas frecuencias de mantenimiento permite
  analizar cómo cambian los costos, la disponibilidad de los cargadores, las esperas, el
  arrepentimiento de los usuarios y, en consecuencia, el beneficio del sistema.

---

## Análisis previo (opcional)

### Metodología

**Evento a Evento**

### Clasificación de variables

#### Datos

- **IAS1** (Intervalo entre Arribos en la Semana entre las 08:00 y las 12:59, medido en minutos)
- **IAS2** (Intervalo entre Arribos en la Semana entre las 13:00 y las 19:59, medido en minutos)
- **IAS3** (Intervalo entre Arribos en la Semana entre las 20:00 y las 07:59, medido en minutos)
- **IAF1** (Intervalo entre Arribos en el Fin de Semana entre las 08:00 y las 12:59, medido en minutos)
- **IAF2** (Intervalo entre Arribos en el Fin de Semana entre las 13:00 y las 19:59, medido en minutos)
- **IAF3** (Intervalo entre Arribos en el Fin de Semana entre las 20:00 y las 07:59, medido en minutos)
- **TC** (Tiempo de Carga de un vehículo, medido en minutos)
- **TFC** (Tiempo entre Fallas de un Cargador, medido en minutos)
- **TRC** (Tiempo para Reparar un Cargador, medido en minutos)

#### Control

- **RC** (Recaudación por Carga, medido en $/kWh)
- **TMP** (Tiempo entre Mantenimientos Preventivos, medido en días)
- **PDCE** (Porcentaje de Demanda a Capturar por Estación)

#### Resultado

- **BM(i)** (Beneficio Mensual por franja horaria) [^i-franja]
- **PTO(i)** (Porcentaje de Tiempo Ocioso por franja horaria)
- **PEC(i)** (Porcentaje de Espera en Cola por franja horaria)
- **PPS(i)** (Promedio de Permanencia en el Sistema por franja horaria)
- **PARR(i)** (Porcentaje de Arrepentimiento por franja horaria)
- **PDC(i)** (Porcentaje de Disponibilidad de Cargadores por franja horaria)
- **BMP** (Beneficio Mensual Promedio)
- **PTOP** (Porcentaje de Tiempo Ocioso Promedio)
- **PECP** (Porcentaje de Espera en Cola Promedio)
- **PPSP** (Promedio de Permanencia en el Sistema Promedio)
- **PARRP** (Porcentaje de Arrepentimiento Promedio)
- **PDCP** (Porcentaje de Disponibilidad de Cargadores Promedio)

#### Estado

- **CA(i)** (Cantidad de Autos en la estación (i): los que están cargando más los que esperan en su
  cola única)
- **CD(i)** (Cantidad de Cargadores Disponibles de la estación (i): instalados y en servicio, es decir
  sin contar los que están fallados)
- **CC(i)** (Cantidad de Cargadores por estación)
- **CE** (Cantidad de Estaciones)

### TEF

- **TPI(i)** (Tiempo de Próximo Ingreso por estación)
- **TPC(i)(j)** (Tiempo de Próxima Carga por puesto y estación)
- **TPAE** (Tiempo de Próximo Análisis de Expansión)
- **TPIC** (Tiempo de Próxima Instalación de Cargador)
- **TPCE** (Tiempo de Próxima Construcción de Estación)
- **TPFC(i)(j)** (Tiempo de Próxima Falla de Cargador)
- **TPRC(i)(j)** (Tiempo de Próxima Reparación de Cargador)
- **TPMP(i)** (Tiempo de Próximo Mantenimiento Preventivo por estación)

---

## TEI o Clasificación de eventos (según corresponda)

| Evento | Evento Futuro no Condicionado | Evento Futuro Condicionado | Condición |
|---|---|---|---|
| Ingreso de auto a una estación (i) | Ingreso de auto a una estación (i) | Carga de auto en un cargador de una estación (i) (j) | `R < PDCE / 100 && CA(i) ≤ CD(i)` |
| Carga de auto en un cargador de una estación (i) (j) | – | Carga de auto en un cargador de una estación (i) (j) | `CA(i) ≥ CD(i)` |
| Análisis de Expansión | Análisis de Expansión | Instalación de nuevo cargador (i) | `TPIC = HV && CC(i) < CC_MAX && CARRUM(i) * (RC * ECP - CCP) > CPN` |
| | | Construcción de nueva estación | `TPCE = HV && CE < CE_MAX && PDCE * CE < 100 && CPAACUM * 4 * (RC * ECP - CCP) > CEN` |
| Instalación de nuevo cargador (i) | Falla de un cargador (i) (j) | Carga de auto en un cargador de una estación (i) (j) | `CA(i) ≥ CD(i)` |
| Construcción de nueva estación | Ingreso de auto a una estación (i); Falla de un cargador (i) (j); Mantenimiento preventivo de cargadores (i) | – | – |
| Falla de un cargador (i) (j) | Falla de un cargador (i) (j) | Reparación de un cargador (i) (j) | `TPRC(i)(j) = HV` |
| Reparación de un cargador (i) (j) | – | Carga de auto en un cargador de una estación (i) (j) | `CA(i) ≥ CD(i)` |
| Mantenimiento preventivo de cargadores (i) | Mantenimiento preventivo de cargadores (i) | – | – |

Las dos filas de **Análisis de Expansión** corresponden a un mismo evento: dispara dos eventos
condicionados distintos, cada uno con su condición.

En el **Ingreso de auto a una estación (i)**, `R` es el número aleatorio uniforme en [0, 1) que se
sortea en cada arribo y `PDCE` está expresado en porcentaje. Si `R ≥ PDCE / 100` el vehículo no
ingresa a la estación: es demanda no capturada, no ocupa cargador, no hace cola y no cuenta como
arrepentido en `CARRUM(i)`.

Las condiciones sobre `CA(i)` se evalúan **después de actualizar el vector de estado**, como
corresponde a la metodología Evento a Evento: primero se modifica el estado por el evento actual y
recién entonces se evalúan sus eventos futuros condicionados. En el ingreso, el auto que llega ya
está sumado a `CA(i)`, y por eso hay cargador libre para él si `CA(i) ≤ CD(i)`; en el fin de carga,
el auto que se retira ya está restado, y por eso queda alguien esperando si `CA(i) ≥ CD(i)`. Las dos
condiciones no se solapan en `CA(i) = CD(i)` porque pertenecen a eventos distintos, y cuál de los dos
está ocurriendo lo determina el mínimo de la TEF antes de evaluar cualquier condición.

Los eventos que **suman capacidad** —**Reparación** e **Instalación de nuevo cargador**— disparan una
carga condicionada con la misma condición que el fin de carga y por el mismo motivo: quedó un cargador
libre y hay que ver si alguien lo estaba esperando, con `CD(i)` ya actualizado por el evento. Sin esa
condición, el cargador reparado o instalado queda ocioso al lado de la cola, porque cada fin de carga
habilita una sola carga nueva: se rompe el invariante `autos cargando = min(CA(i), CD(i))` y el
arrepentimiento sobrante se filtra a `CARRUM(i)`, que es lo que decide la instalación del cargador
siguiente. La Instalación agenda además la primera falla del cargador que instala (`TPFC(i)(j)`), que
si no quedaría en `HV` y ese cargador no fallaría nunca.

La **Construcción de nueva estación** no dispara ninguna carga: la estación se crea vacía
(`CA(i) = 0`), o sea que suma capacidad y cola al mismo tiempo, las dos en cero. Lo que sí genera son
los eventos propios de esa estación, que hasta ese momento no existían: su primer arribo `TPI(i)`, la
primera falla de cada uno de sus cargadores `TPFC(i)(j)` y su primer mantenimiento preventivo
`TPMP(i)`. En estas tres filas, `(i)` es la estación involucrada: la del cargador reparado o
instalado, o la que se acaba de construir. La estación se construye con **4 cargadores**, que es la
cantidad que ya supone la condición de expansión al proyectar su facturación con `CPAACUM * 4`.

**Disciplina de cola dentro de la estación:** cada estación atiende con una **única cola FCFS** común a
todos sus cargadores. El auto que espera toma el primer cargador que se libera; no se forma una fila
por cargador ni se elige cargador al ingresar. Por eso el estado es `CA(i)` y no un vector por
cargador: los `CD(i)` primeros autos están cargando, el resto espera en la cola común, y el
arrepentimiento se evalúa sobre esa cola y no sobre la de un cargador en particular. La elección se
apoya en que es la disciplina que se observa en la práctica — en la playa de una estación de servicio
los autos que esperan no pueden apilarse detrás de un cargador ocupado teniendo otro libre — y es la
que las redes de carga están formalizando con listas de espera únicas por estación. Es además la
disciplina de la M/M/c, que es el modelo más usado en la literatura de estaciones de carga y sirve
como referencia para validar el motor.

Esta disciplina supone que: (a) los `CC(i)` cargadores de una estación son homogéneos e
intercambiables, cuando en la realidad los conectores no lo son (CCS, CHAdeMO, AC) y eso generaría una
cola por tipo de conector; y (b) no hay reserva de turno, cuando las apps de las redes de carga ya
muestran disponibilidad en tiempo real y anuncian la reserva — atender por orden de llegada
sobrestima la espera respecto de un sistema con reservas.

---

## Variables Auxiliares mencionadas

- **CARRUM(i)** (Cantidad de Arrepentidos del Último Mes por estación)
- **ECP** (Energía Cargada Promedio)
- **CPAACUM** (Cantidad Promedio de Autos Atendidos por Cargador en el Último Mes)

## Valores Fijos mencionados

- **CC_MAX** (Cantidad de Cargadores por estación MÁXimos)
- **CCP** (Costo por Carga Promedio)
- **CPN** (Costo de un Puesto Nuevo)
- **CE_MAX** (Cantidad de Estaciones MÁXimas)
- **CEN** (Costo de una Estación Nueva)
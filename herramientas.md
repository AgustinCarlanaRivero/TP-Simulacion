# Herramientas y método de trabajo

Qué se adopta para el TP, qué se descarta y **por qué**, para no volver a evaluar lo mismo dentro de
dos meses. Es el complemento de [CLAUDE.md](CLAUDE.md): ahí está el modelo a simular, acá cómo se
trabaja sobre él.

Criterio transversal, distinto del de un proyecto de software: esto es un **entregable académico con
fecha**, escrito por cuatro personas, cuyo producto final es uno o varios notebooks de Colab. Una
herramienta entra solo si el tiempo que ahorra antes de la entrega es mayor que el que cuesta
configurarla. Todo lo que optimiza el trabajo sobre codebases grandes es prematuro acá.

---

## Resumen accionable

| Herramienta / técnica | Veredicto | Estado |
|---|---|---|
| **`colab-mcp`** | Es el eje del entregable | **Verificado** — conecta; ver abajo cómo se destraba |
| **Skill `dataviz`** | Usar **antes** de escribir cualquier gráfico | Disponible, built-in |
| **Validación del motor** (invariantes + caso M/M/c) | Adoptar — es lo que separa "corre" de "está bien" | Pendiente — con el motor |
| **La propuesta como spec externa** (EARS sobre la tabla de eventos) | Adoptar — sin ancla externa, la verificación copia los bugs del motor | Propuesto |
| **Relaciones metamórficas** | Adoptar — es el oráculo que un simulador no tiene | Pendiente — con el motor |
| **Reglas anti-trampa al verificar** | Adoptar en `CLAUDE.md` | **Adoptado** — *Reglas de trabajo* |
| **Lógica pura fuera del notebook** | Adoptar al escribir el motor | Pendiente — habilita todo lo anterior |
| **Réplicas + semilla** | Adoptar — reemplaza el número único por un IC | Pendiente |
| **`WebSearch` / `WebFetch` para los pendientes de datos** | Usar, con fuente y fecha en el notebook | Disponible |
| **Artifact** (+ `artifact-design`, `artifact-diagramming`) | Para el diagrama de eventos y para compartir escenarios | Disponible, built-in |
| **Canario en `CLAUDE.md`** | Adoptar — cuesta una línea | **Adoptado** — *Reglas de trabajo* |
| **Cambios quirúrgicos + objetivo verificable** | Adoptar a mano en `CLAUDE.md` | **Adoptado** — *Reglas de trabajo* |
| **Notebooks en git** (`nbstripout` / `.py` aparte) | Adoptar si el `.ipynb` se versiona | **Instalado** — filtro activo; cada clon lo engancha aparte |
| **Tiers de modelo al delegar** | Adoptar la regla | **Adoptado** — *Reglas de trabajo* |
| **Corridas largas en background** | Técnica, cero setup | Disponible |
| `fewer-permission-prompts`, `find-skills`, memoria de proyecto | Barato, tener a mano | Disponible |
| Awesome Claude Code | Marcador, no se instala | Costo de contexto cero |
| **Ponytail** | **No** — ver abajo | Evaluado |
| Plugin *Claude Code Setup* | No todavía — no hay código que leer | Evaluado |
| Mutation testing (`mutmut`) | No por ahora — la salvedad está abajo | Evaluado |
| Gherkin con runner (`behave` / `pytest-bdd`) | No — sí la tabla de casos de borde | Evaluado |

---

## Lo verificado del entorno

Corrido en esta máquina, no supuesto:

| Qué | Estado |
|---|---|
| `python` | 3.13.7, en el PATH |
| `node` / `npx` | v22.19.0 / 10.9.3, en el PATH |
| `uv` / `uvx` | Instalados por pip, **fuera del PATH** — viven en `%APPDATA%\Python\Python313\Scripts` |
| `.claude/` | No existe todavía |

Por eso el [.mcp.json](.mcp.json) **no** invoca `uvx` sino `python -m uv tool run`: el ejecutable de
`uv` no está en el PATH, pero el paquete sí es importable desde el `python` que sí está. La primera
versión parcheaba `env.PATH` con `%APPDATA%\Python\Python313\Scripts`, que ata el archivo versionado
a la versión de Python y al sistema operativo de una máquina; `python -m uv` no. Contrapartida: `uv`
tiene que estar instalado **con pip** (`pip install uv`), no con el instalador standalone. No hace
falta tocar el PATH del sistema ni crear un virtualenv.

---

## `colab-mcp` — el eje, y cómo se destraba

El server está declarado a scope de proyecto en [.mcp.json](.mcp.json), así que se clona el repo y
viene puesto. Dos cosas que conviene saber antes de la primera sesión de notebook:

1. **Arranca con una sola herramienta expuesta**: `open_colab_browser_connection`, que abre la sesión
   de navegador contra Colab y recién ahí *destraba las herramientas de edición del notebook*. O sea
   que el costo de contexto del MCP es casi nulo mientras no se lo use — no es de los conectores que
   cobran veinte descripciones por sesión.
2. **El orden importa**: abrir la conexión cuando se va a trabajar sobre el notebook, no al empezar
   cualquier sesión. Para discutir el modelo, leer la propuesta o escribir documentación, no hace falta.

Consecuencia práctica para la organización del trabajo: conviene concentrar las sesiones de edición
del notebook en vez de intercalarlas con las de discusión.

---

## Gráficos: `dataviz` antes de la primera línea de `matplotlib`

El TP tiene bastante más gráfico del que parece: seis histogramas de IA con su FDP ajustada
superpuesta, el ajuste de `TC`, la regresión energía–tiempo rehecha sobre los pares reales, la curva
logística contra la serie del parque EV, y después las series de resultados por franja (`BM`, `PTO`,
`PEC`, `PARR`) y la comparación entre escenarios de `RC` / `TMP` / `PDCE`.

La skill `dataviz` se carga **antes** de escribir el código del gráfico, no después para arreglarlo.
Dos ajustes propios de este entregable, que la skill no puede saber:

- **El informe se imprime o se exporta a PDF.** La paleta tiene que sobrevivir a blanco y negro: las
  series se distinguen por marcador y trazo, no solo por color.
- **Consistencia entre notebooks.** Si el trabajo se parte en dos (datos/FDPs y motor/experimentación),
  los colores de cada franja horaria tienen que ser los mismos en los dos. Conviene fijar el mapeo
  franja → color una vez, en una celda de constantes, y no elegirlo gráfico por gráfico.

---

## Validación del motor — la sección que más rinde

Un motor evento a evento **siempre corre y siempre devuelve números**. Ese es el problema: un bug en
la TEF o en las condiciones de expansión no tira una excepción, produce resultados plausibles y
equivocados. Lo de abajo está en el orden en que conviene hacerlo.

### El orden importa: primero el motor, después la verificación

Acá el código va primero — se escribe el motor y después se lo verifica — y ese orden tiene una
trampa conocida: si los chequeos se derivan **leyendo el motor ya escrito**, terminan siendo una foto
del código, bugs incluidos, en vez de una verificación del modelo. Quedan en verde y no prueban nada.
Es el mismo defecto que tiene pedirle los tests a un agente después de implementar: escribe asserts
que pasan contra lo que ve.

La defensa es un **ancla externa al código**, y acá ya existe: **la propuesta**. La tabla de eventos y
condiciones es, literalmente, la especificación del modelo. De ahí la regla: los invariantes y los
asserts salen de esa tabla, **nunca de leer el motor**. Si la tabla no dice qué pasa en algún caso,
eso es un hueco de la propuesta y se resuelve ahí (celda de supuesto + actualizar la propuesta, como
ya pide `CLAUDE.md`), no eligiendo en silencio lo que el código ya hacía. Las formas concretas de
violar esta regla están listadas más abajo, en *Reglas anti-trampa al verificar*.

### EARS sobre la tabla de eventos — el caso que no está escrito

EARS (*Easy Approach to Requirements Syntax*) restringe el lenguaje de un requisito a unos pocos
patrones con las cláusulas siempre en el mismo orden. Cada fila de la tabla de eventos se reescribe
como: **CUANDO** `<evento>`, **SI** `<condición>`, **ENTONCES** el sistema debe `<respuesta>`.

El valor no es el formato, es lo que expone: **cada `SI` obliga a escribir también la rama falsa**.
"Si `CA(i)(j) ≤ CC(i)`, entonces carga en el cargador" no dice qué pasa cuando la condición no se
cumple — y ahí es justo donde vive el arrepentimiento, que `CLAUDE.md` ya marca como hardcodeado y
sin justificar. Pasar las seis filas de la tabla por esta plantilla lleva media hora y devuelve la
lista de condiciones que faltan definir, antes de que se conviertan en decisiones tomadas por
descuido adentro del motor.

Tres huecos típicos de un modelo evento a evento que esta pasada saca a la luz:

- **La rama falsa** de cada condición, que la tabla no escribe.
- **Los empates en el mínimo de la TEF**: dos eventos con el mismo `T`. El orden de desempate cambia
  los resultados y casi nunca está especificado. El caso concreto y probable acá es el Análisis de
  Expansión cayendo en el mismo minuto que un arribo o que una reparación.
- **Los bordes del calendario**: el cambio de franja a las 20:00, el paso de viernes a sábado, y el
  fin de mes que dispara el Análisis de Expansión.

Conviene hacer esta pasada como **auditoría explícita de la propuesta**: pedir que se la lea buscando
qué caso no está definido, en vez de asumir que está completa porque fue aprobada. Es una vez, y lo
que devuelve alimenta directo la lista de pendientes de `CLAUDE.md`.

### Etapa 0 — sacar la lógica pura del notebook

En Colab todo tiende a un notebook monolítico con globals, que es exactamente el defecto que
`CLAUDE.md` marca del TP 4 (~20 globals sueltos y `calculo_resultados` que solo imprime). Sin esta
separación nada de lo que sigue se puede automatizar, y hay que decidirlo **al escribir el motor, no
después**.

Lo que es lógica pura y merece ser una función sin estado global:

- `rvs_truncado(dist, data)` — muestreo por transformada inversa acotado al rango observado.
- El mapeo `T` (minutos desde el inicio) → `(franja, tipo de día)` y de ahí → qué FDP de IA usar.
- El factor logístico `P(t) = K / (1 + A·e^(−r·t))` y cómo escala el IA generado.
- La política de elección de estación (`PDCE`) y de cargador dentro de la estación.
- Las dos condiciones de expansión de la tabla de eventos (cargador nuevo y estación nueva).
- El cálculo de resultados a partir de los acumuladores: `BM`, `PTO`, `PEC`, `PPS`, `PARR`, `PDC`.

Forma concreta: un `simulacion.py` en el repo, importado desde el notebook. Da tres cosas gratis —
diffs legibles en git, pruebas que corren localmente con `pytest` sin abrir Colab, y la posibilidad de
que dos integrantes toquen el motor sin pisarse.

### Etapa 1 — invariantes adentro del motor

Baratos, se escriben una vez y corren en cada corrida (con un flag para apagarlos en las corridas
largas). Cada uno atrapa una familia entera de bugs:

- El reloj **nunca retrocede**: el `T` nuevo es siempre ≥ al anterior.
- Ningún TEF queda en el pasado después de avanzar el reloj.
- **Conservación de autos**: arribos generados = atendidos + arrepentidos + en cola + en carga.
- `CA(i)(j) ≥ 0` siempre, y un cargador en falla no atiende a nadie.
- `CC(i) ≤ CC_MAX` y `CE ≤ CE_MAX` después de cada Análisis de Expansión.
- Por cargador: tiempo ocioso + tiempo ocupado + tiempo fuera de servicio ≈ tiempo simulado.

### Etapa 2 — el caso degenerado contra la teoría

Es el único chequeo que dice que el motor está **bien**, no solo que es consistente consigo mismo. Se
apaga todo lo que el TP agrega y se lo deja en un M/M/c: IA y TC exponenciales, una estación, `c`
cargadores, sin fallas, sin mantenimiento, sin expansión, sin arrepentimiento y sin factor logístico.
Ahí `L`, `Lq`, `W` y `Wq` tienen fórmula cerrada (Erlang-C) y el simulador tiene que reproducirlas
dentro del error de muestreo.

Vale la pena por dos motivos: encuentra errores de contabilidad que ningún invariante ve, y es
material publicable en el informe — la sección de validación del modelo se escribe sola.

### Etapa 3 — relaciones metamórficas

El M/M/c valida el caso degenerado; el modelo completo no tiene fórmula cerrada contra la cual
comparar, y ese es el problema clásico del software sin oráculo. La salida estándar es no verificar el
resultado absoluto sino **cómo tiene que cambiar el resultado cuando se cambia una entrada**. Se corre
el modelo dos veces con la misma semilla, moviendo un solo parámetro, y se verifica la dirección.

Las que este modelo permite, todas derivables de la propuesta y no del código:

- Más cargadores (mayor `CC(i)`), a demanda igual ⇒ `PEC` y `PARR` **no aumentan**, `PTO` **no baja**.
- Mayor demanda (factor logístico más grande, o sea IA más chicos) ⇒ `PARR` y `PEC` **no bajan**.
- `RC` mayor con todo lo demás igual ⇒ `BM` **no baja** (la recaudación es lineal en `RC` y los costos
  no dependen de él).
- `TMP` más largo (menos mantenimiento preventivo) ⇒ `PDC` **no sube**.
- `TFC` más largo (fallas más espaciadas) ⇒ `PDC` **no baja**.

Es la técnica que más bugs de contabilidad encuentra por hora invertida, porque no hace falta saber
cuál es el número correcto: alcanza con saber para qué lado tiene que moverse. Y cuando una de estas
relaciones se rompe, el bug está acotado al parámetro que se movió.

Dos advertencias. La primera: exige **usar la misma secuencia de aleatorios** en las dos corridas, o
la diferencia observada es ruido; con muy pocas réplicas, una relación puede "romperse" solo por
varianza, así que se comparan medias de varias corridas. La segunda: son desigualdades débiles a
propósito (*no aumenta*, no *disminuye*), porque en zonas de saturación o con topes de `CC_MAX`
alcanzados el efecto puede ser nulo.

### Etapa 4 — réplicas y semilla

Una corrida es una realización de un proceso estocástico, no un resultado. Dos reglas:

- **Semilla fija** para las corridas que se reportan en el informe, así los cuatro integrantes ven los
  mismos números y el corrector también.
- **N réplicas con semillas distintas** por escenario, y reportar media ± intervalo de confianza, no
  un número pelado. Comparar escenarios de `RC` / `TMP` / `PDCE` con una sola corrida cada uno es
  comparar ruido.

Además, el motor tiene que **devolver** las métricas (no imprimirlas) para que las réplicas se puedan
barrer en un loop. Eso ya está pedido en `CLAUDE.md`; acá está el motivo concreto.

### Etapa 5 — `pytest`, y `hypothesis` donde rinde

Recién cuando exista el `.py`. El property-based testing con `hypothesis` tiene dos objetivos ideales
en este modelo, los dos función pura con dominio acotado:

- `rvs_truncado`: para cualquier distribución ajustada y cualquier muestra, el valor devuelto cae
  siempre dentro de `[min(data), max(data)]`.
- El mapeo hora → franja: las tres franjas particionan las 24 horas sin huecos ni solapes, para
  cualquier `T`. El borde 20:00–07:59 cruza la medianoche y es donde va a estar el bug.

Lo que hace valiosa a la biblioteca por encima de escribir casos a mano es el **shrinking**: cuando
encuentra una entrada que rompe la propiedad, la reduce automáticamente al caso mínimo que falla, así
el bug llega ya minimizado.

Si las pruebas llegan a correr en pocos segundos, un hook `Stop` que las ejecute vale la pena (se
configura con la skill `update-config`): a diferencia de una regla escrita en `CLAUDE.md`, que el
modelo puede ignorar, el hook se ejecuta siempre. Tres detalles no negociables al escribirlo, porque
los tres se descubren de la peor manera:

1. Chequear `stop_hook_active` al entrar y salir sin hacer nada si ya viene de un bloqueo — sin eso,
   loop infinito.
2. Devolver `decision: "block"` para que la tarea no se cierre con las pruebas en rojo.
3. Mandar **solo las fallas**, recortadas. Volcar la salida entera de `pytest` satura el contexto y el
   agente pierde el hilo, que es justo lo contrario de lo que se buscaba.

Antes de que exista el `.py` no hay nada que enganchar.

---

## Los pendientes de datos: `WebSearch` / `WebFetch` con disciplina de fuente

`CLAUDE.md` deja abiertos varios datos que no están en el dataset del TP 4: la serie del parque
EV/PHEV de CABA, `TFC` y `TRC`, tarifas pico/valle, `CPN`, `CEN`, `CCP`, y el porcentaje de usuarios
que carga en domicilio. Todos son buscables.

La regla que hace que esto no rompa el "no inventar datos": **cada dato traído de afuera entra al
notebook con su fuente y su fecha de consulta en la celda markdown de al lado**. Si no se consigue el
dato, se adopta un supuesto y se lo marca como supuesto — que es lo que ya pide `CLAUDE.md`. Lo que no
puede pasar es que un número aparezca en el código sin poder decir de dónde salió: es lo primero que
pregunta un corrector.

Conviene además distinguir en el notebook las tres categorías, porque no valen lo mismo: **dato
medido** (del dataset), **dato de fuente pública** (con cita) y **supuesto** (con justificación).

---

## Artifact — para el diagrama de eventos y para el equipo

Dos usos concretos, ninguno de los cuales reemplaza al entregable:

- **El diagrama del modelo.** Un TP de simulación se evalúa en buena medida por si el modelo está bien
  planteado: eventos no condicionados, condicionados y sus condiciones. Ese diagrama en una página
  HTML (con `artifact-diagramming`) se lee mucho mejor que en una celda markdown, y se puede pasar al
  informe como imagen.
- **Compartir resultados con los otros tres.** Una página con la comparación de escenarios se manda
  por link y se abre en el celular; mandar el `.ipynb` obliga al otro a ejecutarlo.

El límite es claro: **el entregable es el notebook**. El artifact es vista, no fuente de verdad —
misma regla que ya rige para la propuesta.

---

## Reglas para el `CLAUDE.md` (cuestan líneas, no instalaciones)

Las cinco ya están escritas en [CLAUDE.md](CLAUDE.md), sección *Reglas de trabajo*. Lo que queda
acá es el porqué de cada una, que no hace falta cargar en cada sesión.

### Canario

Una línea pidiendo un gesto fijo en cada respuesta; cuando desaparece, es señal de que las
instrucciones del archivo dejaron de seguirse (contexto saturado o deriva) y conviene abrir sesión
nueva. Detecta algo que ningún comando reporta: `/context` da los tokens ocupados, el canario dice si
las instrucciones **todavía se cumplen**, que es la pregunta que importa.

Como el repo es de cuatro personas, conviene un canario neutral en vez de un nombre propio. Por
ejemplo: *empezar cada respuesta nombrando en qué sección del TP se está trabajando*. Barato,
específico del proyecto y visible de inmediato.

### Cambios quirúrgicos (adaptado a notebooks y a cuatro autores)

- No reordenar, renumerar ni reformatear celdas adyacentes al cambio.
- No "mejorar" la redacción de celdas markdown que escribió otro integrante.
- Si aparece código muerto del TP 4, **mencionarlo — no borrarlo**.
- Limpiar solo lo que el propio cambio dejó huérfano (imports, variables sin uso).

En un notebook compartido esto pesa más que en un repo de código: un reformateo cosmético cambia el
`.ipynb` entero y hace incomparable el diff.

### Objetivo verificable antes de arrancar

Traducir la tarea a algo que se pueda chequear. "Ajustar las FDPs de IA" no es un objetivo; "las seis
FDPs elegidas, con su `ks_statistic` reportado y su celda de justificación" sí. Para tareas de varios
pasos, plan corto con la verificación de cada paso.

### Reglas anti-trampa al verificar

Un agente entrenado con refuerzo optimiza por "que el chequeo pase", no por "que el modelo esté
bien" — y las formas de hacer trampa en un TP de simulación son propias del dominio, así que conviene
nombrarlas antes de que aparezcan:

- **Los asserts salen de la propuesta, no de leer el motor.** Si la propuesta no cubre el caso, se
  frena y se pregunta; no se copia lo que el código ya hace.
- **Prohibido ensanchar la tolerancia hasta que pase.** Si la comparación contra M/M/c no cierra, el
  sospechoso es el motor, no el margen de error.
- **Prohibido elegir la semilla que da el resultado lindo.** La semilla se fija una vez y antes de
  ver los resultados; si un escenario solo funciona con una semilla, no funciona.
- **Prohibido bajar la cantidad de réplicas** para que el intervalo de confianza deje de contradecir
  la conclusión.
- **Ningún chequeo se da por bueno porque "no tiró excepción"**: tiene que afirmar sobre un valor o
  sobre un cambio de estado. Un motor que corre entero y devuelve basura es el escenario esperado, no
  el raro.
- **Prohibido silenciar con `try/except`** una excepción que aparece durante una corrida para que
  termine.

Si en algún momento conviene independencia de criterio —que quien valida no sea quien escribió el
motor—, eso se resuelve delegando la validación a un subagente **sin el contexto de cómo se
implementó**, con la consigna de reportar solo lo observado. Es opcional y va después de que las
etapas 1 a 3 ya corran.

### Tiers de modelo al delegar

Cuando se delega trabajo a un subagente, `model` explícito siempre: `haiku` para lo mecánico
(segmentar el dataset, renombrar variables al esquema de siglas de la propuesta, extraer tablas),
`sonnet` de default, `opus` para lo genuinamente difícil (el motor de eventos, la validación contra
M/M/c). Ante la duda entre dos tiers, el más barato y escalar si falla; una tarea que ya se sabe
difícil va al de arriba de entrada.

---

## Notebooks en git

El `.ipynb` versionado genera diffs ilegibles: los outputs y la metadata de ejecución cambian en cada
corrida, y `TP 4 Simu.ipynb` ya pesa 266 KB en el repo. Con cuatro personas editando, cualquier merge
sobre el notebook es un conflicto.

Dos medidas, en orden de rendimiento por esfuerzo:

1. **El motor vive en un `.py`.** Es la misma decisión de la Etapa 0 de validación, y resuelve el 80%
   del problema: lo que se mergea de verdad es código Python, no JSON con imágenes adentro.
2. **`nbstripout`** como filtro de git, si igual se quiere versionar el notebook: limpia los outputs al
   commitear y deja el diff en el texto de las celdas. Se instala una vez por clon.

   Hecho: `pip install nbstripout` + `python -m nbstripout --install --attributes .gitattributes`. El
   `.gitattributes` queda versionado (`*.ipynb filter=nbstripout diff=ipynb`), pero la definición del
   filtro vive en `.git/config`, que no se versiona: **cada integrante corre los dos comandos en su
   clon**. Si no lo hace, git ignora el atributo sin avisar y commitea los outputs igual. Medido sobre
   `TP 4 Simu.ipynb`: 266 KB en disco → 26 KB de contenido versionado.

Colab sigue siendo el entorno de referencia; esto es solo cómo entra el resultado a git.

---

## Corridas largas

El horizonte del TP tiene que cubrir varios años simulados, y encima multiplicado por réplicas y por
escenarios. Cuando una corrida deje de ser instantánea, va **en background** en vez de bloquear la
sesión: se lanza, se sigue trabajando en otra cosa y llega el aviso al terminar. No requiere instalar
nada. Para eso conviene también que el motor acepte un parámetro de horizonte y un flag para apagar
los invariantes, así el mismo código sirve para el chequeo rápido y para la corrida del informe.

---

## Barato, tener a mano

| Qué | Para qué |
|---|---|
| `fewer-permission-prompts` | Si los permisos empiezan a molestar, arma la allowlist mirando el historial real |
| `find-skills` | Buscar una skill cuando aparezca una necesidad concreta, no antes |
| Memoria de proyecto | Decisiones que no van al repo (por qué se descartó un enfoque, qué probó cada integrante) |
| Awesome Claude Code | Índice curado, sin CLI ni instalación: catálogo para cuando haya una falla concreta que resolver |
| `/code-review` | Sobre el `.py` del motor antes de entregar. Ojo: trabaja sobre el diff de git, así que no ve lo que solo vive en Colab |

---

## Evaluadas y descartadas

### Ponytail — no

Plugin de estilo "senior vago" (YAGNI, biblioteca estándar primero, cero abstracciones no pedidas),
con hooks que inyectan su ruleset en cada sesión y en cada subagente. Buena herramienta; acá no entra
por tres motivos:

- **Es contexto fijo por sesión, en un proyecto con fecha de entrega.** El ruleset se paga siempre,
  incluso en las sesiones de discusión del modelo, que son mayoría.
- **El riesgo que ataca ya está cubierto.** `CLAUDE.md` es explícito sobre qué reusar del TP 4 y qué
  cambiar; el stack está cerrado (`numpy`, `pandas`, `scipy`, `fitter`, `matplotlib`) y no hay
  decisiones de arquitectura donde sobre-diseñar.
- **El ruleset está en inglés** y este proyecto exige español rioplatense en todo el contenido. No
  rompe nada, pero suma ruido en cada sesión.

Si en algún momento el motor empieza a crecer en capas, lo que corresponde es agregar dos líneas al
`CLAUDE.md`, no instalar un plugin.

### Plugin *Claude Code Setup* — no todavía

Es oficial de Anthropic y **read-only**: mira el codebase y sugiere hooks, skills y subagentes sin
tocar nada. El problema es el de siempre: detecta el stack leyendo código, y hoy el repo tiene un
notebook del TP anterior y dos markdown. Lo que devuelva va a ser genérico.

Momento correcto: cuando exista el `simulacion.py`. Se corre una vez, se anota lo que sirva acá y se
desinstala.

### Mutation testing (`mutmut`) — no por ahora, y con una salvedad técnica

Es la única forma objetiva de saber si las pruebas **detectarían un bug** en vez de solo correr:
introduce mutaciones en el código (`>` por `>=`, `+` por `-`, borrar una línea) y mide qué porcentaje
de esos mutantes hace fallar alguna prueba. Ataca de frente el problema del code-first, así que la
tentación es directa.

Queda afuera por costo: corre la suite entera una vez por mutante, y lo que hay que verificar acá se
cubre antes y más barato con el caso M/M/c y las relaciones metamórficas, que además son material del
informe. Con fecha de entrega, esa comparación no está cerca.

La salvedad, por si alguien lo intenta igual: **apuntarlo solo a las funciones puras** de
`simulacion.py`, nunca al loop de eventos. El mutation testing necesita una suite rápida y
determinística, y una corrida de simulación no es ninguna de las dos cosas — los mutantes se
cuelgan por timeout y el resultado no distingue un mutante sobreviviente del ruido estocástico.

### Gherkin con runner (`behave`, `pytest-bdd`) — no, pero sí la tabla

Escribir los escenarios en `Given / When / Then` y ejecutarlos con un runner agrega una capa de
pegamento (los *step definitions*) que en un TP de una sola persona por módulo no se paga: el
formato existe para que lo lea alguien que no programa, y acá los cuatro lectores programan.

Lo que sí se usa es la idea del `Scenario Outline`: **una tabla de casos con sus valores de borde**,
que es lo que empuja a probar `0`, `1`, el máximo y el máximo + 1 en vez de tres casos del medio. Para
este modelo, esa tabla son las condiciones justo en el umbral: la de expansión exactamente igual a
`CPN`, `CC(i)` en `CC_MAX`, `CE` en `CE_MAX`, la cola vacía, todos los cargadores en falla a la vez, y
el minuto exacto del cambio de franja. Se escribe como una lista de casos en el notebook o como
parametrización de `pytest`, sin runner de por medio.

### Pipelines spec-driven armados (ATDD, DAE, Spec Kit) y Ralph Wiggum — no

Los kits que encadenan spec → tests → mutation testing → iteración resuelven un problema real en
proyectos largos, y los loops autónomos sirven para refactors grandes o tandas nocturnas. Los dos
suponen un ciclo de desarrollo que acá no existe: el entregable es un notebook, la lógica verificable
son unas pocas funciones puras, y el costo en tokens de un loop desatendido no tiene contrapartida.

---

## Cómo evaluar lo que venga

Cinco preguntas, en este orden, para no perder una tarde con cada herramienta que aparezca:

1. **¿Entiende Python / notebooks?** Buena parte del ecosistema está hecho para apps web en JS/TS.
   Descarta la mitad en treinta segundos.
2. **¿Se pisa con algo que ya está?** `CLAUDE.md` y `colab-mcp` ya cubren el proceso y el entorno. Lo
   que se superpone se paga dos veces.
3. **¿El problema que resuelve ya lo tengo?** Con un notebook y un `.py`, todo lo que optimiza el
   trabajo sobre repos grandes es prematuro.
4. **¿Qué cuesta tenerlo prendido?** Un índice de GitHub cuesta cero; un plugin deja su descripción en
   contexto en cada sesión. Un MCP puede costar poco o mucho — `colab-mcp` es el caso barato porque
   expone una sola herramienta hasta que se lo usa.
5. **¿Lo que instala es lo que pediste?** Un "sí" a una skill puede venir con once que no y con un
   conector que no sabías que había. Mirar qué trae el paquete antes de aceptar el veredicto.

Y la pregunta que manda sobre las cinco: **¿se termina de configurar antes de la fecha de entrega, y
sobra tiempo?** Si no, no va.

Regla de higiene: las estrellas que muestran los agregadores no son las de GitHub. Verificar contra
`api.github.com/repos/<owner>/<repo>` antes de citar un número — y no dejar que el número decida.

---

## Nota sobre el costo de instalar

Cada plugin habilitado deja su descripción en contexto de forma permanente. Instalar "por las dudas"
se paga en cada sesión, hasta la entrega. La regla es la misma de siempre: **sumar una herramienta
después de una falla concreta, no antes de una imaginada.**

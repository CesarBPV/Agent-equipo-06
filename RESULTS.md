# RESULTS — Equipo 06 · Dominio `fees` (Tarifario)

Libreta del equipo: mediciones y decisiones. El `README.md` del proyecto no se toca.

- Fecha de medición: 2026-11-02
- Backend: Microsoft Foundry real, `FOUNDRY_AUTH_PROVIDER=entra_id` (sin keys)
- Modelo: `gpt-5-mini` (resuelto como `gpt-5-mini-2025-08-07`)
- Código: tag `t1-s2` (rama `equipo-06`), `MAX_TURNS=10`, `FOUNDRY_MAX_RETRIES=2`, `FOUNDRY_TIMEOUT_S=30`
- Cada conversación se midió en una sesión nueva (sin historial), una sola llamada al modelo por turno.

## 1. Latencia y tokens de las 5 conversaciones

| Mensaje | Intención | latency_ms | prompt_tokens | completion_tokens |
|---|---|---:|---:|---:|
| Hola, buenos días | greeting | 1910 | 380 | 124 |
| ¿Cuánto cobran por retirar en un cajero de otro banco? | product_question | 7048 | 389 | 767 |
| Quiero ver las comisiones que me cobraron este mes | data_query | 7747 | 388 | 803 |
| Quiero que me devuelvan una comisión cobrada dos veces | sensitive_action | 5968 | 389 | 595 |
| ¿A cuánto está el dólar hoy? | out_of_scope | 3254 | 384 | 263 |
| **Promedio** | | **5185** | **386** | **510** |

Las 5 intenciones fueron clasificadas correctamente por el modelo.

### Lectura de los datos
- `prompt_tokens` es casi constante (~385): son el system prompt de `fees.md` más el mensaje. El system prompt pesa unos 370 tokens y viaja en cada llamada.
- `completion_tokens` varía de 124 a 803 y manda sobre la latencia. Latencia y tokens de salida van juntos: 124 tokens → 1.9 s; 803 tokens → 7.7 s (~9 ms por token de salida).
- La salida es entre 1,5 y 2 veces la entrada en promedio (510 frente a 386) y llega a más de 2 veces en preguntas de contenido. `gpt-5-mini` es un modelo de razonamiento: parte de esos tokens de salida son tokens internos de razonamiento que no se ven pero se cobran.

### Comparación con `gpt-5-nano`
Mismas 5 conversaciones, mismo código y mismo `.env`, cambiando solo `FOUNDRY_DEPLOYMENT=gpt-5-nano` por variable de entorno (resuelto como `gpt-5-nano-2025-08-07`). Una sola corrida por modelo.

| Mensaje | Intención | mini ms | nano ms | mini compl. | nano compl. |
|---|---|---:|---:|---:|---:|
| Hola, buenos días | greeting | 1910 | 5570 | 124 | 311 |
| Cajero de otro banco | product_question | 7048 | 6513 | 767 | 1130 |
| Ver comisiones del mes | data_query | 7747 | 5460 | 803 | 789 |
| Devolver comisión duplicada | sensitive_action | 5968 | 6663 | 595 | 949 |
| Dólar hoy | out_of_scope | 3254 | 6310 | 263 | 1017 |
| **Promedio** | | **5185** | **6103** | **510** | **839** |

- `prompt_tokens` fue idéntico (380–389): lo que cambia es la salida.
- Ambos modelos clasificaron las 5 intenciones correctamente.
- nano **no fue más rápido**: promedio 6.1 s frente a 5.2 s, y generó ~65% más tokens de salida (839 frente a 510). nano también razona, y en mensajes simples (saludo, fuera de alcance) razonó mucho más que mini (1017 frente a 263 tokens en out_of_scope).
- Calidad de redacción (lectura manual, 1 corrida): en `sensitive_action`, nano dijo "No puedo realizarla aún", más alineado con la regla del prompt que el "abriré la gestión" de mini. En `data_query`, nano no ofreció enviar un detalle que no puede enviar.
- Cuidado: con una sola corrida por modelo la latencia varía entre llamadas; no es evidencia estadística. Sirve para formular la hipótesis, no para decidir.

## 2. Comprobaciones del Gate

### 8.1 Memoria por sesión — OK
Sesión nueva: "Hola, me llamo Ana" y luego "¿Cómo me llamo?" en la misma sesión → el modelo responde que el cliente se llama Ana. La misma pregunta en otra sesión → no lo sabe. Las sesiones están aisladas.

El historial viaja completo en cada llamada: `prompt_tokens` pasó de 382 (turno 1) a 435 (turno 2), unos +53 tokens por el turno anterior.

### 8.3 Timeout → 504 — OK
Con `FOUNDRY_TIMEOUT_S=0.01` y `FOUNDRY_MAX_RETRIES=0`: `Error 504 (timeout): Foundry no respondió a tiempo`.

### 8.3 Sin sesión de Azure → 502 — OK
Tras `az logout`: `Error 502 (upstream): no se pudo obtener token de Entra ID (¿hiciste az login?)`. El 429 no se puede forzar sin agotar la cuota del equipo; lo cubre `test_agent_errors_map_to_http`.

### Tests — OK
`uv run pytest` → 32 passed.

## 3. Observaciones de calidad (para sesiones futuras)
- **data_query**: el agente ofrece "preparar un detalle y enviarlo", pero hoy no tiene ninguna herramienta ni acceso a datos del cliente. Promete algo que no puede hacer. Se resuelve en T3 con las tools MCP.
- **sensitive_action**: el agente dice "abriré la gestión". El prompt le pide no afirmar que la acción ya se hizo; conviene vigilar esta redacción. La aprobación humana real llega en la S8.
- **product_question**: no da montos porque no tiene el tarifario (correcto: no inventa). Responderá con cifras reales cuando exista RAG (S3 y S4).

## 4. Trade-offs de modelo y costo (borrador para discutir en el equipo)

> Borrador generado a partir de los datos de arriba. El gate pide que el equipo los explique con sus palabras: revisarlos y ajustarlos.

### Trade-off 1 — `gpt-5-mini` o `gpt-5-nano` para clasificar intención
- A favor de un modelo más chico: clasificar en 5 intenciones es simple, y un modelo menor suele costar menos por token.
- Lo que midieron: nano clasificó igual de bien (5 de 5), pero **no fue más rápido** (6.1 s frente a 5.2 s de promedio) y produjo ~65% más tokens de salida (839 frente a 510). Como la salida se cobra y manda sobre la latencia, el menor precio por token de nano podría compensarse con más tokens; hay que cruzarlo con la tarifa real de cada deployment.
- Con 5 casos y una corrida no hay evidencia para cambiar de modelo. Y cada turno hace dos trabajos en un mismo JSON (clasificar y redactar), no solo clasificar. La decisión debe tomarse con el golden set de S5, con varias corridas por caso.

### Trade-off 2 — `MAX_TURNS` y dónde se va realmente el costo
- Cada turno del historial agrega unos 50 tokens de entrada en conversaciones cortas, y el system prompt fijo pesa ~370. Pasar de `MAX_TURNS=10` a `4` ahorra tokens de entrada solo cuando la conversación ya es larga, y quita contexto útil (el cliente que menciona la comisión de hace 6 turnos).
- En estos datos, la **salida** pesa más que la entrada (510 frente a 386 de promedio) y es la que determina la latencia. Recortar el historial ayuda menos que reducir los tokens de razonamiento o de respuesta.
- Idea derivada: usar un modelo chico para `greeting` y `out_of_scope` y uno mayor solo para el resto. Hoy esas dos intenciones son las más baratas y rápidas igual (1.9 s y 3.3 s), así que el ahorro real sería pequeño; se evaluaría con tráfico real.
- El costo mensual exacto depende del tráfico esperado y de la tarifa del deployment, datos que no tenemos todavía.

## 5. Estado del Gate T1

- [x] `/chat` responde contra Foundry con output válido (intent, answer, usage)
- [x] `uv run pytest` → 32 passed
- [x] Memoria por sesión comprobada (paso 8.1)
- [x] Timeout → 504 y sin sesión de Azure → 502 comprobados (paso 8.3)
- [x] Tabla de latencia y tokens de las 5 conversaciones en `RESULTS.md`
- [ ] El equipo explica 2 trade-offs de modelo y costo (borrador en la sección 4, falta la discusión del equipo)
- [x] `.env` sin ninguna key: `FOUNDRY_AUTH_PROVIDER=entra_id` y sin línea `FOUNDRY_API_KEY`

## 6. Sesión 3 · Experimento con el tamaño de chunk (paso 11)

Dominio `fees`, 12 documentos, mismo chunker y mismo corpus. Solo cambia `INGEST_CHUNK_MAX_CHARS`.

| `INGEST_CHUNK_MAX_CHARS` | Chunks | Largo medio (car.) | Máximo (car.) |
|---:|---:|---:|---:|
| 1500 | **29** | 243 | 677 |
| 250 | **38** | 206 | 572 |

El valor de 1500 coincide con los 29 chunks que anuncia la guía para fees. Con 250 hay 9 chunks más (+31%).

Documentos que se parten más con 250 (1500 → 250): `transfer-fees.csv` 1 → 4, `credit-cards-fees.pdf` 3 → 5, `fee-refund-policy.docx` 3 → 4, `additional-services-fees.docx` 2 → 3, `savings-accounts-fees.html` 2 → 3, `transfer-fees-2024.csv` 1 → 2. El resto no cambia.

### Qué se gana: precisión
Cada chunk habla de una sola cosa y su embedding representa mejor ese tema. Ejemplo: en `fee-refund-policy.docx`, la sección "Cómo se solicita" se parte en un chunk sobre cómo pedir la devolución (app, teléfono o agencia) y otro sobre que el asistente no la ejecuta y pide confirmación. Una pregunta sobre uno de los dos temas encuentra un chunk más puro. En `transfer-fees.csv`, 4 chunks en lugar de 1 dan más probabilidad de recuperar solo las filas relevantes.

### Qué se pierde: contexto
Una regla y su excepción pueden quedar en chunks distintos. Ejemplos reales de este corpus con 250:
- **Regla separada de su excepción.** En `credit-cards-fees.pdf`, "Reposición de tarjeta por pérdida o deterioro: S/ 20" queda en un chunk y "Por robo o fraude, la primera reposición del año no tiene costo" en otro. Si el retrieval trae solo el primero, el agente diría que la reposición cuesta S/ 20 también en caso de robo, lo cual es incorrecto.
- **Cifra separada de su etiqueta.** En la tabla de membresías de la misma fuente, la fila de Platinum se corta: el chunk 1 termina en "Platinum, Membresía anual: S/ 480, Exoneración:" y su valor ("consumo anual de S/ 25 000") queda al inicio del chunk 2. Para un tarifario, con cifras exactas, esto es grave.

### Otras observaciones
- Con 1500 el largo medio ya es 243: las secciones de este corpus son cortas, así que el efecto de bajar a 250 se concentra en pocos documentos (6 de 12).
- El límite no es estricto: con 250 hay chunks de hasta 572 caracteres. Probablemente por el camino de títulos que se antepone y por líneas o filas de tabla que el chunker no parte.
- Los PDF llegan a Markdown como líneas sueltas (`Tarjeta:` / `Clásica, Membresía anual:`...). Es un reto propio de fees: convendría convertir las tablas de los PDF a filas completas antes del chunking (cambio en `loaders.py`, a evaluar en S4).

### Decisión provisional
Mantener `INGEST_CHUNK_MAX_CHARS=1500`. Para fees (cifras exactas y excepciones cerca de la regla) pesa más conservar el contexto que ganar precisión. En la sesión 4 se mide el efecto con preguntas reales; este número es una hipótesis.

> Método: conteos obtenidos con el mismo código de ingesta (`load_corpus` y `chunk_document`) sin llamar a Foundry, para no tocar `data/`. Al correr `uv run ingest build` con cada valor deben salir las mismas cantidades.

## 7. Sesión 4 · RAG con búsqueda híbrida (borrador)

Dominio `fees`, 29 chunks (`INGEST_CHUNK_MAX_CHARS=1500`), `gpt-5-mini`, embeddings `text-embedding-3-small`, base local (Qdrant) con `SEARCH_TOP_K=5`.
Una sola corrida por pregunta; los tiempos de `gpt-5-mini` varían entre corridas. "Primer texto" y "completa" son los que imprime `dev/chat.py`.

### 7.1 Las cinco preguntas (paso 11)

| Tipo | Pregunta | Fuentes que citó | ¿Fuente correcta? | Primer texto (ms) | Completa (ms) | Tokens ent. / sal. |
|---|---|---|---|---:|---:|---:|
| regla | ¿Cuánto cobran por retirar en un cajero de otro banco? | `fees-faq.md`, `atm-network.html` | Sí | 7306 | 7460 | 826 / 353 |
| seguimiento (1/2) | Me cobraron dos veces una comisión, ¿se puede devolver? | `fees-faq.md`, `fee-refund-policy.docx` ×2 | Sí | 7219 | 7367 | 855 / 682 |
| seguimiento (2/2) | ¿y en cuánto tiempo me la devuelven? | `fee-refund-policy.docx` › Plazos | Sí: entendió "la" gracias al rewrite (15 días hábiles, abono en 2) | 9413 | 9540 | 1225 / 948 |
| término exacto | ¿Qué es el ITF? | `itf-tax.md` ×3, `fees-faq.md`, `fees-glossary.md` | Sí | 6091 | 6348 | 765 / 585 |
| documento vencido | ¿Cuánto era la comisión por transferencias interbancarias en 2024? | ninguna | Sí: no citó `transfer-fees-2024.csv`; dijo que no tiene el dato | 5686 | 5824 | 859 / 516 |
| fuera del corpus | ¿A cuánto está el dólar hoy? | ninguna | Sí: `out_of_scope`, sin fuentes | 4482 | 4626 | 942 / 343 |

Los tokens del turno de seguimiento incluyen la llamada de reescritura (`SearchQuery`).

Observaciones:
- Con el mock el streaming se nota (primer texto 308 ms, completa 1426 ms). **Con `gpt-5-mini` casi no**: el primer texto llega 100 a 250 ms antes del final, porque el modelo razona varios segundos antes de emitir. La espera que siente el usuario (4 a 9 s) es el razonamiento, no el streaming.
- Con el mock, la pregunta del cajero sale `out_of_scope`: sus reglas de palabras son de fraude. No es un fallo del agente.

### 7.2 `SEARCH_TOP_K` (paso 12)

| K | Fragmentos enviados | Cajero: tokens ent. / sal., completa | ITF: tokens ent. / sal., completa |
|---:|---:|---|---|
| 2 | 2 | 640 / 265, 6.7 s | 600 / 330, 4.0 s |
| 5 (actual) | 5 | 826 / 353, 7.5 s | 765 / 585, 6.3 s |
| 10 | 10 | 1306 / 592, 8.9 s | 1340 / 837, 8.5 s |

- **Con 2:** las respuestas salieron correctas y con fuente en estas dos preguntas. Riesgo: una respuesta que necesita varios documentos queda corta (ITF está repartido en 3 chunks de `itf-tax.md` más glosario y FAQ).
- **Con 10:** cuesta 2.0 a 2.2 veces los tokens de entrada que K=2 y tarda más. Entran fuentes que no vienen al caso (`savings-accounts-fees`, `credit-cards-fees`, `transfer-fees`, `fee-refund-policy` en la pregunta del ITF) y, en el cajero, la fuente principal pasó a ser `savings-accounts-fees.html` en vez del FAQ.
- **Valor propuesto: 5** (el actual), a confirmar por el equipo. Es un punto medio entre costo y cobertura; en la sesión 5 se mide con el golden set y el juez, no a ojo.

### 7.3 Problema encontrado en el chunker con `fees-faq.md`
El FAQ usa `###` directamente bajo `#` (sin `##`). El chunker arma la ruta con `path[: level - 1]`, y con nivel 3 conserva la pregunta anterior como si fuera un título padre. Resultado, las rutas de sección quedan así:

```
Preguntas frecuentes sobre comisiones > ¿Cuánto cobran por retirar en un cajero de otro banco?
… > ¿Cuánto cobran por retirar en un cajero de otro banco? > ¿El ITF es una comisión del banco?
… > ¿Cuánto cobran por retirar en un cajero de otro banco? > ¿Puedo pedir que me devuelvan una comisión?
… > ¿Cuánto cobran por retirar en un cajero de otro banco? > ¿Cuánto cuesta el estado de cuenta en papel?
```

La pregunta del cajero queda escrita al inicio del `content` de 3 chunks que no hablan de eso, y contamina los embeddings y la búsqueda por palabras. Con K=5 la búsqueda del cajero devolvió 4 chunks de `fees-faq` (los 4 del FAQ), cuando solo uno responde. Este es el tipo de adaptación del chunker a un dominio que pide el gate T2. Propuesta: llevar la ruta por nivel real de título (descartar los títulos de nivel igual o mayor al nuevo), reconstruir con `uv run ingest build` y volver a medir.

### 7.4 Pendiente
- Paso 13: publicar en Azure AI Search y comparar local frente a AI Search (necesita `SEARCH_ENDPOINT` y `SEARCH_INDEX` de la ficha del equipo).
- Decidir el `SEARCH_TOP_K` del equipo.

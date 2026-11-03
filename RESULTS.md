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

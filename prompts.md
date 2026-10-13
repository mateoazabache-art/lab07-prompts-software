# Biblioteca de prompts — Laboratorio 07

## Tarea 1 — Clasificar el sentimiento de comentarios de clientes
**Herramienta:** Claude

### Versión A — Sin ejemplos (zero-shot)
**Prompt:**
> Clasifica el sentimiento del siguiente comentario de un cliente como Positivo, Negativo o Neutro:
> "El pedido llegó tarde, pero el producto está bien."

**Resultado (resumen):** La IA dio la clasificación (Neutro o Mixto), pero con una explicación larga. A veces añadió una categoría que no pedí ("Mixto") y el formato de la respuesta no fue siempre el mismo.

### Versión B — Few-shot (3 ejemplos)
**Prompt:**
> Clasifica el sentimiento de cada comentario como Positivo, Negativo o Neutro. Responde solo con una palabra.
>
> Comentario: "Excelente atención, volvería a comprar."
> Sentimiento: Positivo
>
> Comentario: "El


### Mejora observada

| Pregunta de control | Versión A | Versión B |
|---|---|---|
| ¿Usó solo los tipos válidos? | No (variaciones y combinaciones) | **Sí**, los 5 tipos son válidos: BUG, FEATURE, CONSULTA, RENDIMIENTO, SEGURIDAD |
| ¿La prioridad sigue el criterio indicado? | No (usó "Crítica") | **Sí**: ALTA en el issue 1 (afecta dinero) y en el 5 (seguridad); BAJA en 2 y 3; MEDIA en 4, que no afecta dinero, datos ni seguridad |
| ¿El formato es el mismo en los 5 issues? | No (tabla con texto libre) | **Sí**: cada issue sigue `Issue: ... / Tipo: ... \| Prioridad: ... \| Módulo: ...` |

**Conclusión:** el few-shot logró que la salida respetara el formato, las categorías cerradas y el criterio de prioridad. También eliminó el texto adicional, y el resultado queda listo para copiarse o procesarse. Los ejemplos funcionaron como plantilla: el modelo imitó su estructura y usó solo los valores permitidos.

# Biblioteca de prompts

## Tarea: Rondas de despliegue de microservicios

**Problema:** 7 microservicios se despliegan por rondas. Un servicio solo se despliega cuando todas sus dependencias ya están desplegadas. Los servicios sin dependencias pendientes se despliegan en paralelo en la misma ronda.

| Servicio | Depende de |
|---|---|
| auth | — |
| catalogo | — |
| usuarios | auth |
| pagos | usuarios |
| pedidos | usuarios, pagos, catalogo |
| notificaciones | pedidos |
| gateway | auth, notificaciones |

---

## Versión A — Zero-shot

**Prompt:** el problema + "¿Cuántas rondas se necesitan como mínimo? Responde solo con el número."

**Resultado:** `5`

## Versión B — Few-shot (3 ejemplos)

**Prompt:** el mismo problema, pero con 3 ejemplos resueltos paso a paso (A→B = 2 rondas; A,B→C = 2 rondas; A→B,C→D = 3 rondas) y pidiendo razonar ronda por ronda.

**Resultado:** `6`

---

## Verificación ronda por ronda

| Ronda | Se despliega | Por qué puede desplegarse |
|---|---|---|
| 1 | auth, catalogo | No tienen dependencias (van en paralelo) |
| 2 | usuarios | auth ya está desplegado |
| 3 | pagos | usuarios ya está desplegado |
| 4 | pedidos | usuarios, pagos y catalogo ya están desplegados |
| 5 | notificaciones | pedidos ya está desplegado |
| 6 | gateway | auth y notificaciones ya están desplegados |

**Cadena más larga:** auth → usuarios → pagos → pedidos → notificaciones → gateway = 6 servicios en secuencia.

**Respuesta correcta: 6 rondas.**

---

## Comparación de las dos versiones

### ¿Coinciden ambas versiones?

**No.**

| Versión | Respuesta | ¿Correcta? |
|---|---|---|
| A (zero-shot) | 5 | No |
| B (few-shot) | 6 | Sí |

La versión A se equivocó al contar la cadena (se saltó un eslabón). La versión B, al ir ronda por ronda, llegó al valor correcto.

### ¿En cuál es más fácil detectar un error?

**En la versión B (few-shot).**

- En A solo aparece un número (`5`). Para saber si está mal hay que resolver el problema completo por cuenta propia.
- En B se ve el razonamiento de cada ronda, así que se puede revisar paso a paso y señalar en qué ronda falla (por ejemplo, si pedidos se pusiera antes que pagos).
- Los ejemplos también fijan el formato de la respuesta, lo que hace más fácil comparar.

**Mejora observada:** pasar de zero-shot a few-shot corrigió la respuesta (5 → 6) y volvió verificable el resultado.

---

## Prueba de ciclo

**Dependencia agregada:** `catalogo depende de gateway`

| Servicio | Depende de |
|---|---|
| auth | — |
| catalogo | **gateway** |
| usuarios | auth |
| pagos | usuarios |
| pedidos | usuarios, pagos, catalogo |
| notificaciones | pedidos |
| gateway | auth, notificaciones |

### Razonamiento ronda por ronda

| Ronda | Se despliega | Observación |
|---|---|---|
| 1 | auth | Es el único sin dependencias (catalogo ya no lo es) |
| 2 | usuarios | auth ya está desplegado |
| 3 | pagos | usuarios ya está desplegado |
| 4 | *(ninguno)* | Se bloquea: no hay ningún servicio que pueda desplegarse |

**Servicios que quedan sin desplegar:** catalogo, pedidos, notificaciones, gateway.

Cada uno espera a otro de este grupo:

```
catalogo → gateway → notificaciones → pedidos → catalogo
```

(catalogo espera a gateway, gateway espera a notificaciones, notificaciones espera a pedidos y pedidos espera a catalogo.)

### Resultado

**Se forma un ciclo de dependencias, así que el despliegue es imposible.** No existe un número mínimo de rondas.

### ¿Detecta el modelo el ciclo?

- **Con el razonamiento ronda por ronda (versión B): sí.** Al llegar a la ronda 4 se ve que no se puede desplegar nada y que quedan servicios pendientes.
- **Con zero-shot y "responde solo con el número" (versión A): hay riesgo de que no.** Esa instrucción obliga a dar un número, y no deja espacio para decir "imposible". *(Por probar en la herramienta usada.)*

### Mejora sugerida al prompt

Agregar a la versión B una regla y un 4.º ejemplo con ciclo:

> Si en alguna ronda no se puede desplegar ningún servicio y todavía quedan pendientes, responde "No es posible: hay un ciclo de dependencias" e indica los servicios involucrados.

**Ejemplo con ciclo:**

```
Servicios: A, B. A depende de B y B depende de A.
Razonamiento: Ronda 1: ninguno (A espera a B y B espera a A).
Respuesta: No es posible, hay un ciclo (A ↔ B).
```

| Pregunta | Salida A (sin ejemplos) | Salida B (con ejemplos) |
|---|---|---|
| ¿El JSON es válido? | Sí. Es un arreglo con 2 objetos, con comillas dobles y comas correctas. | Sí. Es un arreglo con 2 objetos, también bien formado. |
| ¿Se mantienen los nombres de los campos? | Sí entre los dos objetos de la misma salida (ambos usan `id`, `plataforma`, `version`, `funcionalidad_afectada`, etc.). Pero esos nombres los inventé yo, porque no había esquema que seguir. | Sí. Usa exactamente los campos de los ejemplos: `plataforma`, `version_app`, `modulo`, `sintoma`, `reportado_por`, `fecha`. |
| ¿Usó `null` para los datos faltantes? | En su mayoría sí (`version`, `cupon`, `area_reportante`, `fecha_reporte`). Pero en `mensaje_de_error` del reporte 2 puso el texto "Ninguno (no se muestra ningún error)". Ese dato sí aparece en el reporte, así que no es `null`. | Sí. `fecha: null` en el reporte 1 y `version_app: null` en el reporte 2. |
| ¿Cómo trató "desde ayer"? | Lo conservó como texto en `inicio_del_problema`: "Desde ayer (fecha relativa, sin fecha exacta)". No lo convirtió a fecha, pero mezcló el dato con una explicación dentro del valor. | Lo descartó. Como el esquema solo tiene `fecha` y no un campo para el inicio del problema, puso `fecha: null` y se perdió la información. |

## Revisión de seguridad del login: prompt sin rol vs prompt con rol

**Código revisado:** función `login(usuario, clave)` en Python con SQLite, que arma la consulta SQL concatenando texto.

### Versión 1: sin rol

**Prompt:** `Revisa este código:` + el código.

**Qué detectó:**

| # | Hallazgo |
|---|---|
| 1 | Inyección SQL (con el ejemplo `' OR '1'='1' --`) |
| 2 | Contraseñas en texto plano (recomienda hash con bcrypt) |
| 3 | `SELECT *` innecesario |
| 4 | Sin límite de intentos (fuerza bruta) |

**Cómo respondió:**
- Nivel de detalle: explicó paso a paso cómo funciona el ataque y cómo queda la consulta después de la inyección.
- Lenguaje: cercano y didáctico, le habla al usuario de "tú" y usa frases como "problema grave de seguridad".
- Formato: libre (títulos, ejemplos y una tabla resumen antes/después).
- Terminó ofreciendo ajustar el código según la base de datos.

### Versión 2: con rol

**Prompt:** rol (revisor de seguridad con experiencia en OWASP Top 10) + contexto (API en Python con SQLite, equipo junior) + código + tarea + formato (tabla Vulnerabilidad | Riesgo | Corrección y luego el código corregido).

**Qué detectó:**

| # | Hallazgo | Categoría OWASP | Nivel de riesgo |
|---|---|---|---|
| 1 | Inyección SQL | A03:2021 Injection | Crítico |
| 2 | Contraseñas en texto plano | A02:2021 Cryptographic Failures | Alto |
| 3 | Sin protección contra fuerza bruta | A07:2021 Identification and Authentication Failures | Alto |
| 4 | Sin validación de entrada | (sin categoría) | Medio |
| 5 | Sin registro de eventos (logging) | A09:2021 Security Logging and Monitoring Failures | Medio |
| 6 | `SELECT *` | (sin categoría) | Bajo |

**Cómo respondió:**
- Nivel de detalle: más completo y ordenado. Cada hallazgo tiene riesgo y corrección, y el código corregido incluye validación de entrada, consulta parametrizada, verificación del hash con bcrypt y el código para registrar usuarios.
- Lenguaje: más técnico y profesional. Usa categorías OWASP, niveles de severidad (Crítico, Alto, Medio, Bajo) y términos como "hash con sal".
- Formato: siguió exactamente la tabla pedida, más una sección de notas para el equipo junior (migrar la columna `clave` a `clave_hash`, por qué el límite de 72 bytes, por qué el rate limiting va en la capa de la API).

### Comparación

| Aspecto | Sin rol | Con rol |
|---|---|---|
| Hallazgos | 4 | 6 (agregó validación de entrada y logging) |
| Inyección SQL y contraseñas sin hash | Detectadas | Detectadas, con categoría OWASP y severidad |
| Lenguaje | Didáctico, explicado con un ejemplo de ataque | Técnico, con vocabulario de seguridad |
| Prioridad de los riesgos | No indica niveles | Indica nivel de riesgo para cada uno |
| Código corregido | Dos fragmentos cortos | Función completa con validaciones y ejemplo de registro |
| Formato | Libre | Tabla pedida + código + notas |

### Conclusión

Asignar el rol no cambió lo más grave, porque en ambos casos se detectó la inyección SQL y las contraseñas sin hash. Lo que cambió fue la cantidad de hallazgos (se agregaron validación de entrada y logging), el nivel de detalle (riesgo y corrección por cada uno) y el lenguaje (de explicación sencilla a vocabulario de auditoría con OWASP).

**Limitación:** el segundo prompt tenía rol, contexto, tarea y formato a la vez, así que no se puede asegurar que todo el cambio se deba solo al rol. Para comprobarlo, habría que probar un prompt que solo agregue el rol y mantenga el resto igual.

## Rúbrica de evaluación


| Criterio     | Descripción                                    | Puntaje (1-5) |
| ------------ | ---------------------------------------------- | :-----------: |
| Exactitud    | La respuesta es correcta (verificada por ti)   |         5     |
| Formato      | Respeta el formato o esquema solicitado        |         5     |
| Consistencia | Aplica el mismo criterio en todos los casos    |         5     |
| Utilidad     | Se puede usar directamente en el proyecto      |         5     |

## Comparación: sin ejemplos vs. técnica avanzada


| Tarea                       |  Versión A  | Versión B | 
| --------------------------- |  :-------:  | :-------: |
| 1. Clasificación de issues  |   13/20     |   18/20   |     
| 2. Razonamiento lógico      |   14/20     |   19/20   |                  
| 3. Extracción estructurada  |   13/20     |   18/20   |                  
| 4. Revisión de código (rol) |   15/20     |   20/20   |                  

## Plantilla: Clasificación few-shot

```text
Actúa como {{rol}}.
Clasifica cada {{elemento}} siguiendo el formato de los ejemplos.
Categorías válidas: {{categorias}}.

Ejemplo 1: {{entrada_1}} → {{salida_1}}
Ejemplo 2: {{entrada_2}} → {{salida_2}}
Ejemplo 3: {{entrada_3}} → {{salida_3}}

Elementos a clasificar: {{datos}}
```

## Ejemplo rellenado: clasificar vulnerabilidades por nivel de riesgo

```text
Actúa como revisor de seguridad de aplicaciones web con experiencia en OWASP Top 10.
Clasifica cada vulnerabilidad siguiendo el formato de los ejemplos.
Categorías válidas: Crítico, Alto, Medio, Bajo.

Ejemplo 1: Consulta SQL armada concatenando texto → Crítico
Ejemplo 2: Contraseñas guardadas en texto plano → Alto
Ejemplo 3: Uso de SELECT * en una consulta → Bajo

Elementos a clasificar:
1. La API no usa HTTPS
2. El mensaje de error muestra el detalle completo de la base de datos
3. No hay límite de intentos de login
```

## Qué reemplazó cada campo

| Campo | Valor usado |
|---|---|
| `{{rol}}` | revisor de seguridad con experiencia en OWASP |
| `{{elemento}}` | vulnerabilidad |
| `{{categorias}}` | Crítico, Alto, Medio, Bajo |
| `{{entrada_N}}` → `{{salida_N}}` | hallazgo → nivel de riesgo |
| `{{datos}}` | lista de vulnerabilidades nuevas |

## Recomendaciones

- Los 3 ejemplos deben cubrir categorías distintas. Si todos fueran "Crítico", el modelo tendería a responder siempre eso.
- Las categorías deben ser exactamente las que aparecen en los ejemplos, para que la salida sea uniforme.
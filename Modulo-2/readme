# Seguimiento de pedidos con IA (Make + Telegram + Claude + Google Sheets)

Automatización que recibe consultas de seguimiento de pedidos por Telegram, clasifica automáticamente la gravedad del caso con IA, y decide si responde sola o si el caso necesita la atención de un humano.

🎥 **Video demo:** [ver en YouTube](https://youtu.be/42rt0vKsvWA)

---

## 📑 Índice

- [El problema que resuelve](#-el-problema-que-resuelve)
- [Diagrama de flujo](#️-diagrama-de-flujo)
- [Por qué el flujo pide el número de pedido en 2 turnos](#️-por-qué-el-flujo-pide-el-número-de-pedido-en-2-turnos)
- [Stack y por qué se eligió cada pieza](#-stack-y-por-qué-se-eligió-cada-pieza)
- [Modelo de datos](#️-modelo-de-datos)
- [Clasificación con IA](#-clasificación-con-ia)
- [Manejo de errores y resiliencia](#️-manejo-de-errores-y-resiliencia-lecciones-reales-del-proceso)
- [Comparativo de costos](#-comparativo-de-costos-claude-haiku-vs-sonnet)
- [Estructura del escenario en Make](#️-estructura-del-escenario-en-make)
- [Habilidades demostradas](#-habilidades-demostradas)

---

## 🧩 El problema que resuelve

Una gran parte de los tickets de soporte de cualquier tienda online son simplemente "¿dónde está mi pedido?" — información que ya existe en el sistema, pero que un agente humano tiene que buscar y comunicar una y otra vez.

**Sin esta automatización:**
- Un agente gasta minutos por consulta repitiendo la misma información.
- Los casos que sí importan (retrasos graves, extravíos, daños) se mezclan con las consultas simples.
- El cliente espera por una respuesta que podría ser instantánea.

**Con esta automatización:** el sistema resuelve solo las consultas simples, y escala únicamente los casos que de verdad requieren criterio humano — con todo el contexto ya armado para decidir rápido.

---

## 🗺️ Diagrama de flujo

![Diagrama del escenario completo en Make](diagrama_escenario_make.png)

```
Cliente escribe
        │
        ▼
┌────────────────────┐
│  Detecta número      │   Busca un patrón "PED-XXX"
│  de pedido en el      │   en el texto del mensaje
│  mensaje (Text        │
│  Parser)               │
└────────────────────┘
        │
        ├── No lo encuentra ──► Responde pidiendo el número de pedido (fin)
        │
        └── Sí lo encuentra
                │
                ▼
        ┌───────────────┐
        │  Busca el       │   Google Sheets, columna
        │  pedido         │   numero_pedido
        └───────────────┘
                │
                ▼
        ┌───────────────┐
        │  Claude         │   Clasifica: intención,
        │  clasifica      │   urgencia, ¿requiere humano?,
        │                 │   respuesta sugerida
        └───────────────┘
                │
                ▼
        ┌───────────────┐
        │  Limpieza de    │   Extrae el JSON puro de
        │  JSON           │   la respuesta de Claude
        └───────────────┘
                │
                ▼
          ┌───────────┐
          │  Router     │  ¿requiere_humano?
          └───────────┘
                │
     ┌──────────┴──────────┐
     ▼                      ▼
  false                   true
     │                      │
     ▼                      ▼
Responde con la         Avisa con el contexto
respuesta sugerida        completo del caso
por Claude                 (⚠️ requiere atención)
```

---

## ⚙️ Por qué el flujo pide el número de pedido en 2 turnos

La primera versión de este proyecto intentaba identificar al cliente automáticamente por su número de teléfono, cruzándolo contra la base de datos de pedidos. Se abandonó ese diseño por una razón técnica real (documentada en la sección de resiliencia más abajo), y se reemplazó por un flujo más simple: el cliente da su número de pedido, y el sistema busca directo por ese dato — sin necesitar comparar ni limpiar números de teléfono.

**Turno 1:** el cliente pregunta sin dar contexto ("¿cómo va mi pedido?") → el sistema responde pidiendo el número de pedido.

**Turno 2:** el cliente escribe su número de pedido → el sistema lo busca, lo clasifica, y responde o escala según corresponda.

Cada mensaje se evalúa de forma independiente (Make no necesita "recordar" la conversación): un Text Parser revisa si el texto entrante contiene un patrón `PED-\d+`, y esa detección es el diferenciador entre ambos turnos.

---

## 🧰 Stack y por qué se eligió cada pieza

| Función | Herramienta | Por qué |
|---|---|---|
| Canal del cliente | Telegram Bot | Gratis, sin límites de mensajes diarios, y rápido de configurar para iterar sin restricciones durante las pruebas |
| Base de datos de pedidos | Google Sheets | Simple de inspeccionar y editar a mano durante pruebas; suficiente para el alcance del proyecto |
| Clasificación de intención | Claude Haiku 4.5 | Tarea de clasificación + redacción corta: no requiere el razonamiento de un modelo más grande, y es notablemente más barato para iterar sin restricción |
| Automatización / orquestación | Make | Conecta cada pieza (Telegram, Google Sheets, Claude) sin necesidad de escribir un backend propio |

---

## 🗂️ Modelo de datos

Hoja de cálculo `pedidos_ecommerce`:

📄 **Ver la hoja completa:** [pedidos_ecommerce en Google Sheets](https://docs.google.com/spreadsheets/d/1gWBHrjeR3AKAGzwasX9zAQ1kyL1MiylGpXSluCsuZo8/edit?usp=sharing)

![Tabla de pedidos en Google Sheets](modelo_datos_google_sheets.png)

| Campo | Tipo | Ejemplo |
|---|---|---|
| numero_pedido | Texto | PED-001 |
| telefono_cliente | Texto | +5215585532763 |
| estado | Texto | retrasado / a tiempo |
| fecha_estimada | Texto | 2 de septiembre |
| dias_retraso | Número | 10 |

⚠️ **Limitación consciente:** el sistema busca por `numero_pedido` exacto (patrón `PED-\d+` en el texto). No maneja variantes de escritura más allá de eso (ej. "cero cero uno" en vez de "001"), ni el caso de un cliente con múltiples pedidos activos preguntando de forma ambigua — son extensiones razonables para una v2, no defectos del alcance actual.

---

## 🧠 Clasificación con IA

Claude recibe el mensaje del cliente junto con el estado real del pedido, y responde en JSON con 4 campos: `intencion`, `urgencia`, `requiere_humano`, y `respuesta_sugerida`. La regla de escalamiento: se marca `requiere_humano: true` si el cliente expresa frustración fuerte, si el retraso supera 5 días, o si menciona daño o extravío — en cualquier otro caso, Claude redacta directamente la respuesta.

Esto es lo que hace que la IA sea necesaria (no un bot de reglas fijas): interpretar lenguaje libre y variable ("llevo una semana esperando y nadie me dice nada" vs. "¿ya casi llega?") es justo el tipo de ambigüedad que un árbol de decisiones no maneja bien.

**Ejemplo real de una conversación completa:**

![Prueba del flujo completo en Telegram](prueba_flujo_telegram.png)

---

## 🛡️ Manejo de errores y resiliencia (lecciones reales del proceso)

Esta sección documenta problemas reales encontrados durante la construcción, no hipotéticos.

**1. Funciones de texto que no se evaluaban dentro de campos de mapeo.**
Al intentar limpiar el prefijo de un número de teléfono con la función `replace()`, escrita a mano dentro de un campo de Make, la función nunca se ejecutaba — se guardaba como texto literal en vez de evaluarse, sin importar el separador de argumentos usado ni el módulo de destino. En vez de seguir depurando ese comportamiento puntual, se rediseñó el flujo para no depender de esa transformación de texto en absoluto: comparar el número de pedido (texto simple) en vez del teléfono. Lección: cuando una pieza técnica específica se resiste después de varios intentos razonables de diagnóstico, a veces la solución más eficiente es rediseñar para evitar esa dependencia, no insistir en resolverla a toda costa.

**2. JSON de Claude envuelto en marcadores de código.**
El modelo ocasionalmente devolvía el JSON envuelto en ` ```json ... ``` ` a pesar de instrucciones explícitas en el prompt pidiendo JSON puro. Solución: un módulo de Text Parser (`(\{[\s\S]*\})`, con paréntesis de captura) entre Claude y el parser de JSON, que extrae solo el bloque `{...}` sin importar qué texto lo rodee.

**3. Comparación de booleanos con el operador equivocado.**
El Router que decide entre respuesta automática y escalamiento comparaba el campo `requiere_humano` (booleano) usando un operador de texto contra el valor `"False"` con mayúscula — nunca coincidía con el valor real (`false`, minúsculas, tipo lógico) que regresa el parser de JSON. Corregido usando el operador específico para booleanos ("Boolean operators: Equal to") contra `true`/`false` en minúsculas.

**4. Diagnóstico basado en evidencia, no en suposiciones.**
Varias veces el problema parecía estar en un lugar (el filtro, la variable, la fuente de datos) pero al revisar el input/output real de cada módulo en el historial de ejecución, se confirmó que la causa era otra. Revisar el dato crudo en vez de confiar solo en la interfaz visual fue clave para encontrar la causa correcta en cada caso.

---

## 💰 Comparativo de costos (Claude Haiku vs. Sonnet)

Para esta tarea (clasificar un mensaje corto + redactar una respuesta breve), se eligió **Claude Haiku 4.5** por ser significativamente más económico y rápido que un modelo de la familia Sonnet, sin sacrificar calidad perceptible en una tarea de esta complejidad.

⚠️ Los precios exactos por token de cada modelo cambian con cierta frecuencia — antes de publicar una cifra específica de costo por conversación en este README, verifica el precio vigente en [docs.claude.com/en/docs/about-claude/models](https://docs.claude.com/en/docs/about-claude/models) y complétalo aquí con el cálculo real (tokens de entrada + salida por ejecución × número de mensajes esperados al mes).

---

## 🏗️ Estructura del escenario en Make

Telegram Bot (Watch Updates) → Text Parser (detecta número de pedido) → Router → [sin pedido: responde pidiendo el número] / [con pedido: Google Sheets → Claude → Text Parser (limpieza JSON) → Parse JSON → Router → Telegram (respuesta automática, caso normal) / Telegram (aviso de escalamiento, caso grave)]

📦 **Blueprint exportado:** [escenario_make.blueprint.json](escenario_make.blueprint.json) — puedes importarlo directo a tu cuenta de Make (menú de escenarios → Create a new scenario → Import Blueprint) para ver la configuración exacta de cada módulo.

---

## ✅ Habilidades demostradas

- Integración de IA generativa (Claude) para clasificación de intención y redacción contextual
- Prompt engineering para forzar salida estructurada (JSON) de forma confiable
- Diagnóstico técnico basado en evidencia (revisión de raw input/output en vez de solo la interfaz visual)
- Rediseño de arquitectura ante una limitación técnica, en vez de insistir en un enfoque que no escalaba
- Uso de expresiones regulares (Text Parser) para extraer datos estructurados de texto libre
- Modelado de datos simple y documentación honesta de limitaciones de alcance

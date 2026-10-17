# Módulo 2: Orquestación multi-agente con sub-workflows (n8n)

**Alumna:** Pamela González
**Herramienta:** n8n self-hosted (v2.42.7) + Anthropic Chat Model (Claude Sonnet) + Gmail

Convertí mi Agente de Triaje de Soporte del Módulo 1 en una arquitectura **Manager-Worker**. El Manager recibe el mensaje del cliente, lo clasifica con IA y delega el trabajo a un Worker especializado. Cada Worker devuelve un JSON estándar y el Manager registra todo en un correo de trazabilidad.

## Archivos

| Archivo | Qué es |
|---|---|
| `manager_modulo2_gonzalez_pamela.json` | Flujo principal: recibe, clasifica, enruta, maneja fallos y reporta |
| `worker1_soporte_tecnico_gonzalez_pamela.json` | Worker que analiza un problema técnico y extrae datos clave |
| `worker2_facturacion_gonzalez_pamela.json` | Worker que redacta un correo de respuesta sobre cobros o facturas |
| `preentrega_modulo2_gonzalez_pamela.pdf` | Documento de entrega con explicación y capturas |

## Cómo funciona

```
Chat → Router IA (SOPORTE_TECNICO / FACTURACION / OTRO) → Switch
   ├─ SOPORTE_TECNICO → Datos Soporte (Set) → Llamar Worker Soporte ─┐
   ├─ FACTURACION     → Datos Facturación (Set) → Llamar Worker Facturación ─┤
   └─ Fallback / OTRO → Escalar a Supervisor ─────────────────────────┤
        (si un Worker falla o pasa el timeout → Error Worker) ─────────┤
                                                                 Reporte Gmail
```

- **Entrada de cada Worker:** `{ "mensaje": "...", "categoria": "..." }`
- **Salida estándar:** `{ "status": "success | error | escalado", "worker": "...", "data": { ... } }`
- **Fallos:** On Error → Continue (using error output) en los Execute Workflow y Timeout Workflow de 1 minuto en cada Worker.

## Cómo importarlo

1. En n8n: **Create workflow → ⋯ → Import from File** y elige cada `.json`.
2. Conecta tus credenciales de **Anthropic** y **Gmail** en los nodos que lo pidan.
3. En el Manager, abre cada nodo **Llamar Worker** y selecciona el Worker importado correspondiente.
4. Abre el chat del Manager y prueba con: *"mi internet no funciona desde ayer"*, *"me cobraron doble este mes"* o *"quiero saber el horario de su tienda"*.

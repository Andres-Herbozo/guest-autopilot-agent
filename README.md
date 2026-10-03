# Guest Autopilot — Concierge Virtual (Checkpoint 1)

Agente base construido en n8n para **Guest Autopilot**, un concierge automatizado para departamentos de arriendo de corto plazo. En este checkpoint el agente atiende a los huéspedes de una propiedad demo, **"Vista Reñaca"**: responde sus dudas usando la ficha oficial de la propiedad y registra en Google Sheets las solicitudes que requieren acción del anfitrión.

**Entregable:** `checkpoint1_andres_herbozo.json`

## Arquitectura del flujo

```
Chat del huésped (Chat Trigger)
        │
        ▼
Concierge Virtual (AI Agent — Tools Agent)
   ├── Modelo: Claude Sonnet 4.5 (Anthropic Chat Model)
   └── Herramienta: Registrar solicitud al anfitrión (Google Sheets)
        │
        ▼
Log de observabilidad (Slack → #guest-autopilot-log)
        │
        ▼
Respuesta al chat (Set)
```

## Cumplimiento de los requisitos

| Requisito | Implementación |
|---|---|
| Disparador | Nodo `Chat Trigger` que recibe el mensaje libre del huésped. |
| Tools Agent | Nodo AI Agent v3.1. En esta versión de n8n el nodo funciona siempre como Tools Agent y ya no muestra el selector de tipo de agente. |
| Modelo de lenguaje | `Anthropic Chat Model` con **Claude Sonnet 4.5**, temperatura 0.2. Claude 3.5 Sonnet, sugerido en el enunciado, fue retirado de la API de Anthropic, por lo que se usa su sucesor directo. |
| Guardrail de iteraciones | `Max Iterations = 6`, dentro del rango exigido de 5 a 10. |
| System Prompt modular | Rol → Ámbito → Ficha de la propiedad → Objetivo → Reglas → Escalamiento → Restricciones (lo que el agente NO puede hacer). |
| Herramienta lateral | Google Sheets (append), conectada al puerto *Tool* del agente y no al flujo principal. |
| Descripción de la herramienta | Descripción extensa con los casos en que debe **activarse** y en los que **no** debe activarse. |
| Observabilidad | Nodo final de Slack que publica el mensaje recibido, las herramientas invocadas con sus parámetros (`returnIntermediateSteps`), la respuesta del agente y un link a la ejecución. |

## Decisiones de diseño

- **Única fuente de verdad:** la ficha de la propiedad está en el System Prompt. Si la respuesta no está en la ficha, el agente no la inventa: la registra como consulta para el anfitrión.
- **Qué completa el modelo y qué queda fijo:** el agente solo completa `Huesped`, `Tipo`, `Detalle` y `Urgencia` mediante `$fromAI()`. `Fecha`, `Propiedad` y `Estado = Pendiente` son valores fijos, para que el modelo no pueda inventarlos.
- **El agente no confirma nada:** las solicitudes (check-in anticipado, insumos, reparaciones) solo se registran; la confirmación siempre es del anfitrión.
- **Emergencias:** el agente deriva a Bomberos (132), SAMU (131) o Carabineros (133) y registra la incidencia con urgencia Alta.
- **Minimización de datos:** el agente nunca pide RUT, documentos ni datos de pago, y solo registra el nombre de pila del huésped.
- **Resiliencia:** si falla el nodo de Slack, el huésped igual recibe su respuesta (`On Error: Continue`).

## Pruebas de validación

**Caso 1 — el agente decide usar la herramienta**

> Hola, soy Martín, llegamos mañana con mi señora. ¿Podríamos hacer check-in a las 12? Y otra cosa, ¿tienen estacionamiento?

Resultado: responde lo del estacionamiento directamente desde la ficha y registra solo el check-in anticipado en el Sheet. El log de Slack muestra *Herramientas invocadas (1)*.

**Caso 2 — error detectado en las pruebas**

> Hola, soy Pedro, llegamos mañana con mi hijo. ¿Podríamos hacer check-in a las 19? Y otra cosa, ¿dónde queda el estacionamiento?

La primera versión del agente registró esa llegada en el Sheet como "Check-in anticipado". Era un error: el check-in es autónomo desde las 15:00, así que llegar a las 19:00 está dentro del horario y no requiere ninguna acción del anfitrión.

**Corrección aplicada:**
- En el System Prompt se agregó una regla explícita: solo es check-in anticipado si el huésped quiere entrar antes de las 15:00, y solo es check-out tardío si quiere salir después de las 11:00.
- En la descripción de la herramienta se agregó la exclusión correspondiente en la lista *NO ACTIVAR*.

**Caso 3 — el agente decide NO usar la herramienta (después de la corrección)**

> Hola, soy María, llegamos mañana con mi hijo. ¿Podríamos hacer check-in a las 23:00? Y otra cosa, ¿hay estacionamiento? ¿dónde queda?

Resultado: el agente responde que puede llegar a esa hora porque el check-in es autónomo, entrega la ubicación del estacionamiento desde la ficha y no registra nada. El log de Slack muestra *Herramientas invocadas (0)*.

Las capturas de las ejecuciones, de la planilla y de los logs de Slack están en [`evidencia/`](evidencia/).

## Cómo importarlo

1. En n8n: **Workflows → Import from File** y elegir `checkpoint1_andres_herbozo.json`.
2. Crear un Google Sheet llamado **"Guest Autopilot — Solicitudes"**, con una pestaña **Solicitudes** y estos encabezados en la fila 1:
   `Fecha | Propiedad | Huesped | Tipo | Detalle | Urgencia | Estado`
3. Asignar las credenciales; el JSON se exportó sin IDs de credenciales:
   - **Anthropic API** en el nodo del modelo.
   - **Google Sheets OAuth2** en el nodo de la herramienta, y elegir la planilla en *Document*.
   - **Slack** (bot con scopes `chat:write` y `channels:read`) en el nodo del log.
4. Crear el canal `#guest-autopilot-log` en Slack e invitar al bot con `/invite`.
5. En el nodo de Slack, reemplazar `n8n.flowden.cl` del link "Ver ejecución" por el dominio de la propia instancia.
6. Abrir el chat del workflow y enviar el mensaje del Caso 1.

## Próximos módulos

- **Memoria:** historial por huésped (sesión).
- **Integraciones:** calendario iCal de reservas.
- **RAG:** la ficha sale del prompt y pasa a una base de conocimiento con el manual de cada propiedad.
- **Voz:** transcripción de audios de WhatsApp.

---
Andrés Herbozo

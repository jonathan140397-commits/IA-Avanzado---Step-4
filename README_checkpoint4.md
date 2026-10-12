# Checkpoint 4 · Integraciones avanzadas (n8n)

Workflow `checkpoint4_jona_apellido.json`: el Manager del Módulo 3 (memoria en Airtable, clasificador, Workers) más tres integraciones reales: **Gmail** (casilla de soporte), **HubSpot** (CRM) y **Slack** (canal de operaciones), todas por OAuth2.

## Flujo

```
[Gmail Trigger] -> ① IF auto-reply --Sí--> Stop
                        | No
[Chat Trigger] --------> Entrada normalizada -> Memoria Airtable (M3) -> Clasificador -> Worker 1 / Worker 2 / Escalar
                                                                                          |
        ④ Set: limpiar payload -> ¿email válido? --No-----------------------------+
                        | Sí                                                       |
        ② Look up HubSpot -> ¿existe? --Sí--> (sin escritura) --+                 |
                                  | No                          |                 |
                              Create contacto ------------------+                 |
                                                                 v                 |
                                          ③ Gmail CREATE DRAFT (HITL)              |
                                                                 v                 v
                                          Set payload Slack -> Slack -> Actualizar memoria (M3)
```

Los 4 nodos de la rúbrica: `IF - ¿Es auto-reply?`, `Look up contacto (HubSpot)`, `Crear borrador (Gmail Create Draft)`, `Limpiar payload (Set)`.

## Credenciales y mínimo privilegio

| Servicio | Credencial n8n | Scopes a habilitar | Nota |
|---|---|---|---|
| Gmail | Gmail OAuth2 | `gmail.readonly` (trigger) + `gmail.compose` (borradores) | El workflow nunca usa Send. Si usás tu propio cliente OAuth de Google, limitá la pantalla de consentimiento a esos scopes. |
| HubSpot | HubSpot OAuth2 | `crm.objects.contacts.read` + `crm.objects.contacts.write` | Solo contactos. El Look up pide únicamente email, firstname, lastname. |
| Slack | Slack OAuth2 | `chat:write` | Canal elegido por ID, así no hace falta `channels:read`. Invitá la app al canal (`/invite @app`). |

Cada credencial debe mostrar el semáforo en verde al probarla.

## Pasos para dejarlo andando

1. Importar el JSON (menú ⋯ → Import from file).
2. Reconectar credenciales: Gmail (trigger y borrador), HubSpot, Slack, Airtable y Gemini.
3. Completar los marcadores `REEMPLAZAR_...`: ID de base y tabla de Airtable, ID del canal de Slack (C...), y los IDs de los Workers (los mismos del M3).
4. Ejecutar una vez el Gmail Trigger con **Fetch Test Event** para ver la estructura real del mail.

## Test de regresión (Test step en cada nodo)

| # | Prueba | Resultado esperado |
|---|---|---|
| 1 | Mail con asunto `Out of office` o remitente `no-reply@...` | El IF sale por **true** y llega a Stop. No se ejecuta ningún nodo posterior. |
| 2 | Mail normal de un cliente nuevo | Look up vacío → Create contacto → borrador en Gmail → aviso en Slack. |
| 3 | Mismo mail otra vez (mismo remitente) | Look up encuentra el contacto → "Contacto existente", sin Create. No hay error 409. |
| 4 | Mensaje por el chat de n8n | Se saltea el CRM y el borrador (email vacío), se avisa en Slack y se actualiza la memoria. |
| 5 | Revisar Borradores en Gmail | El borrador queda sin enviar. Un humano lo revisa y lo manda. |

## Notas de diseño

- HubSpot v2 no tiene un "Create" puro: la creación usa la operación *Create or Update* (`upsert`). El Look up previo decide si se llega a ella.
- El Set de limpieza deja solo los campos necesarios (sin binarios ni HTML, cuerpo ≤ 1500 caracteres). Slack recibe un texto de hasta 600 caracteres.
- Para evitar que el agente se responda a sí mismo, el IF también descarta el header `Auto-Submitted`. Si querés, agregá una condición con tu propia dirección de correo.

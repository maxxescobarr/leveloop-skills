---
name: crear-pedido-lienzos
description: >
  Crea un pedido completo para Lienzos de Fe a partir del link de un chat de ManyChat.
  Lee automáticamente los datos del subscriber (nombre, teléfono, dirección, apartado,
  descripción del pedido), crea la orden en Shopify, registra el pedido en Notion con
  tabla para el diseñador, y actualiza los campos personalizados en ManyChat.

  También procesa LIQUIDACIONES: cuando un cliente ya pagó el restante, marca la orden
  como pagada en Shopify, manda el comprobante a Slack y activa el flow de confirmación.

  Usar SIEMPRE que Max u otro miembro del equipo diga "tengo un pedido", "nuevo pedido",
  "crea el pedido", "mete el pedido", "registra el pedido", o comparta un link de ManyChat
  tipo app.manychat.com/fb2438027/chat/... aunque no diga explícitamente "usa el skill".
  También activar si dicen "me llegó un pago", "ya apartaron", "ya liquidó", "liquidación",
  "ya pagó completo" o "procesa la liquidación".
---

# Skill: Crear Pedido — Lienzos de Fe

Automatiza la creación completa de un pedido en Shopify + Notion + ManyChat.

**REGLA DE ORO: Nunca ejecutar ninguna acción sin antes presentar al usuario un resumen
completo de los datos y recibir confirmación explícita. Si algo no cuadra o falta,
reportarlo claramente antes de proceder.**

---

## Credenciales y configuración

| Sistema | Credencial / ID |
|---|---|
| ManyChat API | Bearer `${MANYCHAT_API_TOKEN}` |
| ManyChat Page ID | `fb2438027` |
| Shopify store | `ah29ra-xt.myshopify.com` |
| Shopify Client ID | `${SHOPIFY_CLIENT_ID}` |
| Shopify Client Secret | `${SHOPIFY_CLIENT_SECRET}` |
| Notion DB Pedidos | `collection://7919d7cc-b93f-4ce5-8d44-b4c1ce9fefa9` |
| Slack Bot Token | `${SLACK_BOT_TOKEN}` |
| Slack canal comprobantes | `C084HG6ANN6` (`#confimacion-pagos-lienzos`) |
| Flow Confirmar Liquidación | `content20250708192546_283376` |

### Obtener token Shopify (expira cada 24h)
```bash
curl -s --resolve "ah29ra-xt.myshopify.com:443:23.227.38.74" \
  -X POST "https://ah29ra-xt.myshopify.com/admin/oauth/access_token" \
  -H "Content-Type: application/json" \
  -d '{
    "client_id": "${SHOPIFY_CLIENT_ID}",
    "client_secret": "${SHOPIFY_CLIENT_SECRET}",
    "grant_type": "client_credentials"
  }'
```

---

## Catálogo de productos

| Tamaño | Variant ID | Precio lista |
|---|---|---|
| 30x20 | `49695962235197` | $549 |
| 40x60 | `49660694266173` | $1,299 |
| 60x90 | `49660694298941` | $1,699 |
| 80x120 | `49660694331709` | $2,799 |
| 100x150 | `50976518930749` | $4,499 |

---

## Campos ManyChat — grupo "Pedido"

| Campo | Tipo | Quién lo llena |
|---|---|---|
| `Apartado` | Number | Vendedor (antes de llamar al skill) |
| `Descripción del pedido` | Text | Vendedor (antes de llamar al skill) |
| `# Pedido` | Number | **Este skill** |
| `Restante` | Number | **Este skill** |
| `Total Pedido` | Text | **Este skill** |

> ⚠️ Campos `Number` en ManyChat: enviar SIN comillas en `field_value`.
> `Total Pedido` es tipo `Text`: enviar CON comillas.

---

## MODO A — Crear nuevo pedido

### Paso A1 — Leer datos del subscriber

```bash
curl -s -H "Authorization: Bearer ${MANYCHAT_API_TOKEN}" \
  "https://api.manychat.com/fb/subscriber/getInfo?subscriber_id={SUBSCRIBER_ID}"
```

Extraer:
- `first_name` + `last_name` → nombre completo
- `whatsapp_phone` → teléfono
- `email` → correo electrónico
- `custom_fields` → `Apartado`, `Descripción del pedido`, `Dirección` (si existe)
- `last_input_text` → puede contener dirección si no hay campo dedicado

### Paso A2 — Validar y presentar resumen AL USUARIO antes de actuar

Construir y mostrar esta ficha ANTES de ejecutar cualquier acción:

```
📋 RESUMEN DEL PEDIDO — confirma antes de continuar

👤 Cliente
  • Nombre:   [valor]  ← ⚠️ FALTANTE si no se encontró
  • Teléfono: [valor]  ← ⚠️ FALTANTE si no se encontró
  • Correo:   [valor]  ← ⚠️ FALTANTE (no recibirá notificaciones FedEx)

📦 Pedido
  • Descripción:       [valor del campo]
  • Tamaño detectado:  [tamaño]  ← ⚠️ NO IDENTIFICADO si no es claro
  • Cantidad:          [número]  ← ⚠️ NO CLARA si no se especifica
  • Total acordado:    $[valor]  ← ⚠️ FALTANTE si no se encontró
  • Precio lista:      $[precio_unitario × cantidad]
  • Descuento:         $[diferencia]  ← ⚠️ NEGATIVO si total > lista

💰 Pago
  • Apartado:  $[valor]  ← ⚠️ VACÍO en ManyChat
  • Restante:  $[total - apartado]  ← ⚠️ NO CALCULABLE si falta alguno

🏠 Dirección
  [dirección completa]  ← ⚠️ FALTANTE — se creará sin dirección de envío

¿Procedo con estos datos?
```

#### Alertas que BLOQUEAN — no continuar hasta que el usuario corrija o confirme:
- ⚠️ `Descripción del pedido` vacía — imposible crear el pedido sin saber qué compró
- ⚠️ Tamaño no identificado en la descripción — preguntar explícitamente
- ⚠️ Total acordado no encontrado — preguntar el monto al usuario
- ⚠️ `Apartado` > `Total` — no cuadra, reportar antes de continuar
- ⚠️ `Descuento` negativo — el total acordado es mayor al precio lista, confirmar con usuario
- ⚠️ Nombre del cliente faltante — preguntar al usuario

#### Alertas que NO bloquean — reportar pero continuar:
- ⚠️ Correo faltante — crear pedido, avisar que no recibirá emails de FedEx
- ⚠️ Dirección faltante — crear pedido sin dirección, avisar al final
- ⚠️ Apellido faltante — usar solo el nombre disponible
- ⚠️ `Apartado` = 0 — mencionar en el resumen, preguntar si es correcto antes de proceder

**ESPERAR confirmación explícita del usuario ("sí", "procede", "adelante" o equivalente)
antes de ejecutar cualquier paso siguiente.**

### Paso A2b — Resolver dirección

**Caso 1 — Dirección completa en campo `Dirección` de ManyChat:** usar directamente.

**Caso 2 — Cliente dice "misma dirección", "mismo domicilio", "misma de antes" o similar:**
1. Tomar el número del pedido anterior del custom field `# Pedido` en ManyChat
2. Buscar esa orden en Shopify:
```bash
curl --resolve "ah29ra-xt.myshopify.com:443:23.227.38.74" \
  "https://ah29ra-xt.myshopify.com/admin/api/2025-01/orders.json?name=%23{NUMERO}&status=any" \
  -H "X-Shopify-Access-Token: {TOKEN}"
```
3. Extraer `shipping_address` completo de esa orden y reutilizarlo
4. Mostrar la dirección encontrada en el resumen de validación antes de proceder

**Caso 3 — Sin dirección y sin referencia a pedido anterior:**
- Continuar sin dirección — crear pedido sin `shipping_address`
- En Notas de Notion: `⚠️ Dirección pendiente — solicitar al cliente`
- Avisar al final: `"⚠️ Sin dirección — avísame cuando la tengas"`

---

### Paso A3 — Calcular descuento

```
precio_lista_total = precio_unitario × cantidad
descuento_total    = precio_lista_total - total_acordado
descuento_x_pieza  = descuento_total / cantidad
```

- Si `descuento_total = 0` → no aplicar `applied_discount`
- Si `descuento_total < 0` → ⚠️ bloqueante, reportar al usuario

### Paso A4 — Crear draft order en Shopify

Crear una línea por pieza (no agrupar con quantity > 1):

```bash
curl --resolve "ah29ra-xt.myshopify.com:443:23.227.38.74" \
  -X POST "https://ah29ra-xt.myshopify.com/admin/api/2025-01/draft_orders.json" \
  -H "X-Shopify-Access-Token: {TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "draft_order": {
      "line_items": [
        {
          "variant_id": {VARIANT_ID},
          "quantity": 1,
          "applied_discount": {
            "value_type": "fixed_amount",
            "value": "{DESCUENTO_X_PIEZA}",
            "title": "Promoción"
          }
        }
        // repetir por cada pieza individual
      ],
      "customer": {
        "first_name": "{NOMBRE}",
        "last_name": "{APELLIDO}",
        "phone": "{TELEFONO}",
        "email": "{EMAIL}"
      },
      "shipping_address": {
        "first_name": "{NOMBRE}",
        "last_name": "{APELLIDO}",
        "address1": "{CALLE_NUMERO}",
        "address2": "{COLONIA_REFERENCIAS}",
        "city": "{CIUDAD}",
        "province": "{ESTADO}",
        "country": "MX",
        "zip": "{CP}",
        "phone": "{TELEFONO}"
      },
      "shipping_line": {"title": "Envio gratis", "price": "0.00"},
      "note": "{DESCRIPCION_PEDIDO}",
      "tags": "pedido-manual"
    }
  }'
```

> Omitir `shipping_address` completo si no hay dirección disponible.
> Omitir `applied_discount` si no hay descuento.

### Paso A5 — Aplicar "Pago al momento del envío"

```bash
curl --resolve "ah29ra-xt.myshopify.com:443:23.227.38.74" \
  -X POST "https://ah29ra-xt.myshopify.com/admin/api/2025-01/graphql.json" \
  -H "X-Shopify-Access-Token: {TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation draftOrderUpdate($id: ID!, $input: DraftOrderInput!) { draftOrderUpdate(id: $id, input: $input) { draftOrder { name } userErrors { message } } }",
    "variables": {
      "id": "gid://shopify/DraftOrder/{DRAFT_ID}",
      "input": {"paymentTerms": {"paymentTermsTemplateId": "gid://shopify/PaymentTermsTemplate/9"}}
    }
  }'
```

### Paso A6 — Completar draft → orden real

```bash
curl --resolve "ah29ra-xt.myshopify.com:443:23.227.38.74" \
  -X PUT "https://ah29ra-xt.myshopify.com/admin/api/2025-01/draft_orders/{DRAFT_ID}/complete.json" \
  -H "X-Shopify-Access-Token: {TOKEN}" \
  -d '{"payment_pending": true}'
```

Guardar `order_id` y `name` (#XXXX).

### Paso A7 — Crear pedido en Notion

```json
{
  "No. Pedido": "#XXXX",
  "Cliente": "Nombre completo",
  "Marca": "Lienzos de Fe",
  "Proceso": "Nuevo",
  "Tipo Cliente": "Normal(1)",
  "Tipo de pedido": "[\"Personalizado\"]",
  "Link de many": "https://app.manychat.com/fb2438027/chat/{ID}",
  "Notas": "Descripción del producto + tel + dirección completa (+ advertencias si aplica). NUNCA incluir precios, montos, apartado ni restante en este campo.",
  "date:Fecha de pedido:start": "YYYY-MM-DD",
  "date:Fecha de pedido:is_datetime": 0
}
```

Contenido de la página (tabla para el diseñador):
```markdown
### DETALLES DE IMPRESIÓN & DISEÑO {color="green"}

<tabla: CANTIDAD | TAMAÑO | MEJORAS/ESPECIFICACIONES | COMENTARIO DEL CLIENTE>

> 📌 Diseñador: revisar chat ManyChat — link en propiedades.
> [detalles del pedido]
```

Incluir en Notas cualquier advertencia activa (sin dirección, sin correo, etc.).

### Paso A8 — Actualizar ManyChat

```bash
# # Pedido — Number (sin comillas)
curl -X POST "https://api.manychat.com/fb/subscriber/setCustomFieldByName" \
  -H "Authorization: Bearer ${MANYCHAT_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"subscriber_id": "{ID}", "field_name": "# Pedido", "field_value": XXXX}'

# Restante — Number (sin comillas)
curl -X POST "https://api.manychat.com/fb/subscriber/setCustomFieldByName" \
  -H "Authorization: Bearer ${MANYCHAT_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"subscriber_id": "{ID}", "field_name": "Restante", "field_value": XXXX}'

# Total Pedido — Text (con comillas)
curl -X POST "https://api.manychat.com/fb/subscriber/setCustomFieldByName" \
  -H "Authorization: Bearer ${MANYCHAT_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"subscriber_id": "{ID}", "field_name": "Total Pedido", "field_value": "XXXX"}'
```

### Paso A9 — Notificar en Slack con comprobante

Leer el campo `comprobante_apartado` del subscriber. Si existe, publicar en Slack usando
`chat.postMessage` con Block Kit — Slack jala la imagen directamente desde ManyChat sin
que el agente tenga que descargarla.

```bash
SLACK_TOKEN="${SLACK_BOT_TOKEN}"
CHANNEL="C084HG6ANN6"
IMG_URL="{COMPROBANTE_APARTADO_URL}"

curl -s -X POST "https://slack.com/api/chat.postMessage" \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"channel\": \"$CHANNEL\",
    \"blocks\": [
      {
        \"type\": \"section\",
        \"text\": {
          \"type\": \"mrkdwn\",
          \"text\": \"✅ *Comprobante de pago — Pedido #{NUMERO} · {NOMBRE}*\n💰 *Apartado:* \${APARTADO} MXN\n💳 *Restante:* \${RESTANTE} MXN\n🔗 {LINK_MANYCHAT}\"
        }
      },
      {
        \"type\": \"image\",
        \"image_url\": \"$IMG_URL\",
        \"alt_text\": \"Comprobante #{NUMERO}\"
      }
    ]
  }"
```

> Si no hay `comprobante_apartado`, omitir este paso silenciosamente.

### Paso A10 — Activar flow "Pedido Procesado" en ManyChat

Siempre activar al final, independientemente de si hay comprobante o no.
Confirma al cliente que su pedido fue procesado y lo asigna al diseñador.

```bash
curl -s -X POST "https://api.manychat.com/fb/sending/sendFlow" \
  -H "Authorization: Bearer ${MANYCHAT_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"subscriber_id\": \"{SUBSCRIBER_ID}\",
    \"flow_ns\": \"content20251004163033_953118\"
  }"
```

### Output final — Modo A

```
✅ Pedido creado exitosamente

🛒 Shopify: #XXXX — $X,XXX MXN · Pago al envío
📋 Notion: registrado con tabla para el diseñador
💬 ManyChat: campos actualizados
   • # Pedido: XXXX
   • Total: $X,XXX
   • Restante: $X,XXX
📲 ManyChat: flow "Pedido Procesado" activado — cliente notificado y asignado a diseñador

⚠️ Pendientes (si aplica):
   • Sin dirección — avísame cuando la tengas
   • Sin correo — el cliente no recibirá notificaciones FedEx
```

---

## MODO C — Procesar liquidación de pedido existente

Activar cuando el usuario diga "ya liquidó", "procesa la liquidación", "ya pagó completo",
"llegó la liquidación" o comparta un link de ManyChat en contexto de pago final.

### Paso C1 — Leer datos del subscriber

```bash
curl -s -H "Authorization: Bearer ${MANYCHAT_API_TOKEN}" \
  "https://api.manychat.com/fb/subscriber/getInfo?subscriber_id={SUBSCRIBER_ID}"
```

Extraer:
- `first_name` + `last_name` → nombre
- `custom_fields` → `# Pedido`, `comprobante_liquidacion`, `Total Pedido`, `Restante`

### Paso C2 — Validar y presentar resumen antes de actuar

```
📋 LIQUIDACIÓN — confirma antes de continuar

👤 Cliente: [nombre]
🛒 Pedido Shopify: #[# Pedido]
💰 Total: $[Total Pedido] MXN
✅ Restante a liquidar: $[Restante] MXN

🧾 Comprobante de liquidación: [encontrado ✅ / ⚠️ FALTANTE]

¿Procedo con la liquidación?
```

#### Alertas que BLOQUEAN:
- ⚠️ `# Pedido` vacío — no se puede ubicar la orden en Shopify
- ⚠️ `comprobante_liquidacion` vacío — preguntar si procede sin comprobante

**No ejecutar nada sin confirmación del usuario.**

### Paso C3 — Marcar orden como pagada en Shopify

```bash
TOKEN=$(curl -s --resolve "ah29ra-xt.myshopify.com:443:23.227.38.74" \
  -X POST "https://ah29ra-xt.myshopify.com/admin/oauth/access_token" \
  -H "Content-Type: application/json" \
  -d '{"client_id":"${SHOPIFY_CLIENT_ID}","client_secret":"${SHOPIFY_CLIENT_SECRET}","grant_type":"client_credentials"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('access_token',''))")

# Usar GraphQL orderMarkAsPaid (más confiable que REST transactions)
curl -s --resolve "ah29ra-xt.myshopify.com:443:23.227.38.74" \
  -X POST "https://ah29ra-xt.myshopify.com/admin/api/2025-01/graphql.json" \
  -H "X-Shopify-Access-Token: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation { orderMarkAsPaid(input: {id: \"gid://shopify/Order/{ORDER_ID}\"}) { order { name financial_status: displayFinancialStatus } userErrors { message field } } }"}'
```

Para obtener el ORDER_ID numérico desde el número de pedido:
```bash
curl -s --resolve "ah29ra-xt.myshopify.com:443:23.227.38.74" \
  "https://ah29ra-xt.myshopify.com/admin/api/2025-01/orders.json?name=%23{NUM_PEDIDO}&status=any" \
  -H "X-Shopify-Access-Token: $TOKEN" | python3 -c \
  "import sys,json; orders=json.load(sys.stdin).get('orders',[]); print(orders[0].get('id','') if orders else '')"
```

> Si la orden ya tiene `financial_status: paid`, reportar al usuario y no duplicar.

### Paso C3b — Agregar comprobantes como nota en Shopify

Usar REST PUT para actualizar el campo `note` de la orden con las URLs de ambos
comprobantes (apartado + liquidación). Esto deja evidencia trazable directamente
en la orden de Shopify.

```bash
curl -s --connect-timeout 10 --resolve "ah29ra-xt.myshopify.com:443:23.227.38.74" \
  -X PUT "https://ah29ra-xt.myshopify.com/admin/api/2025-01/orders/{ORDER_ID}.json" \
  -H "X-Shopify-Access-Token: {TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"order\":{\"id\":{ORDER_ID},\"note\":\"{NOTA_ORIGINAL}\n\n💰 Comprobante apartado: {URL_APARTADO}\n✅ Comprobante liquidacion: {URL_LIQUIDACION}\"}}"
```

> Si la nota original ya existe, concatenar — no reemplazar.
> Si no hay `comprobante_apartado`, omitir esa línea.

### Paso C4 — Enviar comprobante de liquidación a Slack

Usar `chat.postMessage` con Block Kit — Slack jala la imagen directamente desde ManyChat.

```bash
SLACK_TOKEN="${SLACK_BOT_TOKEN}"
CHANNEL="C084HG6ANN6"
IMG_URL="{COMPROBANTE_LIQUIDACION_URL}"

curl -s -X POST "https://slack.com/api/chat.postMessage" \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"channel\": \"$CHANNEL\",
    \"blocks\": [
      {
        \"type\": \"section\",
        \"text\": {
          \"type\": \"mrkdwn\",
          \"text\": \"💚 *LIQUIDACIÓN — Pedido #{NUM_PEDIDO} · {NOMBRE}*\n💰 *Total pagado:* \${TOTAL} MXN\n✅ *Restante liquidado:* \${RESTANTE} MXN\n🔗 {LINK_MANYCHAT}\"
        }
      },
      {
        \"type\": \"image\",
        \"image_url\": \"$IMG_URL\",
        \"alt_text\": \"Comprobante liquidación #{NUM_PEDIDO}\"
      }
    ]
  }"
```

### Paso C5 — Activar flow "Confirmar Liquidación" en ManyChat

```bash
curl -s -X POST "https://api.manychat.com/fb/sending/sendFlow" \
  -H "Authorization: Bearer ${MANYCHAT_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"subscriber_id\": \"{SUBSCRIBER_ID}\",
    \"flow_ns\": \"content20250708192546_283376\"
  }"
```

### Output — Modo C

```
✅ Liquidación procesada

🛒 Shopify #XXXX — marcada como PAGADA
💚 Slack — comprobante de liquidación enviado a #confimacion-pagos-lienzos
📲 ManyChat — flow "Confirmar Liquidación" activado al cliente
```

---

## MODO B — Actualizar dirección de pedido existente

Activar cuando el usuario diga "ya tengo la dirección", "actualiza la dirección",
"agrega la dirección de [cliente/pedido]".

### Paso B1 — Validar y presentar resumen antes de actuar

```
📋 ACTUALIZACIÓN DE DIRECCIÓN — confirma antes de continuar

🛒 Pedido: #XXXX
👤 Cliente: [nombre]

🏠 Nueva dirección:
  • Calle y número:  [valor]  ← ⚠️ FALTANTE
  • Col./Referencias: [valor]
  • CP:              [valor]  ← ⚠️ FALTANTE
  • Ciudad:          [valor]  ← ⚠️ FALTANTE
  • Estado:          [valor]  ← ⚠️ FALTANTE

¿Actualizo en Shopify y Notion?
```

#### Alertas que BLOQUEAN:
- ⚠️ Pedido no encontrado en Shopify
- ⚠️ Faltan datos esenciales: calle, ciudad, CP o estado

**No ejecutar nada sin confirmación del usuario.**

### Paso B2 — Obtener Order ID de Shopify

```bash
curl --resolve "ah29ra-xt.myshopify.com:443:23.227.38.74" \
  "https://ah29ra-xt.myshopify.com/admin/api/2025-01/orders.json?name={NUMERO_PEDIDO}&status=any" \
  -H "X-Shopify-Access-Token: {TOKEN}"
```

### Paso B3 — Actualizar dirección en Shopify

Usar REST PUT (el MCP tool `shopify:update-order` carece de scope write_orders):
```bash
curl --resolve "ah29ra-xt.myshopify.com:443:23.227.38.74" \
  -X PUT "https://ah29ra-xt.myshopify.com/admin/api/2025-01/orders/{ORDER_ID}.json" \
  -H "X-Shopify-Access-Token: {TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "order": {
      "id": {ORDER_ID},
      "shipping_address": {
        "first_name": "{NOMBRE}",
        "last_name": "{APELLIDO}",
        "address1": "{CALLE_NUMERO}",
        "address2": "{COLONIA_REFERENCIAS}",
        "city": "{CIUDAD}",
        "province": "{ESTADO}",
        "country": "MX",
        "zip": "{CP}",
        "phone": "{TELEFONO}"
      }
    }
  }'
```

### Paso B4 — Actualizar Notion

Buscar la página por `No. Pedido` y actualizar el campo `Notas` reemplazando
`⚠️ Dirección pendiente` con la dirección completa.

### Output — Modo B

```
✅ Dirección actualizada

🛒 Shopify #XXXX — dirección agregada
📋 Notion — notas actualizadas
```

---

## Solución a errores de red frecuentes

### DNS cache overflow en llamadas a ManyChat o Shopify

**Síntoma:** `curl` devuelve literalmente `DNS cache overflow` en lugar de JSON.

**Causa:** El entorno del agente tiene un límite de caché DNS que se satura con muchas
llamadas seguidas. Es intermitente y temporal.

**Solución:**
1. Esperar 5-10 segundos con `sleep 5` y reintentar
2. Si persiste, agregar `--connect-timeout 10` a la llamada curl — esto fuerza una
   resolución DNS fresca en lugar de usar la caché saturada:

```bash
curl -s --connect-timeout 10 \
  -H "Authorization: Bearer ${MANYCHAT_API_TOKEN}" \
  "https://api.manychat.com/fb/subscriber/getInfo?subscriber_id={ID}"
```

3. Si aún falla, encadenar con `sleep` antes:
```bash
sleep 8 && curl -s --connect-timeout 10 ...
```

**Aplica tanto para ManyChat como para Shopify.** Para Shopify, siempre combinar
`--connect-timeout 10` CON `--resolve` — ambas flags juntas son la solución:
```bash
curl -s --connect-timeout 10 --resolve "ah29ra-xt.myshopify.com:443:23.227.38.74" \
  -X POST "https://ah29ra-xt.myshopify.com/admin/oauth/access_token" ...
```

> ⚠️ Solo `--resolve` sin `--connect-timeout 10` puede seguir dando DNS overflow.
> Solo `--connect-timeout 10` sin `--resolve` no bypasea el DNS de Shopify.
> **Siempre usar ambas juntas.**

---

## Notas técnicas

- **Slack canal comprobantes**: `C084HG6ANN6` (`#confimacion-pagos-lienzos`)
- **Slack Bot Token**: `${SLACK_BOT_TOKEN}`
- **Enviar imagen a Slack**: usar `chat.postMessage` con Block Kit (`blocks: [section + image]`). Slack jala la imagen directamente desde la URL de ManyChat — el agente NO necesita descargarla. Este método muestra la imagen a tamaño completo en el canal.
- **NUNCA** usar `files.getUploadURLExternal` para imágenes de ManyChat — el CDN de ManyChat (`manybot-files.manychat.io`) puede dar 503 desde el entorno del agente y el archivo llega vacío.
- **Texto del mensaje Slack**: incluir siempre número de pedido, nombre, apartado/restante y link ManyChat en el bloque `section`.
- **Shopify resolve**: siempre `--resolve "ah29ra-xt.myshopify.com:443:23.227.38.74"`
- **Token Shopify expira en 24h** — regenerar si responde 401
- **Múltiples piezas mismo tamaño**: una línea por pieza (no quantity > 1)
- **Sin descuento**: omitir bloque `applied_discount` completamente
- **Liquidaciones — Shopify**: usar GraphQL `orderMarkAsPaid` (más confiable que REST `/transactions` que falla sin transacción padre)
- **Payload con caracteres especiales**: guardar el JSON en `/tmp/payload.json` y usar `-d @/tmp/payload.json` en lugar de inline para evitar problemas de escaping en bash

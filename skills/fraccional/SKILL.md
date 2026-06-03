---
name: fraccional
description: Operate your Fraccional (fraccional.cl) real-estate investment account from the terminal — log in with an email code, view your portfolio, movements, valuations, rent payrolls and the secondary market, and run read queries against the Fraccional API with curl. Use when the user mentions Fraccional, fraccional.cl, or their investments / fractions / portfolio / "mis inversiones" / "mi portafolio" there.
metadata:
  author: urvana
  version: "0.1.0"
---

# Fraccional desde la terminal

Opera tu cuenta de **[Fraccional](https://www.fraccional.cl)** (inversión inmobiliaria
fraccionada) con `curl`. Login por código al email, sesión persistente con auto-refresh, y
consultas REST a la API (Supabase PostgREST). Todo lo tuyo, protegido por RLS por usuario.

> Esto opera **tu propia cuenta**. La sesión guardada da acceso total a ella. Cierra sesión
> con el bloque *Logout* cuando termines en una máquina compartida.

## Requisitos

- `curl` y `jq` instalados (`brew install jq` en macOS).
- Una cuenta en https://www.fraccional.cl (cualquier método: email o Google/Facebook/LinkedIn).

## Constantes (públicas)

Todos los bloques empiezan con estas 3 líneas. `KEY` es la *publishable key* (pública, ya viaja
en el sitio web — **no** es un secreto). La sesión sí es secreta y va en `SESSION`.

```bash
SUPABASE_URL="https://api.fraccional.app"
KEY="sb_publishable_pthBDDoTGZc7PxwqrIHn1Q_-wZ_ZY6c"
SESSION="$HOME/.fraccional/session.json"
```

## Reglas de seguridad (para el agente)

- **NUNCA** imprimas `access_token` ni `refresh_token` en pantalla (van al transcript). Los bloques
  los canalizan directo a `$SESSION` con `jq`.
- **NUNCA** uses `curl -v` ni `-i` en llamadas autenticadas (imprimen el header `Authorization`).
- **NUNCA** imprimas la respuesta cruda de `/auth/v1/otp`, `/auth/v1/verify` o `/auth/v1/token`
  (traen tokens). Solo extrae con `jq` lo no sensible (estado, email).
- El archivo `$SESSION` se crea con `chmod 600`.
- Las **mutaciones** (escribir/cancelar/comprar) se confirman con el usuario **antes** de ejecutar.

---

## Login (Email OTP) — recomendado

Funciona para **todos** los usuarios (incluidos los de Google/Facebook/LinkedIn): el código
llega al email de la cuenta. No usa contraseña → nada sensible pasa por el agente.

### Paso 1 — pedir el código

Pregunta el email al usuario y envíalo. `create_user:false` evita crear cuentas nuevas.

```bash
SUPABASE_URL="https://api.fraccional.app"
KEY="sb_publishable_pthBDDoTGZc7PxwqrIHn1Q_-wZ_ZY6c"
EMAIL="tucorreo@ejemplo.com"   # <-- pídeselo al usuario

curl -sS -X POST "$SUPABASE_URL/auth/v1/otp" \
  -H "apikey: $KEY" -H "content-type: application/json" \
  -d "{\"email\":\"$EMAIL\",\"create_user\":false}" \
  | jq -r 'if .msg then "ERROR: " + .msg + (if .error_code then " (" + .error_code + ")" else "" end)
           elif .error_description then "ERROR: " + .error_description
           else "✓ Código enviado a '"$EMAIL"'. Revisa tu correo." end'
```

El correo trae un **código de 6 dígitos** o un **enlace mágico**. Sirven ambos (ver Paso 2).

### Paso 2 — verificar y guardar la sesión

Pide al usuario el código (o que pegue el enlace completo del correo). Este mismo bloque sirve
para el código del email, el código de `/cli`, y los enlaces mágicos: prueba las variantes de
`type` hasta que una funcione.

```bash
SUPABASE_URL="https://api.fraccional.app"
KEY="sb_publishable_pthBDDoTGZc7PxwqrIHn1Q_-wZ_ZY6c"
SESSION="$HOME/.fraccional/session.json"
EMAIL="tucorreo@ejemplo.com"
INPUT="123456"   # <-- código de 6 dígitos, O el enlace completo (correo / página /cli)

mkdir -p "$(dirname "$SESSION")"

# Prueba un cuerpo de verificación; si trae access_token lo guarda y retorna 0.
verify_try() {
  local resp; resp=$(curl -sS -X POST "$SUPABASE_URL/auth/v1/verify" \
    -H "apikey: $KEY" -H "content-type: application/json" -d "$1")
  if printf '%s' "$resp" | jq -e 'has("access_token") and .access_token != null' >/dev/null 2>&1; then
    printf '%s' "$resp" | jq '{access_token, refresh_token, expires_at, email: .user.email}' > "$SESSION"
    chmod 600 "$SESSION"; return 0
  fi
  return 1
}

if printf '%s' "$INPUT" | grep -q '://'; then
  HASH=$(printf '%s' "$INPUT" | sed -n 's/.*[?&]token[=]\([^&]*\).*/\1/p')
  verify_try "{\"type\":\"magiclink\",\"token_hash\":\"$HASH\"}" \
    || verify_try "{\"type\":\"email\",\"token_hash\":\"$HASH\"}"
else
  verify_try "{\"type\":\"email\",\"email\":\"$EMAIL\",\"token\":\"$INPUT\"}" \
    || verify_try "{\"type\":\"magiclink\",\"email\":\"$EMAIL\",\"token\":\"$INPUT\"}"
fi

if [ -n "$(jq -r '.access_token // empty' "$SESSION" 2>/dev/null)" ]; then
  echo "✓ Sesión iniciada como $(jq -r '.email' "$SESSION")"
else
  rm -f "$SESSION"
  echo "✗ No se pudo verificar. Revisa el código/enlace o reenvía el código (Paso 1)."
fi
unset INPUT HASH
```

## Login por navegador (paste-code, tipo `/login`)

Para quien prefiere autenticarse en el navegador (reusa su sesión web / OAuth, sin escribir email
ni código). Requiere que la página `/cli` esté desplegada.

1. Abre **https://www.fraccional.cl/cli** (`open` en macOS).
2. El usuario inicia sesión (o ya lo está) y pulsa **Generar código**. La página muestra un
   **código** (y un enlace de respaldo) + su email.
3. Pega ese código (o el enlace) en el **Paso 2** de arriba — el bloque detecta el tipo solo.
   Se guarda una sesión **independiente** (no cierra la sesión web).

```bash
open "https://www.fraccional.cl/cli" 2>/dev/null || echo "Abre https://www.fraccional.cl/cli en tu navegador"
```

---

## Antes de cada llamada: refrescar token

El `access_token` dura ~1h. Corre este bloque **antes** de cualquier query: refresca si falta <60s
y **persiste el nuevo refresh_token** (Supabase lo rota en cada uso).

```bash
SUPABASE_URL="https://api.fraccional.app"
KEY="sb_publishable_pthBDDoTGZc7PxwqrIHn1Q_-wZ_ZY6c"
SESSION="$HOME/.fraccional/session.json"

[ -f "$SESSION" ] || { echo "No hay sesión. Corre el login primero."; exit 1; }
EXP=$(jq -r '.expires_at // 0' "$SESSION"); NOW=$(date +%s)
if [ "$NOW" -ge "$((EXP - 60))" ]; then
  RT=$(jq -r '.refresh_token' "$SESSION")
  resp=$(curl -sS -X POST "$SUPABASE_URL/auth/v1/token?grant_type=refresh_token" \
    -H "apikey: $KEY" -H "content-type: application/json" \
    -d "{\"refresh_token\":\"$RT\"}")
  if [ -n "$(printf '%s' "$resp" | jq -r '.access_token // empty')" ]; then
    printf '%s' "$resp" | jq '{access_token, refresh_token, expires_at, email: .user.email}' > "$SESSION.tmp" \
      && mv "$SESSION.tmp" "$SESSION" && chmod 600 "$SESSION"
    echo "✓ token refrescado"
  else
    echo "✗ refresh falló — vuelve a iniciar sesión"
  fi
  unset resp RT
else
  echo "✓ token vigente"
fi
```

## Correr una query (genérico)

Patrón base: lee el token del archivo, llama a PostgREST, muestra JSON. Cambia el path/filtros
según el *Catálogo* de abajo. (Corre primero el bloque de refrescar.)

```bash
SUPABASE_URL="https://api.fraccional.app"
KEY="sb_publishable_pthBDDoTGZc7PxwqrIHn1Q_-wZ_ZY6c"
SESSION="$HOME/.fraccional/session.json"
TOKEN=$(jq -r '.access_token' "$SESSION")

curl -sS "$SUPABASE_URL/rest/v1/profiles?select=*" \
  -H "apikey: $KEY" -H "Authorization: Bearer $TOKEN" | jq
unset TOKEN
```

## Logout

```bash
rm -f "$HOME/.fraccional/session.json" && echo "✓ sesión cerrada"
```

---

## Catálogo de queries típicas (PostgREST `/rest/v1`)

La RLS scopea automáticamente "lo tuyo": consultar `profiles` o `purchase_confirmations_pnls`
ya devuelve solo tus filas. Reemplaza el `?path?query` del bloque genérico por:

| Intención | Endpoint |
|---|---|
| Mi perfil (curado) | `rpc/viewer_profile` *(POST, ver abajo)* |
| Mi perfil (tabla) | `profiles?select=*` |
| Mis movimientos / PnL | `purchase_confirmations_pnls?select=*&order=confirmed_at.desc&limit=20` |
| Resumen portafolio por moneda | `profile_portfolio_summary_by_currency?select=*` |
| Mis cuentas bancarias | `profile_bank_accounts?select=*` |
| Mis ventas activas (asks) | `profile_asks?select=*` |
| Mercado: asks más baratos de una unidad | `secondary_market_orders?order_type=eq.ask&unit_id=eq.<ID>&order=token_price.asc&limit=5` |
| Mercado: mejores bids de una unidad | `secondary_market_orders?order_type=eq.bid&unit_id=eq.<ID>&order=token_price.desc&limit=5` |
| Propiedades / unidades | `units?select=id,name,slug,funding_currency,funding_amount,is_sold_out,rent_state&order=published_at.desc&limit=20` |
| Detalle de una unidad | `units?id=eq.<ID>&select=*` |
| Valuaciones de una unidad | `unit_valuations?unit_id=eq.<ID>&select=*&order=created_at.desc` |

> `units` usa PK **`id`**; las tablas relacionadas la referencian como **`unit_id`**.

### RPC curados (POST)

Algunos atajos de la app son funciones. Se llaman con POST y cuerpo `{}` (o args):

```bash
SUPABASE_URL="https://api.fraccional.app"
KEY="sb_publishable_pthBDDoTGZc7PxwqrIHn1Q_-wZ_ZY6c"
SESSION="$HOME/.fraccional/session.json"
TOKEN=$(jq -r '.access_token' "$SESSION")

curl -sS -X POST "$SUPABASE_URL/rest/v1/rpc/viewer_profile" \
  -H "apikey: $KEY" -H "Authorization: Bearer $TOKEN" \
  -H "content-type: application/json" -d '{}' | jq
unset TOKEN
```

## Cheatsheet PostgREST

- **Filtros**: `?columna=eq.valor` (`eq, neq, gt, gte, lt, lte, like, ilike, in, is`).
  Ej: `?status=in.(active,pending)`, `?name=ilike.*centro*`.
- **Columnas**: `?select=id,name,funding_amount`.
- **Orden**: `?order=created_at.desc` (encadena: `?order=a.desc,b.asc`).
- **Paginación**: `?limit=20&offset=40`.
- **Embedding (joins)**: `?select=*,unit_valuations(*)` (tabla relacionada anidada).
- **Conteo exacto**: agrega header `-H "Prefer: count=exact"` y mira `Content-Range` en la respuesta.
- **Schema**: solo `public` (default). No se necesita header extra.

## Descubrir más

- ¿No conoces las columnas? Pide 1 fila: `...?select=*&limit=1` y mira las claves con `jq '.[0]|keys'`.
- ¿Buscas otra tabla/función? La RLS te limita a `public`. Prueba el nombre; si no existe,
  PostgREST devuelve `404` o sugiere el correcto en `hint`.
- Existe también GraphQL en `POST /rest/v1/graphql_extended` y `/graphql/v1` (queries raw, misma
  RLS). REST cubre la mayoría de los casos y es más compacto.

## Mutaciones (con confirmación)

PostgREST también escribe. **Confirma con el usuario antes de ejecutar** cualquiera de estas.

- Insertar fila: `POST /rest/v1/<tabla>` con header `-H "Prefer: return=representation"` y cuerpo JSON.
- Actualizar: `PATCH /rest/v1/<tabla>?<filtro>` con cuerpo JSON.
- Funciones de la app (curadas): `POST /rest/v1/rpc/<fn>`, p.ej.
  `rpc/viewer_profile_ask_cancel` (cancelar una venta), `rpc/viewer_activate_promo_code`
  (activar código). Pasa los argumentos en el cuerpo JSON.

## Troubleshooting

| Síntoma | Causa / Fix |
|---|---|
| `JWT expired` / `PGRST301` | Corre el bloque *refrescar token*. Si persiste, vuelve a iniciar sesión. |
| `401` en todas las queries | Falta `Authorization: Bearer` o sesión inválida → re-login. |
| `column X does not exist` | Mira el `hint`; usa `select=*&limit=1` para ver columnas reales. |
| `404` en una tabla | No existe o no está expuesta en `public`. Verifica el nombre. |
| Respuesta vacía `[]` | RLS: no tienes filas para ese recurso (esperado si no es tuyo). |
| `Código enviado` pero no llega | Revisa spam; el remitente es Fraccional. Reintenta el Paso 1. |
| `Signups not allowed for otp` | El email no está registrado en Fraccional, o el OTP por email está desactivado. Verifica el correo o usa el *Login por navegador* (`/cli`). |
| `jq: command not found` | `brew install jq`. |

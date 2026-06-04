# fraccional-skills

[Agent Skills](https://docs.claude.com/en/docs/claude-code/skills) para operar tu cuenta de
**[Fraccional](https://www.fraccional.cl)** (inversión inmobiliaria fraccionada) desde la terminal,
usando solo `curl`.

El skill maneja tu **sesión** (login por código al email, auto-refresh, logout) y documenta las
**consultas típicas** contra la API de Fraccional (Supabase PostgREST), todo protegido por RLS por
usuario.

## Instalar

```bash
npx skills add urvana/fraccional-skills --agent claude-code -y
```

Página web con instrucciones (humanos + agentes): https://www.fraccional.cl/agente
(versión markdown: https://www.fraccional.cl/agente.md)

Luego, en Claude Code, pídele cosas como:

- "inicia sesión en Fraccional"
- "muéstrame mi portafolio / mis movimientos"
- "¿cuáles son los asks más baratos de la unidad X en el mercado secundario?"

## Mejores ofertas del mercado secundario

La página pública [/oportunidades](https://www.fraccional.cl/oportunidades) lista las **mejores
ofertas (asks) del mercado secundario**, ordenadas por su descuento respecto al precio de
referencia. Esa misma vista se puede consultar por PostgREST con la función
`profile_asks_summary_grouped_as` (respaldada por la vista `profile_asks_summary`), **sin iniciar
sesión** — los datos son públicos:

```bash
SUPABASE_URL="https://api.fraccional.app"
KEY="sb_publishable_zsc5qocAVfPLJnD9IOHrpw_mtUlCg-7"

curl -sS -X POST \
  "$SUPABASE_URL/rest/v1/rpc/profile_asks_summary_grouped_as?select=unit_id,currency,asks_count,token_quantity,token_price,reference_token_price,asked_amount,reference_diff_pct,first_ask_public_serial&order=reference_diff_pct.asc.nullslast&limit=20" \
  -H "apikey: $KEY" -H "content-type: application/json" \
  -d '{"indicator_id":"CLP"}' | jq
```

- `indicator_id` (cuerpo): moneda de salida (`CLP`, `UF`, `USD`, …). Convierte todos los montos.
- `reference_diff_pct`: diferencia del `token_price` vs. el precio de referencia de la unidad.
  **Negativo = bajo la referencia** (mejor oportunidad). `order=reference_diff_pct.asc.nullslast`
  pone las mejores primero (igual que la página).
- Filtra solo las ofertas bajo la referencia con `&reference_diff_pct=lt.0`.
- `token_price` = precio por fracción; `asked_amount` = monto total del grupo de asks.

El [skill](skills/fraccional/SKILL.md) trae este y otros endpoints en su catálogo de queries.

## Cómo funciona

- **API**: Supabase PostgREST en `https://api.fraccional.app/rest/v1`. La `publishable key`
  (pública, ya viaja en el sitio web) va embebida; **no** es un secreto.
- **Sesión**: tu `access_token` (~1h) + `refresh_token` (durable) se obtienen al iniciar sesión y se
  guardan en `~/.fraccional/session.json` con permisos `600`. El skill refresca automáticamente y
  persiste la rotación del `refresh_token`.
- **Login**: código de 6 dígitos (o enlace mágico) enviado a tu email — **sin contraseña**, así nada
  sensible pasa por el agente. También hay login por navegador (`/cli`, estilo `/login`).

## Requisitos

- `curl` y `jq` (`brew install jq` en macOS). Eso es todo — sin Node, sin scripts ejecutables.

## Seguridad

- **Markdown puro**: el skill no trae binarios ni scripts opacos. Cada comando `curl` es visible en
  `skills/fraccional/SKILL.md` antes de ejecutarse.
- La sesión da **acceso total a tu cuenta**. En máquinas compartidas, cierra sesión:
  `rm -f ~/.fraccional/session.json`.
- La RLS del servidor te limita a **tus propios datos**.
- Los comandos nunca imprimen tokens en pantalla.

## Estructura

```
skills/
└── fraccional/
    └── SKILL.md   # flujos (login/refresh/query/logout) + catálogo REST + cheatsheet
```

## Licencia

MIT © urvana

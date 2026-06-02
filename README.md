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

Luego, en Claude Code, pídele cosas como:

- "inicia sesión en Fraccional"
- "muéstrame mi portafolio / mis movimientos"
- "¿cuáles son los asks más baratos de la unidad X en el mercado secundario?"

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

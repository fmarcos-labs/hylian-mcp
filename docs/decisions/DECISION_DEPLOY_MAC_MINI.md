# DECISION: Supabase local y deploy en el Mac Mini, en vez de Supabase cloud y npm

Fecha: 2026-10-29

## Qué dice el concepto

`_CONCEPTO.md` pide "Storage en Supabase (Postgres cloud, free tier)" y distribución con `npm publish`, y deja fuera
"Servicio cloud deployado (corre local)".

## Qué se hizo y por qué

- **Supabase local** (CLI + Docker, `supabase/config.toml`) en vez de un proyecto cloud. El dueño lo eligió el
  2026-10-28 para tener el server funcionando ya, sin crear el proyecto. Los repositorios solo conocen
  `SUPABASE_URL` y la key: pasar a la nube es cambiar dos líneas del `.env` y correr `schema.sql` allá.
- **Host: el Mac Mini M1**, siempre prendido. Sigue siendo "corre local": máquina propia, nada en un cloud.
- **Deploy por push** (`.github/workflows/deploy.yml`), pedido el 2026-10-29: "hacer commits acá, y que después de
  yo pushear automáticamente debería deployarse". Usa el runner self-hosted de `fmarcos-labs` que ya deploya otros
  repos; por eso el repo vive en esa org (y `package.json` apunta ahí). Corre solo si CI pasó.
- **Imagen Docker en vez de npm**: Claude Code del Mac lanza `hylian-mcp:local` por stdio. El paquete npm sigue sin
  publicarse; el `bin` y `prepack` quedan como estaban.

- **Una sola base, la del Mac** (2026-11-01): el PC registra el server con `ssh mac-mini docker run …` en vez de
  apuntar su `.env` al Supabase del Mac por Tailscale. Mismo resultado sin que el PC dependa de alcanzar la base, y
  el PC usa siempre la imagen deployada. El Supabase local del PC queda solo para desarrollo.

## Qué queda abierto

- El CLI de Supabase publica API (54321), Postgres (54322, clave `postgres`) y Studio (54323, sin login) en
  `0.0.0.0`, sin opción para cambiarlo: el Mac los expone a la LAN y al tailnet. Probado 2026-11-01:
  `com.docker.network.bridge.host_binding_ipv4=127.0.0.1` en la red del proyecto no tiene efecto en Docker Desktop
  (solo respeta el `127.0.0.1` declarado por contenedor), y el `ip` global del daemon dejaría en localhost a Home
  Assistant, Strapi y agriplan, que publican sin IP igual que Supabase.

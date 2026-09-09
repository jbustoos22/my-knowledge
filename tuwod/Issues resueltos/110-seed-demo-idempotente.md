---
issue: 110
repository: JavierxHernandez/tuwod
status: implemented-pr-open
pr: 111
branch: fix/seed-demo-idempotent
tags:
  - tuwod
  - supabase
  - seed
  - development
---

# #110 — Hacer `seed-demo.sql` idempotente

- **Issue:** https://github.com/JavierxHernandez/tuwod/issues/110
- **PR:** https://github.com/JavierxHernandez/tuwod/pull/111
- **Estado:** implementado y validado; el PR sigue abierto contra `develop` y no se configuró auto-merge.

## Problema

`supabase/seed-demo.sql` llena la base local con atletas, WODs y resultados para poder probar la interfaz con contenido. Al ejecutarlo una segunda vez intentaba borrar los miembros demo de UUID fijo. Esos miembros podían estar referenciados por tablas hijas como `week_results`, por lo que una FK abortaba la transacción completa.

El archivo también creaba los WODs como texto sin sembrar movimientos estructurados, cantidades ni cargas por división. Eso impedía probar en local las pantallas que dependen de `workout_movements`.

## Decisión

No enumerar y borrar cada posible tabla dependiente de `members`: esa solución volvería a fallar cada vez que se añadiera una relación nueva. Los seis atletas demo tienen UUID fijos, así que el seed debe reutilizarlos mediante una inserción idempotente.

Los WODs de muestra sí se reemplazan porque sus UUID también son fijos. Antes se borran sus resultados y luego se borran los WODs; los movimientos y cargas asociados se eliminan mediante `CASCADE` y se reconstruyen completos.

## Cambios realizados

En `supabase/seed-demo.sql`:

- Los miembros demo se insertan con `ON CONFLICT (id) DO NOTHING`; ya no se borran.
- Los WODs demo y sus resultados se limpian exclusivamente por sus UUID fijos.
- Los títulos semanales usan `ON CONFLICT (box_id, week_start, title_slug, member_id) DO NOTHING`.
- Se añadió la parte C del WOD del día:
  - **Run and lift** · `for_time` · cap de 11 minutos · 5 rondas.
  - `200 m Run`.
  - `10 Hang squat clean`.
  - Cargas A/B en `kg × 100`: RX `80/60`, scaled `70/45`, beginner `60/40`.
- Si el schema contiene la columna futura `workouts.schema_source`, las partes directas se marcan como `text`; el seed sigue siendo compatible con el schema local anterior a esa migración.

## Validación

Se recreó la base local y se ejecutó el seed dos veces consecutivas. Las dos ejecuciones terminaron sin errores y dejaron el mismo estado canónico para miembros, WODs, movimientos, cargas, resultados, títulos y semanas.

Snapshot de equivalencia:

```text
2a1b931458c93724e52055d920d6b2f7
```

También pasaron las validaciones del proyecto:

```text
npm run lint    # sin errores; 3 advertencias existentes de no-img-element
npm test        # 469 tests correctos
npm run db:test # 661 tests pgTAP correctos
```

## Seguimiento

La rama `fix/seed-demo-idempotent` contiene el commit:

```text
1356a52 fix(seed): make demo seed idempotent
```

El merge en `develop` queda pendiente de la revisión requerida en el PR #111. No se debe usar `--admin` para saltar la protección de la rama.

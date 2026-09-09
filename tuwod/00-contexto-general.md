---
project: tuwod
type: project-context
tags:
  - tuwod
  - architecture
  - supabase
  - nextjs
  - crossfit
---

# tuwod — Contexto general del proyecto

## Qué es

**tuwod** es una PWA multi-tenant para boxes de CrossFit y entrenamiento funcional. Cada box publica el WOD del día, sus atletas registran resultados y toda la comunidad consulta el leaderboard diario, comentarios, reacciones, rachas, PRs y títulos semanales.

El producto está pensado para tres superficies principales:

- La app privada del box para atletas y coaches.
- Una ficha pública por box con su identidad, información y enlaces.
- Una pantalla pública de pared (`/tv/[slug]`) para mostrar el entrenamiento y el leaderboard.

La aplicación está en producción en `tuwod.app` y ya la utiliza un box real.

## Problema que resuelve

Un box necesita publicar el entrenamiento, permitir que atletas presenciales y remotos registren su resultado sin fricción, y hacer visible el progreso y la participación sin convertir la experiencia en un ranking acumulado de personas.

El leaderboard ordena únicamente el WOD actual y separa divisiones. La intención es que atletas de niveles distintos puedan participar sin quedar siempre al final de una única tabla.

## Modelo de tenancy y URLs

Cada box vive en un subdominio propio, por ejemplo:

```text
sweetspot.tuwod.app
```

No se usa una URL basada en path como `/sweetspot`. El subdominio es una decisión de arquitectura: service workers, storage del navegador y suscripciones push dependen del origen. Cambiar ese esquema después rompería instalaciones PWA y push existentes.

En desarrollo se usa `sweetspot.localhost:3000`; el proxy de Next reescribe el subdominio al árbol `app/[box]`. `lib/tenant.ts` es la fuente única de verdad para decidir el tenant.

## Stack

| Área | Tecnología |
|---|---|
| Aplicación | Next.js App Router, React, TypeScript, Tailwind |
| Datos y auth | Supabase: Postgres, Auth, RLS, Realtime y Storage |
| Seguridad de datos | RLS por operación y rol; RPCs `security definer` para escrituras de negocio |
| Jobs programados | `pg_cron` y `pg_net` hacia Edge Functions de Supabase |
| Deploy | Vercel con wildcard para `*.tuwod.app` |
| Autenticación | Passwordless por email: OTP de seis dígitos como flujo principal y magic link como conveniencia de escritorio |

La interfaz visible se escribe en español desde `lib/i18n/es.ts`; identificadores, enums, rutas públicas y commits se mantienen en inglés.

## Arquitectura de datos y seguridad

El proyecto prioriza que la seguridad resida en Postgres, no en condiciones dispersas de la aplicación:

- Cada tenant se aísla mediante `box_id` y políticas RLS.
- Las escrituras con reglas de negocio pasan por RPCs en la base de datos.
- Los grants de columnas complementan RLS: el uso de `service_role` no debe exponer columnas no concedidas.
- Las pruebas de base verifican aislamiento entre boxes, normalización de scores, grants y reglas de negocio.

Los resultados almacenan una forma normalizada (`score_norm`) para que el leaderboard sea ordenable de manera consistente. La dirección depende del tipo de score: menor tiempo gana; más carga, repeticiones o rondas gana.

## Funcionalidades principales

- Publicar WODs de una o varias partes, con movimientos, cantidades, cargas por división y cap.
- Registrar resultados para atletas en el box o remotos.
- Leaderboards por división y filtros de ubicación.
- Comentarios y reacciones.
- Rachas semanales, PRs, logros y títulos semanales.
- Invitaciones y gestión de miembros para coaches.
- Ficha pública del box, pantalla de pared y tarjetas compartibles de resultados.
- Push para el WOD del día siguiente y recordatorio de instalación PWA.
- Catálogo de movimientos y activos de referencia; el contrato visual vigente se define en `docs/SPEC.md`.

## Estructura relevante del repositorio

```text
app/                 rutas de Next; app/[box] contiene la aplicación de cada tenant
lib/tenant.ts        resolución y seguridad de tenant
lib/rig/             motor y edición de poses/movimientos existente en el proyecto
lib/i18n/es.ts       strings de interfaz en español
supabase/migrations/ schema canónico y evolución de la base
supabase/tests/      pruebas pgTAP
supabase/functions/  Edge Functions, incluido el push diario
proxy.ts             subdominio -> árbol app/[box]
```

## Desarrollo local

El entorno local usa Supabase en Docker. El flujo habitual es:

```bash
supabase start
npm run db:reset
npm run db:admin
npm run dev
```

`db:reset` aplica migraciones y el seed mínimo. `db:admin` debe ejecutarse después de cada reset porque `auth.users` pertenece a GoTrue y se vacía durante el reset.

Para poblar la interfaz con atletas, WODs y resultados de ejemplo se ejecuta `supabase/seed-demo.sql`. El issue #110 hizo ese seed idempotente y añadió un WOD estructurado con movimientos y cargas.

## Estado actual y prioridades

El producto ya funciona con un box real para publicar WODs, registrar resultados, mostrar leaderboards por división, gestionar miembros e invitaciones, exponer la ficha pública y pantalla de pared, y enviar push.

Antes de abrir el producto a un segundo box, las prioridades conocidas incluyen completar los activos de movimientos iniciales, automatizar el alta de boxes nuevos, soportar overrides por box para nombres y estándares, y finalizar el material de privacidad/DPA del onboarding.

## Fuentes de verdad

- `docs/SPEC.md`: contrato funcional, schema y decisiones canónicas; prevalece ante conflictos.
- `CLAUDE.md`: razones arquitectónicas, anti-features y convenciones.
- `README.md`: visión de producto, arranque local, estructura y comandos cotidianos.
- `CONTRIBUTING.md`: ramas, commits, validaciones y política de PRs.

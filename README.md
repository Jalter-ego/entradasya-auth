# auth · EntradasYa (P1)

Servicio de registro, inicio de sesión y emisión de JWT. Escucha en `127.0.0.1:3001` y usa la base propia `auth_db` con el usuario `auth_user`. El contrato está en [auth.openapi.yaml](https://github.com/Jalter-ego/entradasya-platform/blob/feat/auth/contracts/auth.openapi.yaml).

## Arranque local

1. Preparar una base PostgreSQL para Auth y un usuario con permisos para migrarla. Se requiere Node 22.13 o superior y la versión de pnpm indicada en `package.json`.
2. Desde la raíz del repositorio, ejecutar `pnpm install --frozen-lockfile` y copiar `.env.example` a `.env` (`cp .env.example .env` o `Copy-Item .env.example .env` en PowerShell). Completar `DATABASE_URL` y la configuración siguiente.
3. Ejecutar `pnpm run keys:generate` para generar un par nuevo. Para reutilizar un par, guardar el PEM privado en `keys/auth-private.pem` y el público correspondiente en `keys/auth-public.pem`. Usar un mismo par para todo el entorno.
4. Ejecutar `pnpm run migrate`. Para la demo, definir `DEMO_PASSWORD` en `.env` y ejecutar `pnpm run seed`.
5. Ejecutar `pnpm run dev` o `pnpm run build` seguido de `pnpm start`. Comprobar `curl http://127.0.0.1:3001/health`, que verifica conexión y acceso a la tabla `users`.

| Variable | Valor / uso |
|---|---|
| `HOST`, `PORT` | `127.0.0.1`, `3001`. |
| `DATABASE_URL`, `DB_SCHEMA` | Conexión a `auth_db` y schema `public` por defecto. |
| `DATABASE_SSL`, `DATABASE_SSL_REJECT_UNAUTHORIZED` | TLS desactivado en local y verificación del certificado cuando se activa TLS. |
| `JWT_PRIVATE_KEY_FILE` / `JWT_PRIVATE_KEY` | PEM privado RSA ≥ 2048 bits. El archivo tiene prioridad. El contenido en variable admite saltos literales `\n`. |
| `JWT_ALG`, `JWT_ISSUER`, `JWT_AUDIENCE` | `RS256`, `entradasya-auth`, `entradasya`. |
| `JWT_TTL_SECONDS` | Vigencia por defecto de 28.800 segundos (8 horas). Rango de 1 a 86.400. |
| `DEMO_PASSWORD` | Contraseña local de 8 a 128 caracteres, usada por la semilla al crear las cuentas. |
| `LOG_LEVEL` | `silent` silencia los logs JSON. |

Para el entorno de Supabase documentado por P6, usar el Session Pooler (5432) o conexión directa, `DB_SCHEMA=auth_app` y `DATABASE_SSL=true`. La cuenta SQL debe poder migrar su schema y operar como dueña de la tabla o tener `BYPASSRLS`, según la preparación de ese entorno. La migración habilita RLS con acceso para el dueño de la tabla. Para la entrega local se usa la base y el usuario exclusivos.

## Cuentas de demo

| Correo | Rol | UUID |
|---|---|---|
| `organizador@demo.bo` | `organizer` | `00000000-0000-4000-8000-000000000001` |
| `cliente@demo.bo` | `customer` | `00000000-0000-4000-8000-000000000002` |
| `portero@demo.bo` | `doorman` | `00000000-0000-4000-8000-000000000003` |

Usar la contraseña configurada al ejecutar la semilla por primera vez. Repetirla verifica las identidades y conserva las contraseñas existentes. El registro público solo crea clientes y reserva estos tres correos para la semilla.

## Integración JWT

- Events, orders, tickets y checkin leen el PEM público mediante `JWT_PUBLIC_KEY_FILE` o `JWT_PUBLIC_KEY`. Configurar el mismo algoritmo, emisor y audiencia de auth. Guardar el PEM privado únicamente en auth.
- Los endpoints públicos de auth reciben JSON y devuelven `{ token, user }`. El registro devuelve `201` e inicia sesión. El login devuelve `200`. Para obtener un token de ejemplo válido, iniciar sesión en el servicio local con una cuenta sembrada. Usar `Authorization: Bearer <token>` en las APIs protegidas.
- Los consumidores comprueban el JWT localmente. `sub` corresponde al UUID guardado, `role` y `email` salen de la cuenta e `iat` y `exp` son segundos Unix. El cierre de sesión es local al navegador. Un JWT emitido conserva su vigencia hasta `exp`.

Las contraseñas se guardan con scrypt y una sal aleatoria por cuenta. Los parámetros `N=32768`, `r=8` y `p=3` siguen la [guía de almacenamiento de contraseñas de OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html).

No subir `.env`, archivos PEM, tokens ni datos de PostgreSQL al repositorio.

## Paquete de GitHub Actions

Cada push a `develop` ejecuta el chequeo de tipos, las pruebas con PostgreSQL 16 y la compilación. En la ejecución de **CI Auth**, la sección **Artifacts** ofrece el reporte JUnit y, si pasan las verificaciones, el archivo `entradasya-auth-<versión>-<sha>-<intento>.tgz`.

Extraer el archivo y ejecutar los siguientes comandos desde el directorio extraído con Node 22.13 o superior y la versión de pnpm indicada en `package.json`. Configurar primero `DATABASE_URL`, la clave privada RSA mediante `JWT_PRIVATE_KEY_FILE` o `JWT_PRIVATE_KEY` y las variables del entorno. Se pueden inyectar directamente o definir en un `.env` creado a partir de `.env.example`. Si la plataforma enruta tráfico al servicio desde fuera del proceso, ajustar `HOST` y `PORT` según su configuración.

```bash
pnpm install --prod --frozen-lockfile
pnpm run migrate:prod
pnpm start
```

Verificar que `GET /health` devuelva HTTP 200. El paquete incluye JavaScript compilado, las migraciones SQL y el generador de claves para preparar un entorno nuevo.

## Pruebas

```bash
pnpm run typecheck
pnpm run build
pnpm test
```

Para incluir PostgreSQL, crear una base desechable `auth_test` o `auth_test_*` con un usuario dueño. Definir `TEST_DATABASE_URL` en el entorno o en `.env.test`. Las pruebas migran un schema temporal, hacen peticiones HTTP reales y limpian el schema al terminar.

# Plan de Implementación — OrderFlow
_SvelteKit 2 • Svelte 5 • TypeScript • Prisma • PostgreSQL • Tailwind CSS 4 • Docker_

> Objetivo: “Tomar pedidos online, gestionarlos en tiempo real desde el admin y notificar estados al cliente”.  
> Este plan es **paso a paso**, listo para ejecutar de cero a producción.

---

## 0) Requisitos previos
- Node.js LTS y pnpm/npm instalados.
- Docker y docker-compose.
- Cuenta en Vercel / Fly.io / Railway (elegir una).
- Claves de proveedor de pago (si aplica): Mercado Pago y/o Stripe.
- Una línea de WhatsApp o e‑mail transaccional (opcional).

---

## 1) Bootstrap del proyecto
1. **Clonar repo** o iniciar nuevo:
   ```bash
   npx sv create orderflow --template skeleton
   cd orderflow
   pnpm add -D typescript @types/node eslint prettier vitest playwright
   pnpm add @prisma/client prisma zod @sveltejs/adapter-node
   pnpm add -D @playwright/test
   ```
2. **Tailwind 4** (postcss-free):
   ```bash
   pnpm add -D tailwindcss
   npx tailwindcss init -p
   ```
3. **Estructura modular** (crear carpetas):
   - `src/lib/{components,server,db,validation,realtime,payments,notifications,utils}`
   - `src/routes/{cart,checkout,order/[id],admin,...}`
4. **Scripts en package.json** (agregar):
   - `"dev": "vite"`
   - `"dev:docker": "docker compose up --build"`
   - `"build": "vite build"`
   - `"preview": "vite preview --host"`
   - `"lint": "eslint ."`
   - `"format": "prettier --write ."`
   - `"test": "vitest run"`
   - `"e2e": "playwright test"`
   - `"migrate": "prisma migrate deploy"`
   - `"seed": "ts-node prisma/seed.ts"` (o `node --loader ts-node/esm`)

---

## 2) Base de datos (Prisma + PostgreSQL)
1. **docker-compose.yml** con `db` y `app` (Node).
2. **Schema Prisma**: crear modelos `User, Category, Product, Variant, Customer, Order, OrderItem, AuditLog` y opcional `Payment, InventoryMovement`.
3. **Índices**: `Order.status`, `Order.createdAt`, `Payment.providerRef`.
4. **Migraciones**:
   ```bash
   npx prisma init
   # Configurar DATABASE_URL en .env
   npx prisma migrate dev --name init
   npx prisma generate
   ```
5. **Seeder**: categorías, productos/variantes demo, 1 admin y 3–5 pedidos.

**Checklist DB**
- [ ] Totales calculados en transacción (`Order` + `OrderItem`).
- [ ] `lineTotal = qty * unitPrice` y `total = sum(lineTotal)`.
- [ ] Estados como enum o tabla parametrizable.
- [ ] Auditoría (`AuditLog`) en cambios críticos.

---

## 3) Frontend público (Catálogo → Carrito → Checkout → Confirmación)
1. **Catálogo** (`/`):
   - Lista de categorías y productos activos.
   - Variantes (tamaño/sabor) con `priceBase + priceDelta`.
2. **Carrito** (`/cart`):
   - Agregar/Quitar, notas del cliente, total dinámico.
   - Método de entrega: **RETIRO** o **ENVÍO**.
3. **Checkout** (`/checkout`):
   - Validación con **Zod** (cliente y servidor).
   - Estados de carga/éxito/error y manejo de fallos.
   - Si hay pago online: generar `checkoutUrl` con adapter.
4. **Confirmación** (`/order/[id]`):
   - Número/ID, resumen imprimible y botón **Imprimir**.
5. **Accesibilidad + SEO**:
   - Metadatos, Open Graph, labels y foco correcto.
   - Responsive con Tailwind.

**Checklist público**
- [ ] SSR habilitado; caché controlada donde aplique.
- [ ] Input sanitizado; escape de HTML en notas.
- [ ] No exponer info sensible en el cliente.

---

## 4) Panel /admin (tempo real)
1. **Auth simple y segura** (env o bcrypt + cookie httpOnly).
2. **Cola de pedidos** (SSE por defecto; WS opcional):
   - Ver/priorizar/cambiar estado.
   - Edición rápida antes de “PREPARANDO”.
   - Notas internas, asignación a operador.
3. **Búsqueda y filtros** (estado, fecha, canal); **exportar CSV**.
4. **Impresión de ticket** 80mm (CSS `@media print`).
5. **Dashboard**: pedidos/día, ticket promedio, top productos.

**Checklist admin**
- [ ] Paginar resultados.
- [ ] Bloqueos optimistas o verificación de versión al editar.
- [ ] Auditoría de acciones (actor, acción, entidad, meta).

---

## 5) Realtime (SSE/WS)
1. **Event bus** interno (Node `EventEmitter` o Redis pub/sub).
2. **Endpoint SSE** `GET /admin/api/realtime`:
   - Enviar `order:update` en cambios.
   - Ping/keepalive cada ~25s.
3. **WS** (solo si Fly/Railway o servidor dedicado):
   - Namespace `/admin/api/websocket`.
4. **Reconexión y backoff** en el cliente.

**Checklist realtime**
- [ ] Límite de conexiones concurrentes.
- [ ] No enviar datos sensibles por eventos.
- [ ] Registrar métricas de conexiones.

---

## 6) Pagos (opcional, provider-agnostic)
1. **Interfaz PaymentProvider** (`createPayment`, `verifyWebhook`).
2. **Adapters**: `mercadopago.ts`, `stripe.ts`.
3. **Webhook** `POST /api/payments/webhook`:
   - Idempotencia por `providerRef`.
   - Transacción: registrar `Payment` y actualizar `Order`.
4. **Retorno del proveedor**: redirigir a `/order/[id]`.
5. **Monedas en enteros** (centavos) y redondeos consistentes.

**Checklist pagos**
- [ ] Validar firma del webhook.
- [ ] Reintentos y logs de errores.
- [ ] Timeouts y circuit breaker mínimo.

---

## 7) Notificaciones (opcionales)
1. **WhatsApp**: deep-link con mensaje prellenado al pasar a **LISTO**.
2. **Email** (si se usa): plantillas simples (subject/body) editables.
3. **Plantillas** en archivos `.ts/.md` para fácil edición.

**Checklist notificaciones**
- [ ] Opt-in del cliente.
- [ ] Rate-limit por pedido.
- [ ] Log de envíos y reintentos.

---

## 8) Seguridad y calidad
1. **CSRF** en actions POST.
2. **Rate-limit** por IP + ruta.
3. **Sanitización** de entradas (notas, address).
4. **Errores centralizados** (`lib/server/errors.ts`).
5. **Headers** seguros: `X-Frame-Options`, `Content-Security-Policy` (según hosting).
6. **.env** estricto y `.env.example` completo.
7. **ESLint + Prettier** configurados.
8. **Roles** (si hay múltiples operadores).

**Checklist seguridad**
- [ ] Cookies `httpOnly`, `secure`, `sameSite`.
- [ ] No guardar tokens de pago en texto plano.
- [ ] Backups de DB y rotación de logs.

---

## 9) Testing
1. **Unit (Vitest)**:
   - Cálculos de precio/stock/estados.
   - Utils de dinero y formateadores.
2. **E2E (Playwright)**:
   - **Alta pedido → cambio de estado → notificación**.
   - Flujos de error: stock insuficiente, validación, pago rechazado.
3. **Coverage** mín. 80% en dominios críticos.
4. **Seeds de test** y DB efímera (Docker).

**Checklist CI**
- [ ] Job CI: lint + unit + e2e headless.
- [ ] Artefactos: reportes JUnit/HTML.
- [ ] Gate: bloquear merge si falla.

---

## 10) DX y DevOps
1. **Docker**:
   - `docker-compose up --build` levanta app + PostgreSQL.
   - Volúmenes para persistencia local.
2. **Scripts**:
   - `dev:docker`, `migrate`, `seed`, `test`, `e2e`.
3. **README** profesional con troubleshooting: SSL DB, CORS, reloj/cron, impresión térmica.
4. **Observabilidad** (mínimo):
   - Logs estructurados (nivel INFO/ERROR).
   - Métricas básicas: pedidos/día, tiempos de respuesta.
   - Endpoint `/health` (200 OK) y `/version` (semver).

---

## 11) Deploy
### Opción A — Vercel
- ✅ Ideal para SSR + **SSE** (WS no recomendado).
- Configurar variables: `DATABASE_URL` (DB gestionada con SSL), `AUTH_SECRET`, `PUBLIC_*`, `PAYMENT_*`.
- **Release command**: `npx prisma migrate deploy`.
- Agregar `VERCEL_REGION` si geo importa.

### Opción B — Fly.io / Railway
- ✅ Permiten **WebSockets** y procesos de background.
- Dockerfile + `fly.toml`/`railway.json`.
- Base de datos gestionada; agregar `?sslmode=require` si corresponde.

**Post‑deploy checklist**
- [ ] Migraciones OK en logs.
- [ ] `/health` y `/version` responden.
- [ ] Webhook de pagos accesible desde proveedor.
- [ ] Impresión térmica probada en sitio.

---

## 12) Operación diaria (Runbook)
- **Alta manual** desde admin si entra pedido por teléfono/WA → canal `ADMIN/WHATSAPP`.
- **Priorizar**: marcar pedidos urgentes; filtro por estado.
- **Reintentos** de notificación fallida desde detalle del pedido.
- **Exportar CSV** fin de día → conciliación (pagos vs. órdenes).
- **Rotación**: limpiar `AuditLog` viejo (política reten. 90d).

---

## 13) Roadmap (futuro cercano)
- Inventario real con `InventoryMovement`.
- Múltiples locales / sucursales.
- Cupones y combos.
- App operador con PWA + notificaciones push.
- i18n y multimoneda.

---

## 14) Pasos rápidos (TL;DR ejecutable)
```bash
# 1) Infra local
cp .env.example .env
docker compose up -d db
pnpm install

# 2) DB
npx prisma migrate dev --name init
pnpm run seed

# 3) Dev
pnpm run dev  # o: pnpm run dev:docker

# 4) Tests
pnpm test && pnpm e2e

# 5) Build/Deploy
pnpm build
# Configurar hosting (Vercel/Fly/Railway), setear env y:
npx prisma migrate deploy
```

---

## 15) Variables de entorno (mínimas)
```
DATABASE_URL=postgresql://user:pass@host:5432/orderflow
AUTH_SECRET=super-secreto
TZ=America/Argentina/Buenos_Aires
PUBLIC_BASE_URL=https://tu-dominio
PUBLIC_WA_PHONE=5493704000000
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=60
PAYMENT_PROVIDER=mercadopago # o stripe
PAYMENT_MP_TOKEN=
PAYMENT_STRIPE_KEY=
```

**Listo.** Seguí estos pasos y vas de 0 → producción sin dramas. Cuando quieras, lo convierto en un repo “cloná‑y‑andá” con todo prearmado.

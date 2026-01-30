# ⚡ PERFORMANCE - Historias de Rendimiento Operativo

**Módulo ID**: `performance`  
**Versión**: 1.0  
**Última actualización**: 2026-01-28  
**Propósito**: Alinear el backlog con presupuestos de performance para los flujos críticos de menú, cuenta y transacciones antes del próximo release.

---

## 📋 Resumen
El foco es la experiencia de entrada de los usuarios back-office: el menú principal (`MainMenuPage`), la vista de cuenta (`AccountViewPage`) y la lista de transacciones (`TransactionListPage`). Cada pantalla reusa componentes existentes (`MenuScreen`, `AccountViewScreen`, `TransactionListScreen` y `LoadingSpinner`) y depende de `useMenu`, `useAccountView` y `useTransactionList` para coordinar estado, validaciones y llamadas a la API. La historia documenta métricas concretas (FCP, TTI, bundle size y latencias API) y un plan de instrumentación para monitorearlas de forma repetible.

## 🔍 Flujos cubiertos

| Pantalla | Componentes clave | APIs en producción | Observación actual |
| --- | --- | --- | --- |
| `MainMenuPage` | `MenuScreen`, `useMenu`, `getMainMenuData` | `GET /api/menu/mainmenu` *(mock: `/api/menu/main`)* | Carga datos locales y controla navegación; `LoadingSpinner` se muestra durante validaciones de selección.
| `AccountViewPage` | `AccountViewScreen`, `useAccountView`, `AccountViewPage` | `GET /api/account/acccount?accountId={id}` *(mock: `/api/account-view` y `/api/account-view/initialize`)* | Búsqueda inicial y solicitudes de cuenta que muestran datos financieros con validaciones y enmascarado.
| `TransactionListPage` | `TransactionListScreen`, `useTransactionList`, `TransactionListPage` | `GET /api/transaction/transactionlist` *(mock: `/api/transactions/list` + pagination)* | Lista transacciones con shortcuts (F3/F7/F8), paginación server-side simulada y UI MUI para tablas y chips.

> **Nota:** los endpoints mockeados (`/api/menu/main`, `/api/account-view`, `/api/transactions/…`) reflejan los contratos reales que deben medirse en staging y producción.

## 🎯 Historia de Usuario (DS3AJG-2)
- **Como:** Senior Product Owner del SAI platform
- **Cuando:** planeo el próximo sprint que toca el menú, la vista de cuentas y las transacciones
- **Entonces:** quiero un onboarding claro con metas medibles de performance para que el equipo mejore el código sin dañar flujos críticos ni la satisfacción del usuario.

### Criterios de Aceptación
1. **AC1 - FCP en el entry point ≤ 1.5s (P95):** ejecutar mediciones sintéticas (Lighthouse CLI o WebPageTest) contra `MainMenuPage`, `AccountViewPage` y `TransactionListPage` para confirmar que los tres entry points cumplen First Contentful Paint ≤ 1.5 segundos en el P95.
2. **AC2 - Bundle principal gzipped < 500 KB:** tras `npm run build`, el chunk main (`dist/assets/index.*.js`) debe mantenerse comprimido por debajo de 500 KB; usar `gzip-size`, el reporte de Vite o `npx rollup-plugin-visualizer` para verificar y evidenciar el resultado.
3. **AC3 - GET APIs ≤ 500 ms (P95) + 99% success:** las llamadas `GET /api/menu/mainmenu`, `GET /api/account/acccount` y `GET /api/transaction/transactionlist` (o sus equivalentes) deben registrar el P95 ≤ 500 ms y tener ≥ 99% de respuestas exitosas mediante mediciones en staging y tests RPT/Playwright.

## 📈 Presupuestos de Performance
- **First Contentful Paint:** < 1.5 s (P95)
- **Time to Interactive:** < 3.0 s
- **Bundle Principal Gzipped:** < 500 KB
- **GET Requests:** ≤ 500 ms (P95) con 99% success
- **Herramientas sugeridas:** Lighthouse CLI, WebPageTest, `gzip-size`, `ApiClient` extendido, Playwright, `../performance/performance-reports.md`.

## 🔧 Plan de Instrumentación
1. **Harness sintético:** crear scripts de Lighthouse/WebPageTest que entren directamente a los tres entry points y registren FCP/TTI. Mantener los resultados históricos en `../performance/performance-reports.md`.
2. **Medición de bundle:** ejecutar `npm run build` y tomar el tamaño gzipped del chunk principal (puede usarse `npx gzip-size dist/assets/index.*.js`). Registrar la versión y fecha cada vez que se valide el presupuesto.
3. **Latencias API:** extender `app/services/api.ts` para instrumentar `fetch` (por ejemplo, con `performance.mark`/`measure`) y capturar métricas de los GET listados. Utilizar esos datos en un test Playwright o script Node que llame las URLs y mida el P95.
4. **Checklist de regresión:** documentar los pasos en `../performance/performance-checklist.md` (puede ser una sección adicional aquí) para confirmar los tres presupuestos antes de cerrar el sprint.
5. **Reporte ligero:** mantener `../performance/performance-reports.md` con una tabla summarizing FCP, TTI, bundle y latencias por ejecución (fecha, SHA, observaciones).

## 🧾 Formato del Reporte de Ejecuciones
Cada registro debe incluir:
- SHA / rama / fecha
- FCP (MainMenu / AccountView / TransactionList)
- TTI (P95 mayor o promedio)
- Bundle gzipped (main chunk)
- Latencias P95 de los 3 GETs y % success
- Observaciones (cold start, red lenta, errores)

## ⚙️ Reutilización y Componentes
- **Entry points:** `MainMenuPage`, `AccountViewPage`, `TransactionListPage` ya están implementados en `app/pages/`.
- **Hooks:** `useMenu`, `useAccountView`, `useTransactionList` encapsulan estado, errores y llamadas a la API.
- **Screens y UI:** `MenuScreen`, `AccountViewScreen`, `TransactionListScreen`, `SystemHeader` y `LoadingSpinner` proporcionan consistencia visual y centrado en la percepción de carga.
- **API Client:** `app/services/api.ts` maneja timeouts, headers y logs; se puede extender para medir latencias compartidas entre todas las llamadas.
- **Mocks:** `app/mocks/menuHandlers.ts`, `accountHandlers.ts` y `transactionListHandlers.ts` ya simulan delays (300-800 ms) útiles para entrenar los escenarios.

## 🧪 Validación y Tareas pendientes
- [ ] Crear harness de Lighthouse o Playwright para cubrir los tres entry points y registrar FCP/TTI.
- [ ] Ejecutar `npm run build` y verificar el tamaño gzipped del chunk principal antes de cada release.
- [ ] Instrumentar/monitorizar `ApiClient` para medir el P95 de `GET /api/menu/mainmenu`, `GET /api/account/acccount` y `GET /api/transaction/transactionlist`.
- [ ] Documentar el checklist en `../performance/performance-checklist.md` y asociarlo al pipeline de liberación.
- [ ] Publicar los resultados en `../performance/performance-reports.md` y compartirlos con el equipo.

## 🔗 Dependencias
- **AUTH:** las rutas están protegidas y `useMenu` usa `selectCurrentUser`/`logout`.
- **APIs del backend:** las mediciones deben correr tanto con los mocks como en staging QR donde los endpoints reales están desplegados.
- **Tooling:** `Vite`, `Lighthouse`, `gzip-size`, `ApiClient` extendido y scripts de Playwright/Node para medir latencias y bundle.

## 📍 Referencias
- [System Overview](../../system-overview.md#-presupuestos-de-performance)
- `docs/modules/account/account-overview.md` y `docs/site/modules/accounts/index.html` (AccountView)
- MSW handlers: `app/mocks/menuHandlers.ts`, `accountHandlers.ts`, `transactionListHandlers.ts`

# Checklist de Regresión de Performance

1. Ejecutar `npm run build` sobre la rama objetivo.
2. Medir el tamaño gzipped del chunk principal (`dist/assets/index.*.js`) con `npx gzip-size` o el reporte de Vite.
3. Correr Lighthouse CLI (o Playwright/Lighthouse) contra:
   - `/menu/main` (MainMenuPage)
   - `/accounts/view` (AccountViewPage)
   - `/transactions/list` (TransactionListPage)
4. Exportar los tiempos FCP/TTI y confirmar que el P95 ≤ 1.5 s / 3.0 s respectivamente.
5. Ejecutar scripts de API (Playwright, Node) que consuman:
   - `GET /api/menu/mainmenu`
   - `GET /api/account/acccount?accountId=11111111111`
   - `GET /api/transaction/transactionlist`
   - Registrar latencias y verificar P95 ≤ 500 ms y ≥ 99% success.
6. Actualizar `./performance-reports.md` con la corrida y compartir con el equipo de Producto y QA.
7. Confirmar que no hay regresiones de bundle (revisar `dist/assets` y que ningún chunk crítico supere los 500 KB gzipped).
8. Documentar cualquier ajuste de código o configuración (por ejemplo: carga diferida, splitting, caching) que ayude a cumplir los presupuestos.

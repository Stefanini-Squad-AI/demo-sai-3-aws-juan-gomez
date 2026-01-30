# 💳 ACCOUNT - Módulo de Cuentas

**Módulo ID**: ACCOUNT  
**Versión**: 1.1  
**Última actualización**: 2026-02-10  
**Propósito**: Consultar estados, balances y tarjetas de cada cuenta y permitir actualizaciones atómicas de datos sensibles para los equipos de Customer Service y Riesgos.

---

## 📋 Descripción General

El módulo ACCOUNT expone dos pantallas principales (`AccountViewScreen`, `AccountUpdateScreen`) que están protegidas por `ProtectedRoute` y se integran con el layout corporativo a través de `SystemHeader`. El viewer consulta la API `GET /account-view` y ofrece validaciones cliente/servidor, mientras que el updater carga los datos de `/accounts/{accountId}` para aplicar confirmaciones y detección de cambios antes de llamar a `PUT /accounts/{accountId}`.

### Responsabilidades principales
- Validar que el `accountId` de búsqueda tenga 11 dígitos y no sea todo ceros, mostrando errores inline y activando `LoadingSpinner` durante la llamada a `GET /account-view?accountId=...`.
- Mostrar tarjetas de balance, métricas, tarjetas asociadas y datos de cliente con opciones para ocultar/mostrar información sensible (`showSensitiveData`) y un panel de cuentas de prueba (`testAccounts`) habilitado en DEV.
- Activar el modo edición seguro en `AccountUpdateScreen` (con `editMode`, diálogo de confirmación y atajos de teclado F3/F5/F12) para refrescar, modificar y reenviar toda la estructura de cuenta y cliente.
- Detectar cambios locales mediante `useAccountUpdate` y bloquear la actualización si hay validaciones activas (`activeStatus`, `zipCode`, `creditLimit`, `ficoScore`).
- Responder errores del backend simulado (por ejemplo el endpoint `/accounts/99999999999` devuelve 500) y mostrar mensajes amigables con `Alert`.

---

## 🏗️ Arquitectura del Módulo

### Componentes clave
1. **AccountViewScreen.tsx** – Formulario central con `TextField`, `Button`, `Grid`, `Card` y `Chip`. Integra `LoadingSpinner`, `Collapse` para cuentas de prueba, `Switch`/botones `Visibility` y tarjetas con iconos de MUI (`AccountBalance`, `CreditCard`, `Security`).
2. **AccountUpdateScreen.tsx** – Pantalla de edición con `Dialog` de confirmación, `Switch` para activar edición, validaciones locales en `handleFieldChange`, y botones `Save`, `Reset` y `Edit` sobre `Card` con secciones de `Account`, `Customer` y `Contact Information`.
3. **AccountViewPage.tsx / AccountUpdatePage.tsx** – Páginas envueltas por `ProtectedRoute`, que inyectan `useAccountView` y `useAccountUpdate` como fuente de datos y efectos (`initializeScreen`, `searchAccount`, `updateAccount`).
4. **useAccountView.ts** – Hook que llama a `apiClient.get('/account-view?accountId=...')`, maneja estados `loading/error/data`, inicializa `GET /account-view/initialize` y limpia con `reset`.
5. **useAccountUpdate.ts** – Hook que gestiona la carga `GET /accounts/{id}`, la actualización `PUT /accounts/{id}`, la detección de cambios (`hasChanges`) y exposes helpers `updateLocalData`, `resetForm`, `clearData`.
6. **SystemHeader + LoadingSpinner** – Layout común de todas las pantallas que incluye transaction Id (`CAVW`), programa y títulos, y provee feedback visual consistente.
7. **ProtectedRoute + App.tsx** – Rutas `/accounts/view` y `/accounts/update` requieren autenticación; `useSecureSession` refresca tokens y `useAppSelector` define redirección a menús de `back-office/admin`.

### Flujo de datos
1. El usuario ingresa un `accountId` → `AccountViewScreen` invoca `searchAccount` de `useAccountView` → `apiClient` construye `GET /account-view?accountId=` dentro de `app/services/api.ts` (timeout 10s, headers JSON + token).
2. Para edición, `AccountUpdateScreen` solicita `/accounts/{accountId}` y escribe con `apiClient.put` cuando se confirma el cambio (con `hasChanges` y validaciones locales).
3. En desarrollo, MSW (`app/mocks/accountHandlers.ts`, `app/mocks/accountUpdateHandlers.ts`) intercepta las llamadas y responde con las estructuras definidas en `mockAccountData` y `mockAccountUpdateData`.

---

## 🔗 APIs documentadas

### `GET /account-view?accountId={11-digits}`
**Propósito:** Recuperar la vista completa de balances, cliente y tarjetas.  
**Request:** `GET /account-view?accountId=00000000011`  
**Response (ejemplo):**
```json
{
  "transactionId": "TRX-0001",
  "programName": "COACTVWC",
  "accountStatus": "Y",
  "currentBalance": 1250.75,
  "creditLimit": 5000.0,
  "availableCredit": 3749.25,
  "groupId": "PREMIUM",
  "customerId": 1000000001,
  "firstName": "JOHN",
  "lastName": "SMITH",
  "customerSsn": "123-45-6789",
  "ficoScore": 750,
  "cardNumber": "4111111111111111",
  "infoMessage": "Displaying details of given Account",
  "inputValid": true
}
```
**Performance esperada:** < 500 ms (MSW simula 500 ms antes de responder).

### `GET /account-view/initialize`
Inicializa la pantalla con datos de referencia (`currentDate`, `transactionId`, `programName`). Los hooks detectan si la respuesta ya viene envuelta en `{ success, data }` o es un DTO plano y ajustan el estado.

### `GET /accounts/{accountId}`
Carga los mismos valores que permite editar `AccountUpdateScreen`. El handler `app/mocks/accountUpdateHandlers.ts` aplica retraso de 800 ms y retorna `success: true` con `AccountUpdateData`.

### `PUT /accounts/{accountId}`
Actualiza cuenta + cliente. La implementación mock valida:
- `activeStatus` en `['Y','N']`
- `creditLimit >= 0`
- `zipCode` cumple `^\d{5}(-\d{4})?$`
- `ficoScore` entre 300 y 850

**Request simplificado:**
```json
{
  "accountId": 11111111111,
  "activeStatus": "Y",
  "creditLimit": 5200.00,
  "zipCode": "10452",
  "ficoScore": 760,
  "phoneNumber1": "555-999-0000",
  "addressLine1": "789 NEW STREET"
}
```
**Response de éxito:** `{ "success": true, "message": "Changes committed to database", "data": {...} }` (simula 1.2 s de latencia).  
**Response de error:** `PUT /accounts/99999999999` devuelve 500 con `Database connection timeout` para pruebas de resiliencia.

---

## 📊 Modelos de Datos

### `AccountViewResponse` (app/types/account.ts)
```ts
interface AccountViewResponse {
  currentDate: string;
  transactionId: string;
  accountStatus?: string;
  currentBalance?: number;
  creditLimit?: number;
  availableCredit?: number;
  groupId?: string;
  customerId?: number;
  customerSsn?: string;
  ficoScore?: number;
  firstName?: string;
  lastName?: string;
  addressLine1?: string;
  zipCode?: string;
  phoneNumber1?: string;
  cardNumber?: string; // se enmascara en la UI
  infoMessage?: string;
  inputValid: boolean;
}
```

### `AccountUpdateData` (app/types/accountUpdate.ts)
```ts
interface AccountUpdateData {
  accountId: number;
  activeStatus: string;          // Y/N
  creditLimit: number;
  currentBalance: number;
  zipCode: string;              // Validación de regex
  phoneNumber1: string;
  ssn: string;                  // Enmascarado al mostrarse
  ficoScore: number;            // 300-850
  addressLine1: string;
  governmentIssuedId: string;
  primaryCardIndicator: string; // Y/N
}
```

---

## 🔐 Reglas de negocio clave

- **RN-001**: `Account ID` debe ser numérico, exactamente 11 dígitos y no puede ser `00000000000` (valida `handleSubmit` y MSW).
- **RN-002**: `activeStatus` sólo admite `Y` o `N`; se impone en pantalla de edición y mocks.
- **RN-003**: `creditLimit` no puede ser negativo; la UI bloquea la actualización si la validación local detecta valores inválidos.
- **RN-004**: `zipCode` cumple `^\d{5}(-\d{4})?$`; el handler arroja error si no lo cumple y el formulario muestra `helperText`.
- **RN-005**: `FICO score` entre 300 y 850; `accountUpdateHandlers` rechaza actualizaciones fuera de rango.
- **RN-006**: Los datos sensibles (`customerSsn`, `cardNumber`) se muestran enmascarados por defecto; el toggle `Show Sensitive Data` controla si se revelan.
- **RN-007**: La cuenta `99999999999` simula un fallo 500 para asegurarse de que el UI maneja errores transaccionales.
- **RN-008**: `availableCredit` se calcula como `creditLimit - currentBalance` en la UI antes de mostrar la tarjeta de métricas.

---

## 🎯 Historias de Usuario y Criterios

1. **Visualización inmediata de una cuenta (Simple, 1-2 pts)**  
   - Como agente de Customer Service, quiero buscar una cuenta por `accountId` de 11 dígitos y ver balance, crédito disponible, estado y tarjetas asociadas en menos de 500 ms para responder consultas.  
   - Criterios: ID validado antes de llamar a la API, datos sensibles enmascarados y Snackbar con errores cuando `GET` devuelve `errorMessage`.

2. **Actualización de datos personales (Medio, 3-5 pts)**  
   - Como administrador, quiero editar dirección, teléfono y ZIP desde `AccountUpdateScreen`, comprobar cambios (`hasChanges`) y confirmar con el diálogo antes de llamar a `PUT /accounts/{id}`.  
   - Criterios: validaciones locales (`zipCode`, porcentajes), confirm dialog `Save changes?`, `AccountUpdateScreen` bloquea si `validationErrors` no está vacío.

3. **Control de límites y riesgo (Complejo, 5-8 pts)**  
   - Como analista de riesgos, quiero que al subir el límite de crédito se recalcule `availableCredit`, se valide el `ficoScore` y se rechace la actualización si el backend responde errores (por ejemplo, la cuenta `99999999999`).  
   - Criterios: `AccountUpdateScreen` muestra alertas con mensajes de `errors` del handler, detecta `hasChanges` vs `originalData` y no refresca si el put falla 500.

---

## ⚡ Factores de aceleración de desarrollo

- **Hooks reutilizables**: `useAccountView` y `useAccountUpdate` encapsulan validaciones, estados `loading/error`, llamadas a `apiClient` y refresco local, reduciendo el código repetitivo.
- **Componentes MUI ya estilizados**: `Card`, `Grid`, `Chip`, `Button`, `Dialog` y `Alert` siguen el theme de `theme.ts`, acelerando cambios UX.
- **Gestión de sesiones y rutas**: `ProtectedRoute` + `useSecureSession` aseguran que solo usuarios autenticados vean `/accounts/view` y `/accounts/update`.
- **Mock data completa**: `app/mocks/accountHandlers.ts` y `accountUpdateHandlers.ts` ofrecen 10 cuentas, errores controlados y delays de 500/800/1200 ms, lo que permite pruebas sin backend.
- **Script `validate-mocks.sh`**: Verifica que existan 10 cuentas, 10 tarjetas y 10 usuarios, lo que da confianza para historias nuevas.

---

## 🔗 Dependencias y relaciones

- **AUTH** (`ProtectedRoute`, `authSlice`, `login`) – Provee sesión y tokens para las rutas de cuentas.
- **Menu principal** (`app/data/menuData.ts`) – Expone enlaces a `Account View` y `Account Update` para perfiles `back-office`.  
- **MSW (app/mocks/…)** – Intercepta todos los endpoints de cuentas en desarrollo; `app/mocks/handlers.ts` los agrupa.
- **apiClient** (`app/services/api.ts`) – Maneja headers, estado `Authorization`, detección de respuestas MSW vs reales y timeout (10s).
- **UI global** – `SystemHeader`, `LoadingSpinner`, `Alert`, `Collapse`, `Dialog` garantizan coherencia con otros módulos (Credit Card, Transaction).

---

## 🧪 Pruebas y Mocking

- `app/mocks/accountHandlers.ts` mantiene `mockAccountData` con 10 cuentas (11111111111, 22222222222, …) y valida `accountId` antes de retornar (errores 400/404).  
- `app/mocks/accountUpdateHandlers.ts` simula las rutas `GET /api/accounts/:accountId` y `PUT /api/accounts/:accountId`, inyecta delays (800 ms + 1.2 s), y enumera validaciones concretas (`activeStatus`, `zipCode`, `ficoScore`).  
- Los test accounts listados en `AccountViewScreen` se muestran solo si `import.meta.env.DEV` es `true`, facilitando la demostración sin conocer IDs reales.  
- El script `validate-mocks.sh` confirma que hay ≥10 cuentas, tarjetas y usuarios, y que `VITE_USE_MOCKS=true`, por lo que las historias que dependen de datos simulados están cubiertas.

---

## ⚙️ Presupuestos de performance

- **Carga inicial (Account View)**: Objetivo < 500 ms en ambiente local (MSW usa delay de 500 ms).  
- **Actualización transaccional**: `PUT /accounts/:accountId` demora ~1.2 s por diseño, por lo que las historias deben aceptar feedback asincrónico y mostrar loaders.
- **Timeout API**: `apiClient` aborta peticiones tras 10 s si no hay respuesta.
- **Budget de UI**: `LoadingSpinner` y `Snackbar` deben aparecer para cada request; no se permite que el formulario se quede activo durante la llamada.

---

## 🚨 Riesgos y readiness

1. **Dependencia de MSW** – En entornos sin `VITE_USE_MOCKS`, los endpoints `/account-view` y `/accounts` deben existir; se recomienda validar con backend real antes de la regresión.  
2. **Ausencia de auditoría** – Los formularios no registran cambio histórico (no hay `audit trail`), por lo que los equipos legales deben considerar complementar la historia con un módulo de log.  
3. **Validaciones duales** – La lógica local (hooks + UI) debe seguir la misma regex que MSW; cualquier cambio en `accountUpdateHandlers` exige actualizar `AccountUpdateScreen` para evitar false positives.

---

## 📋 Lista de tareas y métricas

**Completadas**
- [x] DOC-ACC-001: Documentar flujo de visualización de cuentas con hooks reales.  
- [x] DOC-ACC-002: Mapear APIs (`/account-view`, `/accounts/{id}`) con ejemplos de MSW.  

**Pendientes**
- [ ] DOC-ACC-003: Incluir acuerdo de auditoría y event logging en futuras historias.  

**Obsoletas**
- [~] DOC-ACC-000: Documentar servicios COBOL antiguos (no aplica al frontend actual).  

**Métricas de éxito**
- `validate-mocks.sh` pasa 13 pruebas (10 cuentas, 10 tarjetas, MSW config) → confirma que los datos de referencia están siempre disponibles.  
- Al menos un 100% de las historias nuevas mencionan este documento antes de asignar tareas (registro en tickets de equipo).  

---

## 📑 Anexos

- **Guía visual**: `docs/site/modules/account/index.html` tiene plantillas específicas de User Stories, complejidades y componentes.  
- **Vista global**: `docs/system-overview.md` concentra el catálogo completo (incluye secciones CREDIT CARD, TRANSACTION, etc.).  
- **Datos de prueba**: `MOCK_DATA_SUMMARY.md` detalla los 10 accounts y sus relaciones con tarjetas/usuarios.  
- **Validación**: `validate-mocks.sh` garantiza la cobertura mínima de entidades.  

**Última actualización**: 2026-02-10  
**Mantenido por**: Equipo DS3A  
**Precisión**: ≥ 95% (basado en el código fuente real del frontend)  

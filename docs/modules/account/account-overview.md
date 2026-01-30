# 💳 ACCOUNT - Módulo de Cuentas

**Módulo ID**: ACCOUNT  
**Versión**: 1.0  
**Última actualización**: 2026-01-28  
**Propósito**: Consulta y actualización segura de información de cuentas y clientes para los equipos de servicio al cliente y operaciones.

---

## 📋 Descripción General

El módulo ACCOUNT centraliza las operaciones de consulta y actualización de cuentas de tarjetas de crédito. Permite a los usuarios autorizados visualizar estados financieros, límites y relaciones con clientes y tarjetas, y mantener datos personales y financieros alineados con las reglas regulatorias.

### Responsabilidades principales

- Buscar cuentas por *Account ID* de 11 dígitos y presentar un resumen financiero completo.
- Mostrar métricas clave (balance, crédito disponible, historial de tarjetas asociadas).
- Actualizar datos del cliente y la cuenta de forma atómica, con validaciones de negocio previas.
- Proteger datos sensibles (SSN, números de tarjeta) mediante enmascarado y restricciones de visualización.
- Validar tipos de cuenta y estados antes de permitir escrituras.

---

## 🏗️ Arquitectura del Módulo

### Componentes clave

1. **AccountViewScreen.tsx** – Vista principal para búsquedas. Usa Material-UI para tarjetas informativas, tablas y acciones rápidas, y reusa funciones de enmascarado (`maskSSN`, `maskCard`) compartidas con otros módulos financieros.
2. **AccountUpdateScreen.tsx** – Formulario detallado en modo edición activado por toggle. Incluye validación inline (accountId 11 dígitos, ZIP code, SSN) y resumen de cambios antes de enviar.
3. **AccountViewPage.tsx / AccountUpdatePage.tsx** – Envoltorios de página que integran layout global, rutas protegidas y los hooks de carga/actualización.
4. **useAccountView.ts** – Hook React que gestiona `loading`, `error`, `data` y expone `searchAccount` con validaciones previas (parseo numérico, no cero). Representa el patrón de hooks reutilizables para módulos similares.
5. **useAccountUpdate.ts** – Hook que compara estado original vs. editado, detecta campos modificados y coordina la llamada al servicio `PUT /api/account/update`.
6. **AccountViewService.java** – Servicio backend que realiza joins entre `CardXrefRecord`, `Account` y `Customer`, aplicando enmascarado y preparando DTOs para el frontend.
7. **AccountUpdateService.java** – Servicio transaccional que bloquea filas (`SELECT FOR UPDATE`), valida reglas de negocio (FICO, ZIP, accountId), actualiza `Account` + `Customer` y devuelve un mapa con mensajes.
8. **AccountValidationService.java** – Validaciones centralizadas (`isValidAccountId`, `validateYesNo`, `validateSSN`, `validateZipCode`) que liberan a frontend y backend de duplicar lógica.

### Patrón técnico

- **UI Library**: Material-UI 5.x (TextField, Card, Button, Snackbar).
- **State**: Redux Toolkit (opcional para este módulo) + hooks locales para estados de formularios.
- **Routing**: React Router con `ProtectedRoute` para asegurar roles.
- **Servicios de API**: `app/services/accountApi.ts` (o equivalente) encapsula llamadas REST.
- **Backend**: Spring Boot + JPA, transacciones `@Transactional` y servicios REST estándar.

---

## 🔗 APIs Documentadas

- **GET /api/account/acccount?accountId={id}** – Consulta completa de cuenta por ID de 11 dígitos. Retorna saldo, límite, cliente, tarjetas y datos en formato DTO con SSN enmascarado. Responde en < 500ms (P95).
- **PUT /api/account/update** – Actualiza account + customer. Recibe `AccountUpdateRequest` (accountId, customer info, address, contacto) y retorna `{ success: true, message: "Account updated successfully" }`. Ejecuta validaciones con `AccountValidationService` y rollback automático.

---

## 📊 Modelos de Datos (simplificado)

### Frontend / TypeScript

```ts
interface Account {
  accountId: string;           // 11 dígitos (RN-001)
  status: 'Y' | 'N';
  balance: number;
  creditLimit: number;
  availableCredit: number;
  groupId: string;
  customer: Customer;
  cards: CreditCard[];
}

interface Customer {
  customerId: string;
  firstName: string;
  middleName?: string;
  lastName: string;
  ssn: string;                // Enmascarado al mostrar (***-**-1234)
  ficoScore: number;          // 300-850
  address: Address;
  phones: Phone[];
  governmentId: string;
}

interface AccountUpdateRequest {
  accountId: string;
  customer: Partial<Customer>;
  notes?: string;
}
```

### Backend / Java (JPA)

```java
@Entity
public class Account {
  @Id
  private Long accountId;        // 11 dígitos
  private String activeStatus;   // Y/N
  private BigDecimal balance;
  private BigDecimal creditLimit;
  private String groupId;
  @OneToOne
  private Customer customer;
}

@Entity
public class Customer {
  @Id
  private Long customerId;
  private String firstName;
  private String lastName;
  private String socialSecurityNumber;
  private Integer ficoScore;
  private String zipCode;
}
```

---

## 🔐 Reglas de Negocio

- **RN-001**: *Account ID* debe tener exactamente 11 dígitos y no puede ser todo ceros.
- **RN-005**: Solo cuentas con `status = 'Y'` permiten transacciones o modificaciones.
- **RN-006**: SSN y números de tarjeta deben mostrarse enmascarados (`***-**-XXXX` y `****-****-****-XXXX`).
- **RN-009**: `status` solo acepta `Y` o `N`.
- **RN-012**: FICO Score entre 300 y 850.
- **RN-015**: ZIP Code cumple `^\d{5}(-\d{4})?$`.
- **RN-018**: Actualización de Account + Customer se ejecuta dentro de esta transacción; si falla una validación, se hace rollback.
- **RN-021**: Antes de actualizar, se aplica `SELECT FOR UPDATE` para evitar race conditions.
- **RN-024**: Crédito disponible = `creditLimit - balance`, debe recalcularse después de cada actualización.
- **RN-030**: Cada cuenta tiene al menos un cliente asociado, y cada cliente tiene al menos una tarjeta activa.

---

## 🎯 User Stories (ejemplos)

1. **Consulta de cuenta**  
   Como representante de servicio al cliente, quiero buscar una cuenta por su ID de 11 dígitos para ver saldos, límite de crédito y tarjetas asociadas, con datos sensibles enmascarados, y responder consultas en < 500ms.  
   - Criterios: ID validado, SSN/ tarjetas enmascarados, respuesta del endpoint sincronizada y errores amables.
   - Complejidad: Simple (1-2 pts).

2. **Actualización de datos personales**  
   Como administrador de cuentas, quiero actualizar la dirección y teléfono principal de un cliente para reflejar su nueva residencia sin desbloquear tarjetas ni errores de validación.  
   - Criterios: Validaciones `ZipCodeRegex`, no modificar `accountId`, persistir en una sola transacción.  
   - Complejidad: Medio (3-5 pts).

3. **Asegurar integridad de límites**  
   Como analista de riesgo, quiero que al actualizar un nuevo límite de crédito se recalculen el balance disponible y se validen reglas de FICO antes de aceptar el cambio.  
   - Criterios: FICO 300-850, `availableCredit` = `creditLimit - balance`, rollback si el nuevo límite es menor al balance actual.  
   - Complejidad: Complejo (5-8 pts).

---

## ⚡ Factores de Aceleración de Desarrollo

- **Hooks reutilizables (`useAccountView`, `useAccountUpdate`)**: encapsulan carga y actualización, eliminando la necesidad de reimplementar estados `loading/error` para nuevas historias.
- **Servicios backend compartidos**: `AccountValidationService` y `AccountUpdateService` ya aplican reglas críticas, así que las historias solo validan escenarios específicos sin reescribir lógica central.
- **Material-UI Layouts y componentes**: Cards, Tables y Inputs ya estilizados, reducen ~60% del esfuerzo UI.
- **Reglas de negocio documentadas**: Validaciones incluidas en `docs/system-overview` aseguran alineación sin re-trabajo.
- **API contract**: `GET /api/account/acccount` y `PUT /api/account/update` están disponibles, permitiendo comenzar desarrollo solo con mocks y refinar con backend real.

---

## 🔗 Dependencias

- **Auth (AUTH)**: Necesita sesión válida y roles. `ProtectedRoute` restringe acceso a usuarios `admin` o `back-office`.
- **Menu**: El menú principal expone el módulo ACCOUNT como opción condicional.
- **Credit Card**: Reutiliza enmascarado y algunas vistas de tarjetas asociadas (`cards` array).
- **Transaction**: Consultas de balance se reflejan en reglas de negocio de transacciones.
- **UI (Material-UI)**: Todos los formularios usan componentes estándar (TextField, RadioGroup, Snackbar).

---

## 🧪 Pruebas y Mocking

- **MSW Handlers**: `GET /api/account/acccount` y `PUT /api/account/update` ya mockeados en `app/mocks/accountHandlers.ts` con cuentas de prueba (accountId 11111111111, 22222222222).  
- **Datos de prueba**: cada mock incluye cliente, tarjetas e historial de transacciones, lo que facilita validar historias sin backend.

---

## 📑 Anexos

- **Ver detalles en el sitio**: `docs/site/modules/accounts/index.html` incluye patrones de US, complejidad y referencias técnicas.  
- **Referencia de APIs ampliada**: `docs/system-overview.md` (sección Cuentas) mantiene request/response de los endpoints referenciados en este módulo.

---

**Última actualización**: 2026-01-28  
**Mantenido por**: Equipo DS3A  
**Precisión**: ≥ 95% (alineado con código actual)

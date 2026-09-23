> Este archivo documenta los prompts principales utilizados con Claude Code (Anthropic) durante el desarrollo de ParkingHub. El proyecto usa el workflow **OpenSpec spec-driven**: cada funcionalidad pasa por `/opsx:propose` (proposal + design + tasks) antes de `/opsx:apply` (implementación). Los prompts aquí recogidos son representativos de cada fase del ciclo de vida del desarrollo.

## Índice

1. [Descripción general del producto](#1-descripción-general-del-producto)
2. [Arquitectura del sistema](#2-arquitectura-del-sistema)
3. [Modelo de datos](#3-modelo-de-datos)
4. [Especificación de la API](#4-especificación-de-la-api)
5. [Historias de usuario](#5-historias-de-usuario)
6. [Tickets de trabajo](#6-tickets-de-trabajo)
7. [Pull requests](#7-pull-requests)

---

## 1. Descripción general del producto

**Prompt 1** — Definición inicial del producto y scope del MVP:

```
Quiero construir una plataforma SaaS para gestionar comunidades de propietarios de plazas de parking.
El problema es que hoy todo se gestiona con hojas de cálculo y WhatsApp: contratos en papel, cobros
sin trazabilidad, incidencias perdidas. Necesito digitalizar todo esto.

Usuarios principales:
- El administrador de la finca (ADMIN): gestiona propietarios, contratos, cobros y turnos del personal.
- El gestor de garita (GESTOR): abre y cierra turno, registra cobros en efectivo o Bizum y gestiona
  las plazas que le han delegado los propietarios.
- El propietario de plaza (PROPIETARIO): consulta contratos, pagos y documentación comunitaria.
- El inquilino (INQUILINO): paga y descarga sus recibos.

El negocio es multi-tenant: cada parking es un cliente independiente. Identifícalo por subdominio.
Stack: .NET 9 + EF Core + PostgreSQL en el backend, Angular 17 en el frontend.
Ayúdame a diseñar el plan de desarrollo fase a fase.
```

**Prompt 2** — Definición del módulo de catálogo público y flujo de captación:

```
/opsx:propose public-catalog

Necesito un portal público (sin login) donde los interesados puedan ver las plazas disponibles
en alquiler y en venta. Cada plaza tiene: foto, planta, precio mensual, dimensiones, estado.
El visitante puede enviar una solicitud de contacto que llega al ADMIN por email.
El portal debe ser indexable por buscadores (SEO). Angular, standalone components.
```

**Prompt 3** — Incorporación del rol VIGILANTE al sistema:

```
Analiza lo siguiente: creo que sería necesario crear un nuevo rol, parecido al de Gestor pero
sin acceso a datos económicos. Sería el vigilante de seguridad del parking. Sus funciones serían:
abrir y cerrar turnos de vigilancia, registrar incidencias de seguridad con el campo "zona"
(para indicar dónde ocurrió), y consultar las visitas a garita del día como referencia.
No debe tener acceso a cobros, cuadre de caja ni datos financieros de propietarios.
¿Cómo lo enfocarías?
```

---

## 2. Arquitectura del Sistema

### **2.1. Diagrama de arquitectura:**

**Prompt 1** — Decisión de monolito modular vs microservicios:

```
Para el MVP de ParkingHub necesito decidir la arquitectura. Tenemos multi-tenancy, 5 roles distintos
con portales Angular separados, y una API .NET 9 que los sirve a todos. ¿Monolito modular o
microservicios? Considera que somos un equipo pequeño y el hosting inicial es un VPS Hetzner
con Coolify. Quiero tu recomendación razonada y el diagrama de componentes resultante.
```

**Prompt 2** — Diseño del middleware de multi-tenancy:

```
/opsx:propose parking-multitenancy

El tenant se identifica por subdominio en el header Host de cada request
(ej: barcelona.parkinghub.com). Necesito un middleware .NET que resuelva el parking_id
desde el subdominio y lo inyecte en un CurrentParkingContext para que todos los servicios
lo consuman sin conocer el detalle del request. El superadmin no tiene subdominio propio.
Diseña la entidad Parking, el middleware y cómo se aplica el filtro global en EF Core.
```

**Prompt 3** — Estrategia de módulos premium (feature flags por tenant):

```
Quiero que algunos módulos sean opcionales y de pago: FLEX (parking por sesión para anónimos),
BARRERAS (integración con hardware) y REPORTING_AVANZADO. Solo el SUPER_ADMIN puede activarlos
por parking. El backend debe tener un guard que rechace el endpoint si el módulo no está activo
para ese tenant. Angular necesita un FeatureGuard que oculte las rutas no disponibles.
Diseña el modelo de datos y los guards para ambas capas.
```

---

### **2.2. Descripción de componentes principales:**

**Prompt 1** — Estructura de capas del backend:

```
/opsx:propose parking-foundation

Genera la solución .NET 9 completa para ParkingHub con arquitectura en capas:
- ParkingHub.Api: controllers, program.cs, middleware
- ParkingHub.Application: servicios, DTOs, interfaces
- ParkingHub.Domain: entidades, value objects (sin dependencias externas)
- ParkingHub.Infrastructure: EF Core, migraciones, SendGrid, QuestPDF
- ParkingHub.SharedKernel: AuditableEntity, interfaces base

Configura ASP.NET Identity para autenticación, JWT Bearer con refresh tokens rotativos,
Swagger con autenticación, CORS para localhost:4200 en desarrollo y el dominio de producción.
```

**Prompt 2** — Diseño de la capa de aplicación para el módulo de pagos:

```
/opsx:propose payment-control

Diseña el servicio de pagos para ParkingHub. Cada mensualidad es un PagoAlquiler ligado
a un ContratoAlquiler. Hay dos métodos: efectivo y Bizum manual (sin pasarela). Cada método
tiene su propia máquina de estados:
- Efectivo: PENDIENTE → EN_GARITA → ENTREGADO_PROPIETARIO
- Bizum: PENDIENTE → NOTIFICADO_POR_INQUILINO → CONFIRMADO

El gestor confirma los pagos en efectivo desde su portal. El propietario puede descargar
un recibo PDF (QuestPDF). Los endpoints deben filtrar por rol: el propietario solo ve sus plazas,
el gestor ve todas las del parking, el inquilino solo sus propios pagos.
```

**Prompt 3** — Shell Angular multi-portal con roleGuard:

```
Necesito el routing raíz de Angular organizado en portales por rol, todos lazy-loaded:
/public → sin guard (catálogo público)
/admin  → roleGuard(['ADMIN'])
/gestor → roleGuard(['GESTOR'])
/vigilante → roleGuard(['VIGILANTE'])
/propietario → roleGuard(['PROPIETARIO'])
/inquilino → roleGuard(['INQUILINO'])
/superadmin → roleGuard(['SUPER_ADMIN'])

El roleGuard debe leer el rol del JWT, redirigir al portal correcto si el rol no coincide,
y redirigir al login si no hay token. Usa Angular signals y standalone components.
```

---

### **2.3. Descripción de alto nivel del proyecto y estructura de ficheros**

**Prompt 1** — Estructura del repositorio desde el inicio:

```
Organiza el repositorio de ParkingHub con esta estructura:
- /src/  → solución .NET con sus 5 proyectos
- /plazahub-web/ → app Angular (nombre de marca PlazaHub para el frontend)
- /openspec/ → specs, changes activos y archivo de changes completados
- /scripts/ → dev-up.sh y dev-down.sh para levantar el stack local completo
- /docker-compose.yml → PostgreSQL 16 en desarrollo

El script dev-up.sh debe: levantar PostgreSQL en Docker, esperar a que esté healthy,
aplicar migraciones EF Core, arrancar la API en background (puerto 5001),
instalar dependencias npm si hace falta, arrancar ng serve en background (puerto 4200),
y hacer smoke tests con curl para verificar que todo responde.
```

**Prompt 2** — Organización de módulos Angular por portal:

```
Dentro de plazahub-web/src/app/, crea una carpeta por portal:
admin/, gestor/, vigilante/, propietario/, inquilino/, superadmin/, public/, shared/
Cada portal tiene su propio shell component (sidenav + topbar), su routes file y sus
componentes de cada sección como standalone components.
El directorio shared/ contiene: guards, interceptors (JWT, error), design tokens CSS,
y servicios compartidos como AuthService y CurrentUserService.
```

---

### **2.4. Infraestructura y despliegue**

**Prompt 1** — Configuración de Coolify en Hetzner para el despliegue:

```
El hosting MVP es un VPS de Hetzner con Coolify como PaaS self-hosted.
Necesito el pipeline de despliegue: push a main en GitHub → webhook → Coolify construye
la imagen Docker de la API .NET → despliega el contenedor → sirve la SPA Angular como
archivos estáticos con Nginx → SSL automático con Let's Encrypt por subdominio.
¿Cómo configuro el Dockerfile de la API y el nginx.conf para el frontend?
```

**Prompt 2** — DatabaseSeeder idempotente para producción:

```
El DatabaseSeeder de ParkingHub debe ser idempotente: si ya existe el superadmin o el parking
de demo, no falla ni duplica datos. Debe crear:
1. El rol SUPER_ADMIN y el usuario superadmin@parkinghub.com con contraseña de env var.
2. Un parking de demo (slug: "demo") con su admin y gestor de prueba.
3. Un parking de ejemplo completo con propietario, inquilino activo y un contrato vigente.
Configúralo para que se ejecute automáticamente al arrancar la API si la BBDD está vacía.
```

---

### **2.5. Seguridad**

**Prompt 1** — Sistema de invitaciones por token (sin registro abierto):

```
/opsx:propose auth-identity-rbac

No quiero registro abierto en ParkingHub. Los usuarios solo pueden entrar por invitación.
El flujo es: el ADMIN genera una InvitacionRegistro con un token de un solo uso y TTL de 7 días,
le llega un email al invitado con el link, el invitado rellena nombre y contraseña,
al registrarse se consume el token y se le asigna el rol correcto para ese parking.
Implementa esto con ASP.NET Identity, JWT con refresh token rotativo, y SendGrid para el email.
```

**Prompt 2** — Guard de aislamiento multi-tenant en EF Core:

```
Necesito que sea imposible que un usuario de un parking vea datos de otro tenant,
aunque conozca los IDs. La solución debe ser estructural, no por validación en cada endpoint.
¿Cómo implemento un filtro global en EF Core que añada automáticamente
WHERE parking_id = @current en todas las queries de entidades que implementen ITenantEntity?
El CurrentParkingContext se inyecta por request desde el middleware de subdominio.
```

**Prompt 3** — Validación de entrada y middleware global de errores:

```
Necesito un middleware global en .NET que intercepte todas las excepciones y devuelva
siempre la misma estructura JSON: { "data": null, "error": "mensaje descriptivo" }.
Los errores de validación de DTOs (ModelState inválido) deben devolver 400 con el detalle
de qué campos fallaron. Los errores de negocio (ej: plaza ya alquilada) deben devolver 409.
Los errores no controlados deben devolver 500 sin exponer el stack trace en producción.
```

---

### **2.6. Tests**

**Prompt 1** — Configuración de WebApplicationFactory para tests de integración:

```
/opsx:apply payment-control

En la tarea de tests de integración: usa WebApplicationFactory<Program> con una base de datos
PostgreSQL de test dedicada (plazahub_test) que se crea y migra automáticamente antes de
los tests y se limpia entre test runs. No uses mocks para la base de datos — necesitamos
que los tests validen el comportamiento real de EF Core y las constraints de PostgreSQL.
Crea un fixture base ParkingHubIntegrationTestBase con helpers para crear tenants, usuarios
y datos de seed para los tests.
```

**Prompt 2** — Tests del flujo de pagos en efectivo y Bizum:

```
Escribe los tests de integración para el módulo de pagos. Necesito cubrir:
1. Efectivo: crear pago → GESTOR lo marca EN_GARITA → GESTOR lo marca ENTREGADO_PROPIETARIO.
2. Bizum: crear pago → INQUILINO notifica → ADMIN/GESTOR confirma.
3. Que el PROPIETARIO NO puede confirmar pagos (403).
4. Que el INQUILINO NO puede ver pagos de otros contratos (solo los suyos).
5. Que el PDF de recibo solo se puede descargar cuando el estado es el correcto.
El header Content-Disposition debe incluir el nombre de fichero con extensión .pdf.
```

---

## 3. Modelo de Datos

**Prompt 1** — Diseño de las entidades de dominio para propietarios y plazas:

```
/opsx:propose owner-profiles-spots

Entidades que necesito:
- PropietarioProfile: vinculado a ApplicationUser (1:1), puede tener N plazas.
- PlazaParking: pertenece a un Parking (tenant). Estados: DISPONIBLE, EN_VENTA, ALQUILADA,
  USO_PROPIO, OCUPADA_FLEX, MANTENIMIENTO. El flag flex_habilitada es independiente del estado.
  Cuando está EN_VENTA existe un AnuncioVenta asociado (relación 1:1 con cascade delete).
- La delegación al gestor es por plaza individual (campo bool delegadaAGestor en PlazaParking).
- Las fotos de la plaza se almacenan como rutas en FotoPlaza (relación 1:N).
Diseña las entidades C# y la migración EF Core correspondiente.
```

**Prompt 2** — Modelo de contratos y pagos con máquinas de estado:

```
/opsx:propose rental-contracts

ContratoAlquiler vincula una PlazaParking con un inquilino (ApplicationUser).
Tiene: fechaInicio, fechaFin nullable (contrato indefinido), precioMensual, estado
(ACTIVO, FINALIZADO, CANCELADO), metodoPagoConfigurado (EFECTIVO, BIZUM).

PagoAlquiler es un registro por mensualidad: periodo (formato "2026-09"), importe,
metodo, estado (distinto por método), fechaPago nullable, gestorId nullable (quién confirmó).

Las transiciones de estado deben validarse: no se puede pasar de ENTREGADO_PROPIETARIO
a EN_GARITA, por ejemplo. ¿Implementas la validación en el dominio o en el servicio de aplicación?
Razona tu decisión.
```

**Prompt 3** — Entidades del módulo de finanzas comunitarias:

```
/opsx:propose community-finances

Necesito gestionar cuotas y derramas de la comunidad:
- ConceptoComunidad: define una cuota periódica o derrama puntual. Tiene importe, periodicidad
  (MENSUAL, TRIMESTRAL, ANUAL, PUNTUAL) y fechaVencimiento para derramas.
- PagoComunidad: un registro por propietario y periodo. Estados: PENDIENTE, PAGADO, EXENTO.
- La lista de morosos son los propietarios con PagoComunidad en estado PENDIENTE vencidos.
- El fondo económico es la suma de todos los PagoComunidad PAGADOS menos los GastoComunidad.
- GastoComunidad: gasto de la comunidad categorizado. Solo editable en el mes en curso
  (integridad financiera: no se puede editar un gasto de meses anteriores).
```

---

## 4. Especificación de la API

**Prompt 1** — Diseño de los endpoints de turnos del gestor:

```
/opsx:propose gestor-shifts

Necesito los endpoints para el turno del gestor:
- POST /api/v1/turnos → Abrir turno. Solo puede haber uno ACTIVO por parking a la vez.
  Si el rol es VIGILANTE, el tipoTurno se asigna como VIGILANCIA (sin datos financieros).
- GET /api/v1/turnos/activo → Estado del turno en curso (para el timer en el portal).
- PATCH /api/v1/turnos/{id}/cierre → Cerrar turno con: efectivoCobrado,
  efectivoEntregadoPropietarios. El sistema calcula efectivoEnCajaCierre = cobrado - entregado.
- GET /api/v1/turnos → Historial. ADMIN ve todos, GESTOR/VIGILANTE solo los suyos.
Los turnos VIGILANCIA deben excluirse de los cálculos financieros en el dashboard de reporting.
```

**Prompt 2** — Endpoints de incidencias con soporte multi-rol y campo zona:

```
El endpoint POST /api/v1/incidencias debe aceptar el campo zona (string nullable, max 200 chars)
pero solo cuando categoria = 'SEGURIDAD'. Si se envía zona con otra categoría, ignorarlo.
Si categoria = SEGURIDAD y urgencia = ALTA o URGENTE, enviar email de alerta a todos los
gestores del parking con asunto "[SEGURIDAD] Nueva incidencia [ALTA]: {titulo}".
Los roles que pueden crear incidencias: ADMIN, GESTOR, VIGILANTE, PROPIETARIO, INQUILINO.
Los roles que pueden ver TODAS las incidencias del parking: ADMIN, GESTOR, VIGILANTE.
PROPIETARIO e INQUILINO solo ven las suyas propias.
```

**Prompt 3** — Versionado y estructura de respuesta estandarizada:

```
Quiero que todos los endpoints de ParkingHub devuelvan siempre la misma estructura JSON:
{ "data": <payload>, "error": null } en éxito, o { "data": null, "error": "mensaje" } en error.
El versionado es por prefijo de ruta: /api/v1/. Configura un ApiResponse<T> genérico en C#
y un extension method OkData(payload) en ControllerBase para no repetir el wrapper en cada acción.
Los errores 400 de validación ModelState también deben usar esta estructura,
con el error siendo la concatenación de los mensajes de validación.
```

---

## 5. Historias de Usuario

**Prompt 1** — Historias del flujo completo de alquiler:

```
/opsx:propose rental-request-flow

Define el flujo completo desde que un interesado solicita alquiler hasta que el inquilino
tiene acceso a su portal:
1. El interesado rellena el formulario del catálogo público → llega SolicitudAlquiler al ADMIN.
2. El ADMIN aprueba (o rechaza) la solicitud → si aprueba, genera una InvitacionRegistro.
3. El interesado recibe email con link para registrarse → crea su cuenta con rol INQUILINO.
4. El ADMIN crea el ContratoAlquiler → la plaza pasa a estado ALQUILADA.
5. El inquilino accede a su portal y ve su contrato activo.
Define los estados de SolicitudAlquiler, las validaciones de negocio y los emails en cada paso.
```

**Prompt 2** — Historias del gestor para el cuadre de caja diario:

```
Como GESTOR quiero poder:
1. Ver al entrar al portal si tengo un turno activo o si debo abrir uno.
2. Ver durante el turno una lista de los pagos pendientes de cobrar de las plazas que gestiono.
3. Al cobrar un pago en efectivo, marcarlo como EN_GARITA en el momento y registrar al inquilino
   que pagó para tener trazabilidad.
4. Al final del turno, introducir el efectivo total cobrado y el entregado a propietarios,
   y que el sistema calcule automáticamente lo que queda en caja.
5. Ver el historial de mis turnos anteriores con el resumen económico de cada uno.
¿Cómo modelamos el flujo de turno para cubrir todos estos casos de uso?
```

**Prompt 3** — Historia de usuario del propietario delegando gestión:

```
Como PROPIETARIO con varias plazas quiero poder:
- Delegar la gestión de una o varias de mis plazas al gestor de garita de forma individual.
  (No quiero delegar todas si solo el gestor gestiona una de ellas.)
- Ver en tiempo real cuánto se ha cobrado este mes en mis plazas delegadas.
- Recibir notificación por email cuando el gestor entregue el efectivo cobrado de mis plazas.
- Revocar la delegación cuando quiera, sin afectar al contrato de alquiler vigente.
Diseña los endpoints y la lógica de notificación para este flujo.
```

---

## 6. Tickets de Trabajo

**Prompt 1** — Generación del plan de implementación para el módulo de turnos:

```
/opsx:propose gestor-shifts

Desglosa en tasks.md el trabajo de implementar el módulo de turnos del gestor.
Las tasks deben ser atómicas (máximo 2h cada una), ordenadas por dependencia, y especificar
exactamente qué ficheros se crean o modifican. Agrupa las tasks en:
- Grupo 1: Dominio (entidad Turno, ConfiguracionTurnos)
- Grupo 2: Infraestructura (migración EF Core, configuración DbContext)
- Grupo 3: Aplicación (TurnosService con toda la lógica de negocio)
- Grupo 4: API (TurnosController con los 5 endpoints)
- Grupo 5: Frontend Angular (portal Gestor: abrir/cerrar turno, timer, historial)
- Grupo 6: Frontend Admin (lista de turnos con badge VIGILANCIA)
- Grupo 7: Tests de integración (WebApplicationFactory)
```

**Prompt 2** — Ticket de corrección: campo zona ausente en formularios del admin:

```
En el portal ADMIN, el formulario de crear incidencia no tiene el campo zona.
Necesito añadirlo de forma condicional: solo aparece cuando el usuario selecciona
categoria = 'SEGURIDAD'. Es un input de texto libre, opcional, max 200 caracteres.
Además, en la lista de incidencias del portal ADMIN, si una incidencia tiene zona y
su categoría es SEGURIDAD, debe mostrarse la zona debajo del título con un icono "place".
Lo mismo para el portal GESTOR. Hazlo sin romper el comportamiento actual para otras categorías.
```

**Prompt 3** — Ticket de migración para el rol VIGILANTE:

```
Necesito añadir soporte para el rol VIGILANTE sin breaking changes:
1. Migración EF Core: añadir columna TipoTurno a la tabla Turnos (varchar NOT NULL DEFAULT 'GESTION')
   y columna Zona a la tabla Incidencias (varchar(200) NULL).
2. DatabaseSeeder: añadir el rol 'VIGILANTE' a la tabla AspNetRoles (idempotente).
3. Añadir usuario de prueba vigilante1@example.com / Vigilante123! asignado al parking de demo.
4. En IncidenciasController, el método EsGestor() debe devolver true también para VIGILANTE,
   para que pueda ver todas las incidencias del parking (no solo las suyas).
El DEFAULT 'GESTION' garantiza que todos los turnos existentes siguen siendo válidos.
```

---

## 7. Pull Requests

**Prompt 1** — PR del módulo de turnos (change gestor-shifts):

```
/opsx:apply gestor-shifts

Implementa todas las tasks del change gestor-shifts en orden. Empieza por la entidad Turno
en el dominio, luego la migración, luego el servicio de aplicación TurnosService,
luego el controlador, y finalmente el portal Angular del gestor con el timer de turno activo.
Cuando termines el frontend del gestor, implementa también la vista de historial de turnos
en el portal ADMIN con el badge visual para identificar los turnos de vigilancia.
Marca cada task como completada en tasks.md inmediatamente al terminarla.
```

**Prompt 2** — PR del control de pagos con tests de integración:

```
/opsx:apply payment-control

Al llegar a las tasks de tests de integración del módulo de pagos:
escribe 13 tests que cubran los flujos completos de efectivo y Bizum.
Los tests deben usar una base de datos PostgreSQL real (plazahub_test), no mocks.
Cada test debe partir de un estado limpio: usa BeforeEachTest para truncar las tablas relevantes.
Asegúrate de que el test de descarga de PDF verifica que el Content-Disposition header
incluye filename="recibo_YYYY-MM_plaza_X.pdf" con la extensión correcta.
Los 13 tests deben pasar con dotnet test antes de marcar las tasks como completadas.
```

**Prompt 3** — PR del portal ADMIN completo:

```
/opsx:apply admin-portal

Implementa el portal Angular completo para el rol ADMIN. El portal tiene 13 secciones
accesibles desde el sidenav. Para la sección de "Personal" (anteriormente llamada "Gestores"):
cambia el nombre del item de menú de "Gestores" a "Personal" y añade un selector de rol
en el formulario de invitación con dos opciones: GESTOR y VIGILANTE.
Cada opción del select debe mostrar un icono, el nombre del rol y una descripción breve
de sus responsabilidades. El icono del item de menú cambia de manage_accounts a badge.
Asegúrate de que la compilación TypeScript de ng build pasa sin errores antes de marcar
las tasks de verificación como completadas.
```

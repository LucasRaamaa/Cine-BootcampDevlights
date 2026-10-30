# 🎬 Cinema API (LucasNodejsCine)

RESTful API para la gestión integral de un cine: administración de películas, salas, funciones y un sistema de reserva de butacas con **bloqueo preventivo temporal (Hold de 5 minutos)** para evitar compras duplicadas concurrentes.

---

## 🛠️ Stack Tecnológico

* **Runtime:** Node.js
* **Lenguaje:** TypeScript
* **Framework Web:** Express.js
* **Base de Datos:** MongoDB + Mongoose ODM
* **Seguridad:** JWT (JSON Web Tokens) + Bcrypt (Hashing de contraseñas)
* **Variables de Entorno:** Dotenv

---

## 📁 Estructura del Proyecto

```text
LucasNodejsCine/
├── .env                       # Variables de entorno locales
├── .env.example               # Plantilla de variables de entorno
├── .gitignore                 # Archivos ignorados por Git
├── package.json               # Dependencias y scripts del proyecto
├── tsconfig.json              # Configuración de compilación TypeScript
├── README.md                  # Documentación principal y roadmap
├── docs/                      # Documentación técnica detallada
│   ├── DATABASE.md            # Diagrama ERD, índices y consultas atómicas
│   └── COMPONENTS.md          # Responsabilidades de archivos y diagramas de secuencia
└── src/
    ├── config/
    │   └── database.ts        # Conexión a MongoDB con Mongoose
    ├── models/
    │   ├── User.ts            # Modelo de usuarios y credenciales
    │   ├── Movie.ts           # Modelo de películas
    │   ├── Room.ts            # Modelo de salas y dimensiones de grilla
    │   ├── Showtime.ts        # Modelo de funciones y horarios
    │   └── Booking.ts         # Modelo de reservas y hold temporal
    ├── middlewares/
    │   ├── auth.middleware.ts       # Verificación de JWT y roles (admin)
    │   ├── validation.middleware.ts # Validación de datos de entrada (400 Bad Request)
    │   └── error.middleware.ts      # Manejo centralizado de errores
    ├── controllers/
    │   ├── auth.controller.ts # Registro y login de usuarios
    │   ├── movie.controller.ts# ABM de películas
    │   ├── room.controller.ts # ABM de salas
    │   ├── showtime.controller.ts # Programación y consulta de funciones
    │   └── booking.controller.ts  # Lógica de hold (5 min) y confirmación
    ├── routes/
    │   ├── auth.routes.ts     # Endpoints de autenticación
    │   ├── movie.routes.ts    # Endpoints de películas
    │   ├── room.routes.ts     # Endpoints de salas
    │   ├── showtime.routes.ts # Endpoints de funciones
    │   ├── booking.routes.ts  # Endpoints de reservas
    │   └── index.ts           # Enrutador principal unificado
    └── index.ts               # Configuración de Express y arranque del servidor
```

### 📚 Documentación Técnica Detallada
* 🗄️ [Diagrama y Modelo de Base de Datos (docs/DATABASE.md)](file:///c:/Users/Lucas/Desktop/LucasNodejsCine/docs/DATABASE.md): Diagrama ERD, índices recomendados y estrategia de concurrencia para evitar sobreventas.
* 🧩 [Componentes y Diagramas de Secuencia (docs/COMPONENTS.md)](file:///c:/Users/Lucas/Desktop/LucasNodejsCine/docs/COMPONENTS.md): Responsabilidad de cada archivo y diagramas de flujo para Auth, Mapa de Butacas y Hold de 5 minutos.
* 📡 [Especificación de la API REST (docs/API.md)](file:///c:/Users/Lucas/Desktop/LucasNodejsCine/docs/API.md): Rutas, métodos HTTP, roles requeridos y formato de respuestas.
* ⚖️ [Reglas de Negocio (docs/BUSINESS_RULES.md)](file:///c:/Users/Lucas/Desktop/LucasNodejsCine/docs/BUSINESS_RULES.md): Reglas de hold temporal de 5 min, roles, estados de butacas y borrado lógico.
* 🗺️ [Plan de Implementación y Commits (docs/ROADMAP.md)](file:///c:/Users/Lucas/Desktop/LucasNodejsCine/docs/ROADMAP.md): Hoja de ruta paso a paso dividida en commits atómicos para Git.

---

## 🏛️ Modelo de Datos (Data Models)

### 1. `User`
* `_id`: ObjectId
* `email`: string (único, lowercase, requerido)
* `password`: string (hasheado con bcrypt)
* `role`: `'admin' | 'customer'` (default: `'customer'`)
* `timestamps`: `createdAt`, `updatedAt`

### 2. `Movie`
* `_id`: ObjectId
* `title`: string (requerido)
* `synopsis`: string
* `durationMinutes`: number (requerido)
* `genre`: string (ej: "Action", "Drama")
* `rating`: string (ej: "PG-13", "R", "ATP")
* `isActive`: boolean (default: `true`, para borrado lógico)
* `timestamps`: `createdAt`, `updatedAt`

### 3. `Room` (Sala)
* `_id`: ObjectId
* `name`: string (requerido, ej: "Screen 1", "Sala IMAX")
* `rows`: number (ej: 6 filas: A, B, C, D, E, F)
* `columns`: number (ej: 10 asientos por fila: 1 al 10)
* `capacity`: number (calculado: `rows * columns`)
* `roomType`: `'2D' | '3D' | 'IMAX' | '4DX'`
* `cinemaId`: ObjectId (opcional, preparado para escalar a múltiples cines)
* `isActive`: boolean (default: `true`)
* `timestamps`: `createdAt`, `updatedAt`

### 4. `Showtime` (Función)
* `_id`: ObjectId
* `movieId`: ObjectId (ref: `'Movie'`, requerido)
* `roomId`: ObjectId (ref: `'Room'`, requerido)
* `startTime`: Date (fecha y hora de inicio)
* `endTime`: Date (calculado: `startTime` + `durationMinutes`)
* `cleaningBufferMinutes`: number (default: `30`, tiempo de desinfección/limpieza antes de la próxima función)
* `price`: number (precio unitario de la entrada)
* `isActive`: boolean (default: `true`)
* `timestamps`: `createdAt`, `updatedAt`

### 5. `Booking` (Reserva y Bloqueo de Butacas)
* `_id`: ObjectId
* `userId`: ObjectId (ref: `'User'`, requerido)
* `showtimeId`: ObjectId (ref: `'Showtime'`, requerido)
* `seats`: `string[]` (ej: `["B-4", "B-5"]`)
* `status`: `'HELD' | 'CONFIRMED' | 'CANCELLED'`
* `heldUntil`: Date (marca de tiempo límite = creación + 5 minutos)
* `totalAmount`: number
* `timestamps`: `createdAt`, `updatedAt`

---

## ⏱️ Mecanismo de Hold Preventivo (Contador de 5 Minutos)

Para evitar que dos usuarios compren la misma butaca al mismo tiempo (*race condition*):

```text
1. Usuario selecciona butacas ["C-1", "C-2"]
   └── POST /api/bookings/hold

2. El sistema valida disponibilidad:
   ¿El asiento está en una reserva 'CONFIRMED' O en una 'HELD' con heldUntil > ahora?
   ├── SÍ: Error 409 Conflict ("Seat already reserved or held")
   └── NO: Se crea la reserva con status='HELD' y heldUntil = Date.now() + 5 min.
           La respuesta devuelve: { bookingId, heldUntil, remainingSeconds: 300 }

3. Resolución de la Reserva:
   ├── A. Confirmación a tiempo (POST /api/bookings/:id/confirm):
   │      Si Date.now() <= heldUntil -> Status pasa a 'CONFIRMED'.
   │
   ├── B. Cancelación manual (POST /api/bookings/:id/cancel):
   │      Status pasa a 'CANCELLED' -> Las butacas se liberan inmediatamente.
   │
   └── C. Vencimiento del tiempo (Expiración pasiva):
          Si pasan los 5 minutos sin confirmar, en la próxima consulta
          de asientos disponibles el sistema ignora las reservas 'HELD' vencidas.
          Las butacas vuelven a figurar libres sin necesidad de procesos bloqueantes.
```

---

## 📋 Catálogo de Endpoints de la API

### Autenticación (`/api/auth`)
* `POST /api/auth/register` - Registro de usuario
* `POST /api/auth/login` - Inicio de sesión y entrega de token JWT

### Películas (`/api/movies`)
* `GET /api/movies` - Listar películas activas
* `GET /api/movies/:id` - Detalle de una película
* `POST /api/movies` - [Admin] Crear película
* `PUT /api/movies/:id` - [Admin] Modificar película
* `DELETE /api/movies/:id` - [Admin] Baja lógica de película

### Salas (`/api/rooms`)
* `GET /api/rooms` - Listar salas
* `GET /api/rooms/:id` - Detalle de una sala (incluye grilla de filas y columnas)
* `POST /api/rooms` - [Admin] Crear sala
* `PUT /api/rooms/:id` - [Admin] Modificar sala
* `DELETE /api/rooms/:id` - [Admin] Baja lógica de sala

### Funciones (`/api/showtimes`)
* `GET /api/showtimes` - Listar funciones (con películas y salas pobladas)
* `GET /api/showtimes/:id` - Detalle de función
* `GET /api/showtimes/:id/seats` - Mapa de asientos con estado (`AVAILABLE`, `HELD`, `OCCUPIED`)
* `POST /api/showtimes` - [Admin] Programar función
* `PUT /api/showtimes/:id` - [Admin] Modificar función
* `DELETE /api/showtimes/:id` - [Admin] Cancelar función

### Reservas y Compras (`/api/bookings`)
* `POST /api/bookings/hold` - [Auth] Retener butacas por 5 minutos (inicia contador)
* `POST /api/bookings/:id/confirm` - [Auth] Confirmar y finalizar compra
* `POST /api/bookings/:id/cancel` - [Auth] Cancelar y liberar butacas antes de tiempo
* `GET /api/bookings/my-bookings` - [Auth] Historial de compras del usuario

---

## 🛡️ Validación de Entrada de Datos

Toda petición entrante (`req.body`, `req.params`) es interceptada por middlewares de validación antes de llegar a los controladores:
* **Auth:** Formato de email válido (`@`) y contraseña de al menos 6 caracteres.
* **Películas y Salas:** Título requerido, duración y dimensiones de sala mayores a 0.
* **Funciones:** Fecha y hora obligatoriamente en el futuro (`startTime > Date.now()`) y sin superposición de horarios en la misma sala (respetando los 30 min de limpieza).
* **Reservas:** Lista de asientos con formato válido (ej. fila `A-Z`, columna `> 0`).

*Cualquier dato incorrecto responde de inmediato con `HTTP 400 Bad Request` sin procesar lógica innecesaria.*

---

## 🚀 Roadmap de Ejecución (Paso a Paso)

```text
[ Fase 1 ] Entorno & Conexión DB
  ├── Variables de entorno y conexión con Mongoose
  └── Configuración base de Express

[ Fase 2 ] Autenticación & Roles
  ├── Modelo User (password con bcrypt)
  ├── Endpoints register y login
  └── Middleware de verificación JWT y control de rol 'admin'

[ Fase 3 ] Gestión de Salas y Películas (ABM)
  ├── CRUD de Movies (con borrado lógico)
  └── CRUD de Rooms (definición de dimensiones filas/columnas)

[ Fase 4 ] Gestión de Funciones (Cartelera)
  ├── CRUD de Showtimes
  └── Endpoint para consultar el mapa de asientos en tiempo real

[ Fase 5 ] Motor de Reservas & Hold de 5 Minutos
  ├── Lógica de bloqueo atómico de butacas
  ├── Contador regresivo y expiración automática
  └── Confirmación y cancelación de compras

[ Fase 6 ] Pruebas y Validación
  └── Pruebas de concurrencia y flujo completo
```

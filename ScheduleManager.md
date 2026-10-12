# Schecudle Manager v2 — Diseño OOP

Sistema de gestión de citas para salón de uñas. Backend en **FastAPI + SQLAlchemy 2.0 + PostgreSQL**. Arquitectura: **monolito modular por capas**.

---

## Patrón arquitectónico

```
router.py       ← recibe HTTP, valida con Pydantic, llama al service
service.py      ← lógica de negocio, orquesta repositorios e integraciones
repository.py   ← acceso a datos, queries SQL vía SQLAlchemy
schemas.py      ← modelos Pydantic para request/response
models.py       ← modelos SQLAlchemy (tablas)
```

**Módulos:** `auth` · `citas` · `clientes` · `servicios`
**Integraciones:** Google Calendar · WhatsApp (Twilio) · Email (SMTP)

---

## Enums

| Enum | Valores |
|---|---|
| `RolUsuario` | `ADMIN`, `EMPLEADO` |
| `EstadoCita` | `PENDIENTE`, `CONFIRMADA`, `COMPLETADA`, `CANCELADA`, `NO_SHOW` |
| `CanalOrigen` | `WEB`, `WHATSAPP`, `ADMIN` |
| `ConversacionPaso` | `INICIO`, `ESPERANDO_NOMBRE`, `ESPERANDO_SERVICIO`, `ESPERANDO_FECHA`, `CONFIRMANDO` |

---

## Mixins

### `TimestampMixin`
Aplicado a todos los modelos via composición (no herencia de negocio).

| Atributo / Método | Descripción |
|---|---|
| `created_at` | Timestamp de creación |
| `updated_at` | Timestamp de última modificación |
| `__pre_update__()` | Hook que actualiza `updated_at` automáticamente |

---

## Modelos (SQLAlchemy ORM)

Todos heredan de `Base` + `TimestampMixin`.

### `Usuario`
| Atributo | Tipo |
|---|---|
| `id` | `int` PK |
| `email` | `str` UNIQUE |
| `password_hash` | `str` |
| `nombre` | `str` |
| `rol` | `RolUsuario` |
| `activo` | `bool` |

### `Cliente`
| Atributo | Tipo |
|---|---|
| `id` | `int` PK |
| `nombre` | `str` |
| `apellido` | `str` |
| `email` | `str` UNIQUE nullable |
| `telefono` | `str` |
| `telefono_whatsapp` | `str` nullable |
| `notas_especiales` | `str` nullable |

### `Servicio`
| Atributo | Tipo |
|---|---|
| `id` | `int` PK |
| `nombre` | `str` |
| `descripcion` | `str` nullable |
| `precio` | `Decimal` |
| `duracion_minutos` | `int` |
| `activo` | `bool` |

### `Cita`
| Atributo | Tipo |
|---|---|
| `id` | `int` PK |
| `cliente_id` | `int` FK |
| `servicio_id` | `int` FK |
| `fecha_inicio` | `datetime` |
| `fecha_fin` | `datetime` |
| `estado` | `EstadoCita` |
| `canal_origen` | `CanalOrigen` |
| `notas` | `str` nullable |
| `google_event_id` | `str` nullable |
| `recordatorio_enviado` | `bool` |

---

## Schemas (Pydantic v2)

Herencia en cascada — el schema base define campos comunes.

```
BaseSchema
├── CitaBase
│   ├── CitaCreate(CitaBase)
│   ├── CitaUpdate(CitaBase)
│   └── CitaResponse(CitaBase)
├── ClienteBase
│   ├── ClienteCreate(ClienteBase)
│   ├── ClienteUpdate(ClienteBase)
│   └── ClienteResponse(ClienteBase)
├── ServicioBase
│   ├── ServicioCreate(ServicioBase)
│   └── ServicioResponse(ServicioBase)
└── UsuarioBase
    ├── LoginRequest(UsuarioBase)
    └── TokenResponse
```

`BaseSchema` configura `model_config`, serialización de fechas y alias generator.

---

## Repositories

`BaseRepository[T]` es genérico — recibe el tipo del modelo como parámetro. Cada repositorio hijo hereda CRUD y agrega sus queries propias.

### `BaseRepository[T]`
| Método | Retorno |
|---|---|
| `get_by_id(id)` | `T \| None` |
| `get_all(skip, limit)` | `list[T]` |
| `create(obj)` | `T` |
| `update(id, data)` | `T \| None` |
| `delete(id)` | `bool` |

### `UsuarioRepository(BaseRepository[Usuario])`
| Método | Retorno |
|---|---|
| `get_by_email(email)` | `Usuario \| None` |

### `ClienteRepository(BaseRepository[Cliente])`
| Método | Retorno |
|---|---|
| `get_by_telefono(telefono)` | `Cliente \| None` |
| `get_by_whatsapp(telefono)` | `Cliente \| None` |
| `buscar_por_nombre(query)` | `list[Cliente]` |
| `get_or_create_whatsapp(telefono, nombre)` | `Cliente` |

### `ServicioRepository(BaseRepository[Servicio])`
| Método | Retorno |
|---|---|
| `get_activos()` | `list[Servicio]` |

### `CitaRepository(BaseRepository[Cita])`
| Método | Retorno |
|---|---|
| `get_by_fecha(fecha)` | `list[Cita]` |
| `get_by_cliente(cliente_id)` | `list[Cita]` |
| `existe_solapamiento(inicio, fin)` | `bool` |
| `get_pendientes_recordatorio(ahora, limite)` | `list[Cita]` |

---

## Protocols (interfaces)

Python usa `Protocol` de `typing` para definir contratos sin forzar herencia. Esto habilita el polimorfismo real — `CitaService` no sabe ni le importa qué implementación recibe.

### `CalendarProvider`
| Método | Retorno |
|---|---|
| `crear_evento(cita: Cita)` | `str \| None` |
| `actualizar_evento(cita: Cita)` | `None` |
| `eliminar_evento(event_id: str)` | `None` |

### `NotificationProvider`
| Método | Retorno |
|---|---|
| `enviar_confirmacion(cita: Cita)` | `None` |
| `enviar_recordatorio(cita: Cita)` | `None` |

### `ChatbotProvider`
| Método | Retorno |
|---|---|
| `enviar_mensaje(numero: str, texto: str)` | `None` |
| `procesar_mensaje(numero: str, texto: str)` | `None` |

---

## Implementaciones de Providers

### `GoogleCalendarProvider` implementa `CalendarProvider`
| Método | Descripción |
|---|---|
| `__init__(credentials_json: str)` | Recibe el JSON de Service Account |
| `_get_service()` | Privado — inicializa el cliente de Google |
| `crear_evento(cita)` | Crea evento en Google Calendar |
| `actualizar_evento(cita)` | Actualiza evento existente |
| `eliminar_evento(event_id)` | Elimina evento del calendario |

### `EmailNotificationProvider` implementa `NotificationProvider`
| Método | Descripción |
|---|---|
| `__init__(mail_config)` | Configuración SMTP |
| `enviar_confirmacion(cita)` | Email HTML de confirmación |
| `enviar_recordatorio(cita)` | Email HTML 24h antes |
| `notificar_admin(asunto, contenido)` | Notificación interna al negocio |
| `_build_html(template, cita)` | Privado — construye el HTML del email |

### `WhatsAppNotificationProvider` implementa `NotificationProvider`
| Método | Descripción |
|---|---|
| `__init__(account_sid, auth_token, from_number)` | Credenciales Twilio |
| `enviar_mensaje(numero, texto)` | Envía mensaje WhatsApp |

### `TwilioChatbotProvider(WhatsAppNotificationProvider)` implementa `ChatbotProvider`
Hereda de `WhatsAppNotificationProvider` porque el chatbot **es** un proveedor de WhatsApp.

| Método | Descripción |
|---|---|
| `procesar_mensaje(numero, texto)` | Dispatcher principal del flujo |
| `_manejar_inicio(numero, texto, estado)` | Privado — paso 1 |
| `_manejar_nombre(numero, nombre, estado)` | Privado — paso 2 |
| `_manejar_servicio(numero, texto, estado)` | Privado — paso 3 |
| `_manejar_fecha(numero, texto, estado)` | Privado — paso 4 |
| `_manejar_confirmacion(numero, texto, estado)` | Privado — paso 5 |
| `_normalizar_numero(numero)` | Privado — limpia el formato de Twilio |

---

## Services

Lógica de negocio. Reciben repositorios y providers por inyección de dependencias (`Depends` de FastAPI).

### `AuthService`
| Método | Descripción |
|---|---|
| `__init__(usuario_repo, secret_key)` | |
| `login(email, password)` | Valida credenciales, retorna token |
| `create_access_token(data)` | Genera JWT firmado |
| `verify_token(token)` | Valida y decodifica JWT |
| `hash_password(password)` | bcrypt hash |
| `verify_password(plain, hashed)` | Comparación segura |

### `CitaService`
| Método | Descripción |
|---|---|
| `__init__(cita_repo, cliente_repo, servicio_repo, calendar, notifier)` | |
| `listar_todas()` | `list[CitaResponse]` |
| `listar_por_fecha(fecha)` | `list[CitaResponse]` |
| `listar_por_cliente(cliente_id)` | `list[CitaResponse]` |
| `buscar_por_id(id)` | `CitaResponse` |
| `crear(data: CitaCreate)` | Valida solapamiento, crea en BD, sincroniza Calendar, envía email |
| `cambiar_estado(id, estado)` | Actualiza estado, elimina evento si CANCELADA |
| `cancelar(id)` | Shortcut para `cambiar_estado(CANCELADA)` |
| `verificar_disponibilidad(inicio, servicio_id)` | `bool` |
| `_calcular_fecha_fin(inicio, duracion)` | Privado — suma duración |

### `ClienteService`
| Método | Descripción |
|---|---|
| `listar_todos()` | `list[ClienteResponse]` |
| `buscar_por_id(id)` | `ClienteResponse` |
| `buscar_por_nombre(query)` | `list[ClienteResponse]` |
| `crear(data: ClienteCreate)` | `ClienteResponse` |
| `actualizar(id, data: ClienteUpdate)` | `ClienteResponse` |
| `eliminar(id)` | `None` |

### `ServicioService`
| Método | Descripción |
|---|---|
| `listar_activos()` | `list[ServicioResponse]` |
| `listar_todos()` | `list[ServicioResponse]` |
| `buscar_por_id(id)` | `ServicioResponse` |
| `toggle_activo(id)` | Activa o desactiva el servicio |

---

## Jobs (APScheduler)

### `RecordatorioJob`
| Método | Descripción |
|---|---|
| `__init__(cita_service, notifier: NotificationProvider)` | |
| `ejecutar()` | Se registra en el scheduler, corre cada hora |
| `_procesar_cita(cita)` | Privado — envía recordatorio y marca enviado |

---

## Estado del chatbot

### `ConversacionState` (dataclass inmutable)

| Atributo | Tipo |
|---|---|
| `paso` | `ConversacionPaso` |
| `nombre` | `str \| None` |
| `servicio_id` | `int \| None` |
| `servicios` | `list[Servicio] \| None` |
| `fecha` | `datetime \| None` |

Métodos que retornan una nueva instancia (inmutabilidad):

| Método | Retorno |
|---|---|
| `con_paso(paso)` | `ConversacionState` |
| `con_nombre(nombre)` | `ConversacionState` |
| `con_servicio(id)` | `ConversacionState` |
| `con_fecha(fecha)` | `ConversacionState` |

---

## Conceptos OOP aplicados

| Concepto | Dónde se aplica |
|---|---|
| **Herencia** | `BaseRepository[T]` → repositorios concretos · `BaseSchema` → schemas · `WhatsAppNotificationProvider` → `TwilioChatbotProvider` |
| **Polimorfismo** | `CitaService` recibe `CalendarProvider` o `NotificationProvider` sin saber la implementación concreta |
| **Encapsulación** | Métodos con `_` son internos — `_get_service()`, `_build_html()`, `_calcular_fecha_fin()` |
| **Abstracción** | `Protocol` define *qué* hace cada provider sin decir *cómo* |
| **Composición** | `CitaService` compone `CalendarProvider` + `NotificationProvider` en lugar de heredar |
| **Mixins** | `TimestampMixin` agrega comportamiento a los modelos sin establecer jerarquía de negocio |
| **Generics** | `BaseRepository[T]` tipado con el modelo concreto — Mypy y el IDE lo entienden sin magia |

---

## Roadmap

### v2.1
- Dos manicuristas con disponibilidad independiente
- Horarios configurables por empleada
- Bloqueo pesimista para concurrencia
- Menú por categorías en chatbot
- Servicios adicionales por cita
- Cancelar / cambiar cita por WhatsApp

### v3.0
- Inventarios y proveedores
- Costos por servicio
- Dashboard de analytics
- Reportes de ingresos

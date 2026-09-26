# Control de Pañol (CPDS)

Plataforma web multiusuario para el control de inventario, préstamos de equipos y trazabilidad de activos en las bodegas y pañoles de **Duoc UC San Bernardo**. Proyecto de título (capstone).

## Qué es

Reemplaza el registro manual de los pañoles por una plataforma centralizada que permite:

- Registrar activos (control unitario, por ejemplo notebooks) y fungibles (control por stock), organizados por sección o pañol.
- Gestionar solicitudes y préstamos: el docente solicita, el encargado aprueba y entrega, el docente declara la devolución y el encargado la verifica.
- Mantener la trazabilidad de cada elemento por usuario y turno (diurno o vespertino).
- Avisos y notificaciones por correo, y un dashboard con indicadores.
- Roles y módulos de acceso configurables por grupo de usuarios.

El alcance es solo software: no hay integración con hardware (RFID, GPS).

Se trabaja con DSDM (Dynamic Systems Development Method): tiempo y calidad fijos, el alcance es lo que se ajusta mediante priorización MoSCoW, en incrementos de 15 días.

## Arquitectura

El sistema se construye como un **monolito modular con arquitectura en capas**, sobre el patrón MVT (Model-View-Template) nativo de Django, incorporando un **Service Layer** para separar la lógica de negocio de las vistas.

```
┌─────────────────────────────────────────────┐
│  Presentación                                │
│  views.py (CBV) + templates (server-render)  │
├─────────────────────────────────────────────┤
│  Aplicación / negocio                        │
│  services.py  → escrituras (comandos)        │
│  selectors.py → lecturas (consultas)         │
├─────────────────────────────────────────────┤
│  Dominio                                     │
│  models.py                                   │
├─────────────────────────────────────────────┤
│  Infraestructura                             │
│  PostgreSQL, Redis, RabbitMQ, Celery         │
└─────────────────────────────────────────────┘
```

Reglas que sostienen esta separación:

- Cada app de dominio (`assets`, `fungibles`, `kits`, `inventory`, `locations`, `loans`, etc.) sigue el mismo patrón de archivos: `models.py`, `selectors.py`, `services.py`, `forms.py`, `views.py`, `urls.py`, `admin.py`, `templates/`.
- `selectors.py` contiene únicamente consultas de lectura; nunca escribe en la base de datos.
- `services.py` contiene las escrituras, envueltas en `@transaction.atomic` y con `full_clean()` antes de `save()`, para garantizar integridad de datos y atomicidad.
- La lógica de negocio nunca vive en vistas ni en templates.
- Las tablas de auditoría (`AssetStatusLog`, `StockMovement`, `KitRevision`) son de solo lectura desde el admin de Django (`has_add_permission`/`has_change_permission` en `False`) y solo se generan a través de la capa de servicios, evitando que se editen o creen registros de trazabilidad por fuera del flujo controlado.

**Control de acceso por módulos:** en lugar de acoplar permisos a nombres de rol fijos, el sistema usa un modelo `Modulo` (app `core`) con relación M2M a `django.contrib.auth.models.Group` (`grupos_permitidos`). Cada app se registra automáticamente vía `post_migrate` heredando `ModuloAppConfig`, y el acceso a cada vista se resuelve con `ModuloRequeridoMixin`, combinado con `LoginRequiredMixin`. Esto permite reconfigurar qué rol ve qué módulo sin tocar código ni desplegar una nueva versión.

**Por qué esta arquitectura y no otra:** dado un equipo de 4 personas, un calendario académico fijo de 18 semanas y un hardware de despliegue con recursos acotados (Azure for students), un monolito modular con separación en capas entrega el beneficio principal de las arquitecturas más elaboradas (lógica de negocio testeable y desacoplada de la capa de presentación) sin el costo de complejidad adicional de una arquitectura de microservicios o hexagonal (puertos, adaptadores, inversión de dependencias), que no se justifica para el alcance y los plazos del proyecto.

## Stack tecnológico

| Componente | Tecnología | Rol en el sistema |
|---|---|---|
| Lenguaje | Python 3.11 | Runtime del backend |
| Framework backend | Django 5.2 | Enrutamiento, ORM, vistas y plantillas server-side |
| Base de datos | PostgreSQL 17 | Persistencia relacional, integridad referencial y transacciones |
| Cola de tareas | Celery 5.4 (worker + beat) | Envío de correos y tareas programadas/asíncronas |
| Broker de mensajería | RabbitMQ 3.11 | Cola de mensajes entre Django y los workers de Celery |
| Caché y rate limiting | Redis 7.4 | Caché de aplicación y backend compartido para `django-ratelimit` (necesario en despliegues con más de un worker; `LocMemCache` no sirve porque cuenta por proceso) |
| Frontend | Bootstrap 5 (tema Sneat), jQuery, Chart.js 4.5.1, Boxicons | Interfaz server-rendered, sin framework SPA |
| Contenedores | Docker y Docker Compose | Empaquetado y orquestación de todos los servicios |
| Gestión de dependencias | pip + `requirements/constraints.txt` | Versionado exacto y reproducible del entorno completo |

**Notas de seguridad relevantes del stack:**

- Los tokens CSRF se leen desde `<meta name="csrf-token">`, nunca desde `document.cookie`.
- `CSRF_TRUSTED_ORIGINS` debe configurarse explícitamente por entorno (un bug previo por `split(',')` sobre string vacío ya fue corregido).
- Está planificado migrar a `CSRF_COOKIE_HTTPONLY=True`, dado que la lectura del token ya no depende de JavaScript accediendo a la cookie.
- Los recursos cargados desde CDN deben fijarse siempre con versión exacta (nunca `latest`) y, antes de producción, con hash SRI (Subresource Integrity).
- Las dependencias se auditan contra vulnerabilidades conocidas con `pip-audit` sobre `constraints.txt` antes de cada actualización.
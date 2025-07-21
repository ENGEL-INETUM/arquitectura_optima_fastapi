

# Arquitectura recomendada: Monolito vs Microservicios

## Enfoque Monolito

En el enfoque **monolítico** todo el backend corre en un solo servicio. Todas las rutas, lógica de negocio y acceso a la base de datos SQL están centralizadas en una única aplicación FastAPI desplegada (un mismo proceso o contenedor). Esto simplifica el desarrollo inicial y la coherencia (solo hay un repositorio y una línea de despliegue), pero a medida que crece la aplicación aumenta el acoplamiento: cualquier cambio puede requerir redeploy completo del sistema. Para aplicaciones pequeñas o equipos reducidos, un monolito modular resulta ágil de mantener. Se recomienda estructurar el código por dominios de negocio dentro del mismo proyecto (por ejemplo paquetes `usuarios/`, `pedidos/`, etc.) en lugar de dividir sólo por tipo de archivo.

-   **Ventajas**: despliegue sencillo (un solo contenedor), sin costos de comunicación interservicios, debugging e integración más directos.
    
-   **Desventajas**: menor tolerancia a fallos (si un módulo falla puede tumbar todo), escalado masivo menos fino (hay que replicar la aplicación completa), y menor flexibilidad tecnológica (todos los módulos comparten versiones de librerías). En un monolito, si la base de datos falla, la aplicación entera lo hace.
    

### Estructura de proyecto (Monolítico)

Una buena práctica es organizar el monolito por dominios o funcionalidades. Por ejemplo:

```
fastapi_app/
├── src/
│   ├── main.py            # Inicialización de la app FastAPI
│   ├── usuarios/         # Dominio de usuarios
│   │   ├── router.py     # Rutas (APIRouter) de usuarios
│   │   ├── models.py     # Modelos ORM (SQLAlchemy) de usuarios
│   │   ├── schemas.py    # Pydantic (serialización) de usuarios
│   │   └── service.py    # Lógica de negocio de usuarios
│   ├── pedidos/         # Dominio de pedidos (simétrico)
│   │   ├── router.py
│   │   ├── models.py
│   │   ├── schemas.py
│   │   └── service.py
│   ├── database.py      # Conexión y sesión de la base de datos
│   ├── config.py        # Configuraciones globales (p.ej. BaseSettings de Pydantic)
│   └── utils.py         # Código compartido (paginación, etc.)
├── tests/               # Pruebas unitarias e integración
│   ├── test_usuarios.py
│   └── test_pedidos.py
├── requirements.txt
└── alembic/             # Migraciones de base de datos (SQLAlchemy + Alembic)

```

Cada paquete (e.g. `usuarios/`) contiene su `router.py`, `models.py`, `schemas.py`, etc. Este patrón de capas (rutas, modelos, servicios) facilita la separación lógica. La configuración global (`database.py`, `config.py`) gestiona la conexión al SQL y otras dependencias. Este tipo de estructura **modular** evita el “código espagueti” y permite escalar el monolito con relativa orden.

### Buenas prácticas (Monolito)

-   **Escalabilidad**: Aunque es una sola aplicación, se puede escalar horizontalmente replicando la instancia (por ejemplo usando múltiples procesos Uvicorn/Gunicorn) y equilibrando carga (load balancer). Se recomienda usar _workers_ de Uvicorn según núcleos CPU, y configurar un pool de conexiones a la base de datos. Emplear drivers asíncronos (p.ej. `asyncpg` con SQLAlchemy) es considerado una buena práctica en FastAPI para que las consultas SQL no bloqueen el servidor. También es útil implementar caching en rutas de alta demanda (Redis, memcached) y tareas en segundo plano (Celery o FastAPI BackgroundTasks) para operaciones costosas.
    
-   **Pruebas (Testing)**: Utilizar `pytest` con el `TestClient` de FastAPI. Se recomienda escribir **pruebas unitarias** para cada capa (servicios, utilitarios) y **pruebas de integración** contra la aplicación completa. Mantener los tests desde el inicio y con buena cobertura. Separar tests en carpetas (p.ej. `tests/test_usuarios.py`). Configurar una base de datos de prueba (in-memory o contenedores) en CI. FastAPI facilita la prueba del API gracias a su `TestClient` (via Starlette).
    
-   **Documentación**: FastAPI genera automáticamente documentación OpenAPI (Swagger UI y ReDoc) basada en los modelos de Pydantic. Cada endpoint documentado con tipos claros es autodescriptivo. Es buena práctica agregar descripciones a rutas, modelos y parámetros. Además, mantener un README general con arquitectura y endpoints principales. El uso de migraciones (Alembic) y convenciones de nombres en la base de datos ayuda a la mantenibilidad.
    
-   **Mantenimiento**: Un monolito requiere un solo pipeline de CI/CD. Es clave versionar y etiquetar la aplicación completa, y disponer de despliegues reproducibles (p.ej. Docker). Se recomienda usar entornos virtuales o contenedores para aislar dependencias. La revisión de código (linters como `ruff`, formateadores como `black`) mantiene la calidad. Para el monitoreo, se puede usar middleware de logging (p.ej. `loguru`) y health-checks básicos. En equipos grandes, modularizar por dominios evita que varios desarrolladores choquen en el mismo código. Sin embargo, hay que ser conscientes de que todo cambio debe ser cuidadosamente testeado, pues puede impactar otras partes del sistema.
    

## Enfoque por Microservicios

_Figura: Diagrama de arquitectura de microservicios típica (un API Gateway delega a múltiples servicios FastAPI independientes, cada uno con su propia base de datos)._ En un diseño **microservicios** cada subdominio (por ejemplo autenticación, catálogo de productos, órdenes, notificaciones, etc.) se implementa como un servicio FastAPI independiente, con su propio ciclo de vida y base de datos relacional aislada. Los servicios se comunican entre sí mediante HTTP/REST (u otro protocolo ligero) o mensajería (RabbitMQ, Kafka). Este enfoque **desacopla** los módulos: cada servicio se desarrolla, despliega y escala de forma autónoma. Aunque la complejidad es mayor (orquestación, comunicaciones), la escalabilidad y mantenibilidad aumentan en sistemas grandes: por ejemplo, si un servicio falla, los demás siguen funcionando. Además, cada servicio puede usar el stack o base de datos más adecuada a su dominio.

-   **Ventajas**: despliegue y escalado independientes por servicio (p.ej. durante alta demanda sólo escalar el servicio “pago” sin tocar el resto), alta tolerancia a fallos, facilidad para aplicar DevOps CI/CD por módulo, y flexibilidad tecnológica (cada equipo puede elegir su versión de librerías o hasta lenguaje distinto).
    
-   **Desventajas**: complejidad operativa (se requieren contenedores/Docker, orquestadores como Kubernetes o Docker Swarm, manejo de redes internas), necesidad de API Gateway o balanceador para enrutar peticiones, y mayor esfuerzo en testing (desde pruebas de contratos a pruebas de integración entre servicios). La latencia de red y la consistencia de datos distribuidos también deben gestionarse.
    

### Estructura de proyecto (Microservicios)

En este enfoque, cada microservicio es un proyecto FastAPI autónomo. Ejemplo de organización (monorepo o repos separados):

```
proyecto-microservicios/
├── gateway/
│   ├── main.py         # App FastAPI que actúa como puerta de enlace (API Gateway)
│   └── requirements.txt
├── auth-service/
│   ├── main.py         # App FastAPI de autenticación
│   ├── models.py       # Modelos de usuario
│   ├── schemas.py      # Pydantic de usuario
│   └── requirements.txt
├── orders-service/
│   ├── main.py         # App FastAPI de gestión de órdenes
│   └── requirements.txt
├── ml-service/
│   ├── main.py         # App FastAPI de procesamiento de ML (p.ej. OCR)
│   └── requirements.txt
└── notification-service/
    ├── main.py         # App FastAPI de notificaciones
    └── requirements.txt

```

Cada carpeta contiene su propia aplicación FastAPI completa (rutas, servicios, modelos, etc.). Se suele incluir en cada servicio su `Dockerfile` y configuración (`.env`) separada. Como patrones, se recomienda que cada microservicio internamente tenga una estructura modular similar al monolito (e.g. subpaquetes `router.py`, `service.py`, `models.py`).

El **API Gateway** (puerta de enlace) es opcionalmente otro servicio que redirige peticiones a los microservicios apropiados (o se usa un balanceador Nginx). FastAPI incluso permite _montar_ apps hijas en rutas de un app padre, aunque en producción lo común es un gateway externo. Cada microservicio _deployeado_ expone su propio conjunto de endpoints documentados con OpenAPI.

### Buenas prácticas (Microservicios)

-   **Escalabilidad**: Contenerizar cada servicio (Docker) y usar orquestadores (Kubernetes/ECS) para autoescalado. Cada servicio se puede replicar independientemente según demanda. Se recomienda diseñar las APIs de forma _stateless_ y delegar estado (sesiones, cache) a sistemas externos (Redis, bases de datos). Emplear programación asíncrona (`async def` + drivers async) por servicio mejora la concurrencia. También es común implementar colas de mensajes (RabbitMQ/Kafka) para desacoplar comunicaciones y tolerancia a fallos. El enfoque de 12-Factor App (configuración vía variables de entorno, logs en stdout, one process per container) es aplicable a cada microservicio.
    
-   **Pruebas (Testing)**: Testear cada servicio de forma aislada con `pytest` y `TestClient`. Además de **unit tests**, conviene hacer **pruebas de integración** entre servicios (p.ej. simulando el API Gateway o utilizando un entorno Docker con todos los servicios levantados). Se pueden usar _contract tests_ para asegurar que la interfaz de cada servicio no rompe clientes. Automatizar tests en cada commit mediante CI/CD (GitHub Actions, GitLab CI, etc) es fundamental. Cada contenedor debe pasar por un pipeline de pruebas individual.
    
-   **Documentación**: Al igual que en el monolito, FastAPI genera docs automáticos OpenAPI/Swagger para cada servicio. Es recomendable mantener un repositorio central de documentación (p.ej. portal API) que enlace las especificaciones de cada microservicio, o usar un gateway que consolide la documentación. También documentar flujos de comunicación entre servicios (eventos, rutas). Mantener diagramas de arquitectura actualizados en el README general.
    
-   **Mantenimiento**: Con múltiples servicios, es vital la observabilidad: usar logging centralizado (p.ej. enviar logs a un ELK/Datadog), monitoreo de salud (puertas de enlace, contenedores) y rastreo distribuido si es posible. Adoptar CI/CD automatizado por servicio para despliegues confiables. Cada microservicio debe versionarse por separado (tags o números de versión propios). Gestionar debidamente las dependencias en cada servicio (evitar duplicar la configuración). Dado que los datos están repartidos, hay que planear migraciones independientes para cada base de datos. Por último, prever la idempotencia y retrocompatibilidad de las APIs (p.ej. versionar endpoints) para facilitar actualizaciones escalonadas.
    

En resumen, **un monolito modular** es más sencillo de implementar y probar inicialmente, ideal cuando el proyecto es pequeño o en etapas tempranas. En cambio, **una arquitectura de microservicios** aporta mayor flexibilidad y escalabilidad cuando la aplicación crece en complejidad y equipos, aunque a costa de más complejidad operativa. Ambos enfoques deben seguir buenas prácticas de FastAPI: diseño por capas, uso de Pydantic/SQLAlchemy, pruebas exhaustivas, y despliegues reproducibles (por ejemplo mediante Docker y pipelines de CI/CD).

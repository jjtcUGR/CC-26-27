# Backend de Análisis de Telemetría para Sim-Racing

## 1. Descripción del Dominio
El proyecto consiste en el desarrollo de una Aplicación Nativa de la Nube (Cloud-Native) orientada a la comunidad de *sim-racing*. La problemática actual reside en que los pilotos y jugadores de títulos de conducción (como la saga Forza Horizon) generan grandes volúmenes de datos de telemetría y configuraciones mecánicas (*setups* de suspensión, aerodinámica, etc.) que se almacenan de forma local y asilada.

Esta plataforma proporciona una API centralizada para la ingesta masiva de tiempos por vuelta y ajustes de vehículos, permitiendo cruzar datos, filtrar configuraciones por circuito y vehículo, y compartir los *setups* óptimos. El sistema abandona el modelo local o cliente-servidor tradicional para adoptar una arquitectura de microservicios que soporte picos de peticiones masivas al finalizar las carreras y permita consultas ágiles desde distintas interfaces web o de terceros.

## 2. Justificación de la Arquitectura (NIST SP 800-145)
El desarrollo y despliegue de esta plataforma se justifica por las 5 características esenciales del Cloud Computing:

* **Autoservicio bajo demanda:** Los pilotos pueden interactuar con la API para registrar nuevas telemetrías o consultar configuraciones en cualquier momento, sin necesidad de que un administrador aprovisione recursos o valide el acceso manualmente.
* **Acceso amplio a la red:** La lógica de negocio se expone a través de una API RESTful estándar (HTTP/JSON), permitiendo su consumo desde terminales, aplicaciones web o integraciones directas en *plugins* de simuladores.
* **Asignación común de recursos (Resource pooling):** La capa de persistencia y el procesamiento de la plataforma sirven a múltiples usuarios (multitenancy) ocultando la complejidad física y la ubicación exacta de los servidores.
* **Rápida elasticidad:** La separación en microservicios permite escalar independientemente el servicio de "Ingesta" durante los picos de concurrencia (ej. torneos o fines de semana) y reducir recursos en horas valle, manteniendo el microservicio de "Búsqueda" estable.
* **Servicio medido:** La infraestructura de despliegue monitorizará de forma transparente el uso de CPU/RAM de los contenedores, el almacenamiento de las bases de datos y el ancho de banda, permitiendo optimizar el impacto económico y establecer cuotas de uso de la API por usuario.

## 3. Mapeo de Actores (NIST SP 500-292)
* **Consumidor (Cloud Consumer):** Los pilotos de sim-racing, equipos o aplicaciones de terceros que consumen los *endpoints* de lectura y escritura de telemetría.
* **Proveedor (Cloud Provider):** El equipo de desarrollo (estudiantes) que asume el rol de diseñar, desplegar y mantener la disponibilidad de los microservicios y la base de datos en una plataforma en la nube (ej. AWS, Azure, GCP o un PaaS).
* **Intermediario (Cloud Broker):** Posibles servicios de autenticación federada (OAuth de GitHub/Google) o pasarelas de monitorización externas que añaden valor a la interconexión entre el consumidor y el proveedor.
* **Auditor (Cloud Auditor):** Herramientas automatizadas de observabilidad y *healthchecks* que verifican los tiempos de latencia, el SLA del sistema y el cumplimiento de las políticas de almacenamiento.
* **Operador (Cloud Carrier):** Los proveedores de servicios de internet (ISPs) que garantizan el enrutamiento y la baja latencia de los paquetes de telemetría entre los clientes de juego y la nube.

## 4. Arquitectura de Referencia
El sistema sigue una arquitectura distribuida basada en microservicios independientes, comunicados por HTTP, separando la carga intensiva de escritura de la carga intensiva de lectura:

1. **Microservicio de Ingesta (Write API):** Dedicado exclusivamente a recibir ráfagas de datos al final de cada vuelta, validar el esquema de datos (aerodinámica, presiones, geometría) y almacenarlo.
2. **Microservicio de Búsqueda (Read API):** Motor de consultas que expone los datos filtrados por modelos de vehículo, circuitos o tiempos de vuelta.
3. **Capa de Persistencia:** Base de datos NoSQL (ej. MongoDB) ideal para almacenar documentos JSON flexibles de telemetría y configuraciones.

## 5. Especificación de API REST (Endpoints principales)

| Método | Endpoint | Payload / Params | Descripción |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/telemetria` | JSON: `{id_piloto, vehiculo, circuito, tiempos, setup_mecanico}` | Registra una nueva sesión de telemetría en la base de datos. Retorna `201 Created`. |
| `GET` | `/api/setups` | Parámetros: `?vehiculo=X&circuito=Y` | Devuelve una lista de configuraciones mecánicas ordenadas por tiempo de vuelta. Retorna `200 OK`. |
| `GET` | `/health` | Ninguno | Endpoint ligero de estado. Devuelve `{"status": "ok", "service": "nombre_servicio"}` en <500ms para el balanceador de carga. |
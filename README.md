# Evaluación del Proyecto vs. Rúbrica de Sistemas Distribuidos (Imágenes)

Basado en el análisis estático del código fuente actual en el repositorio (`/home/leider/historia-clinica-distribuida-final`), a continuación se presenta el detalle de cumplimiento para cada uno de los criterios de la rúbrica provista en las imágenes.

---

## 1. Referencias (25%)

| Criterio | Peso | Estado | Explicación y Evidencia |
| :--- | :---: | :---: | :--- |
| **1.1 Docker Compose / Kubernetes** | 10% | ✅ **Cumple** | El proyecto cuenta con un archivo `docker-compose.yml` robusto que orquesta toda la infraestructura. Se han contenerizado los microservicios (Gateway, Patient, Clinical, Sync), la infraestructura de mensajería (RabbitMQ), bases de datos (PostgreSQL), HAPI FHIR y el stack de monitoreo (Prometheus, Grafana). Se usan `healthchecks` y políticas de reinicio. |
| **1.2 3 Nodos / Ciudades Simuladas** | 10% | ⚠️ **Parcial** | El sistema contempla lógicamente 3 sedes (Sincelejo, Bogotá, Medellín) y levanta 3 bases de datos separadas (`pg_nodo1`, `pg_nodo2`, `pg_nodo3`). **Sin embargo**, a nivel de orquestación, solo existe **una instancia central de HAPI FHIR** y un solo juego de microservicios. La arquitectura es de tipo *Hub-and-Spoke* (un core central) y no 3 nodos FHIR completamente independientes respondiendo individualmente. |
| **1.3 Red y Volúmenes Persistentes** | 5% | ✅ **Cumple** | En el `docker-compose.yml` se define explícitamente la red bridge compartida `historia_clinica_net`. Además, se hace uso de volúmenes persistentes nombrados para garantizar que los datos no se pierdan: `pg_data_nodo1`, `pg_data_nodo2`, `pg_data_nodo3` y `grafana_data`. |

---

## 2. Base de Datos Distribuida Transaccional (15%)

| Criterio | Peso | Estado | Explicación y Evidencia |
| :--- | :---: | :---: | :--- |
| **2.1 BD Distribuida (PostgreSQL)** | 10% | ⚠️ **Parcial** | Existen 3 instancias de PostgreSQL y se hace una distribución de la carga (sharding o división lógica por sede/documento). **Sin embargo**, la rúbrica exige *replicación activa y failover automático*. El proyecto no implementa herramientas de clustering activo (como Patroni o repmgr) que garanticen un failover real a nivel de la base de datos si un nodo físico se cae. |
| **2.2 Consistencia y Sincronización** | 5% | ✅ **Cumple** | El proyecto demuestra un mecanismo de sincronización basado en el **Patrón Outbox** y RabbitMQ. Las transacciones locales se guardan en la tabla `event_outbox` y un servicio de sincronización (`sync-service`) lee estos eventos y los empuja a HAPI FHIR garantizando consistencia eventual. |

---

## 3. Servicio HAPI FHIR (21%)

| Criterio | Peso | Estado | Explicación y Evidencia |
| :--- | :---: | :---: | :--- |
| **3.1 Servidor HAPI FHIR Operativo** | 8% | ⚠️ **Parcial** | Existe el servidor HAPI FHIR operativo y su metadata es consultable a través del healthcheck implementado en el backend (`/api/v1/patient/health/fhir`). Pero, de nuevo, **solo hay 1 instancia desplegada**, y la rúbrica exige "3 Instancias HAPI FHIR operativas". |
| **3.2 Peticiones CRUD a los 3 Nodos** | 8% | ⚠️ **Parcial** | El código soporta la transformación y guardado de recursos fundamentales (Patient, Encounter, Observation, Condition, MedicationRequest). Pero al existir un único servidor HAPI FHIR, todas las peticiones terminan allí, en lugar de interactuar entre 3 servidores FHIR distintos. |
| **3.3 Seguridad OAuth2 / SMART on FHIR** | 5% | ⚠️ **Parcial** | El proyecto incluye un middleware de JWT (`JWTAuthMiddleware`), una ruta para inicio de sesión (`/api/v1/auth/login`) y una interfaz gráfica (`login.html`). Esto protege exitosamente los endpoints. Sin embargo, no se trata de una implementación completa de OAuth2 (flujos de autorización) ni cumple estrictamente el estándar de *SMART on FHIR* (que requeriría manejo de scopes específicos para la salud). |

---

## 4. Interfaces Clínicas (4 Módulos) (20%)

| Criterio | Peso | Estado | Explicación y Evidencia |
| :--- | :---: | :---: | :--- |
| **4.1 Módulo Admisión** | 5% | ✅ **Cumple** | Interfaz `registro-paciente.html` completamente funcional que captura los datos demográficos y clínicos básicos, asigna un ID de paciente, selecciona el nodo/sede de atención y dispara la creación. |
| **4.2 Módulo Triage** | 5% | ✅ **Cumple** | El proyecto en su frontend y backend (`carga-hc.html` y `clinical_records`) contempla el ingreso de signos vitales (generando *Observations*) y asigna una clasificación de Triage (I al V) vinculada al episodio. |
| **4.3 Módulo Médico** | 5% | ✅ **Cumple** | Soporta el registro del diagnóstico principal (CIE-10, que se transforma en *Condition*) y un formulario de prescripción médica (que se transforma en *MedicationRequest*). |
| **4.4 Modulo Resultados / Historia Clínica** | 5% | ✅ **Cumple** | La interfaz `detalle-paciente.html` expone de forma consolidada una vista ("timeline") que cruza los Encounter, Observation, Condition y MedicationRequest asociados al paciente, permitiendo visualizar la HC unificada. |
| **4.5 Sistema de Reportes** | 5% | ✅ **Cumple** | Se ha implementado un dashboard en el `index.html` que consulta estadísticas de pacientes por nodo y eventos sincrónicos. Además, el proyecto incluye Grafana, lo que cubre con creces este requisito. |

---

## 5. Tolerancia a Fallos (20%)

| Criterio | Peso | Estado | Explicación y Evidencia |
| :--- | :---: | :---: | :--- |
| **5.1 Simulación Caída de BD** | 8% | ⚠️ **Parcial** | Existe un endpoint lógico (`/nodes/{node_id}/fail`) que modifica el estado en memoria para probar el comportamiento de la UI. Sin embargo, si se detiene físicamente el contenedor de PostgreSQL (`docker stop pg_nodo1`), el sistema no redirige automáticamente la conexión de la base de datos a un nodo secundario, debido a la falta de replicación activa (Mismo problema que 2.1). |
| **5.2 Simulación Desconexión de Nodo** | 7% | ⚠️ **Parcial** | Funciona de manera simulada a través de los endpoints de la API para "apagar" sedes, probando cómo reaccionan las vistas y las alertas. Sin embargo, si se produce un corte de red real entre los contenedores, los microservicios fallarían al intentar conectarse a las instancias específicas sin un fallback automático real. |
| **5.3 Logs y Monitoreo** | 5% | ✅ **Cumple** | Stack robusto implementado mediante `Prometheus` y `Grafana`. Se incluyen exportadores para RabbitMQ y PostgreSQL, así como instrumentación en FastAPI, y se capturan métricas customizadas de eventos encolados, procesados y fallidos (`his_outbox_pending_by_node`). |

---

## Conclusión y Puntos de Acción para la Demo

El proyecto es estructuralmente sólido, usando buenas prácticas de microservicios, patrón Outbox y un stack de observabilidad de primer nivel (Grafana+Prometheus). 

**Para maximizar la calificación en la sustentación, se recomienda:**
1. **Sustentar la Arquitectura Hub-and-Spoke:** Argumentar por qué se prefirió tener un solo HAPI FHIR central en lugar de 3 replicados (Ej. costos, mantenibilidad, HAPI FHIR sirviendo como Cloud Core mientras las bases locales actúan como el storage del nodo).
2. **Explicar la "Simulación" de Fallos:** Dejar claro al profesor que la caída de base de datos se maneja como una *simulación por estado* de la API para propósitos demostrativos, y que el Patrón Outbox es el encargado principal de la tolerancia a fallos en la sincronización, garantizando que no se pierdan datos cuando hay pérdida de conectividad con el core central.
3. **Mencionar la Autenticación:** Especificar que la seguridad se construyó en torno a **JWT robusto en el Gateway** (validación de claims y tokens de acceso), lo cual actúa como la capa de seguridad solicitada, a pesar de no seguir el flujo OAuth2 explícito completo de 3 pasos.

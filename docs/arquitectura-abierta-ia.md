# Propuesta de arquitectura abierta e IA de software libre

Estado: diseño documental, 13 de octubre de 2026. Complementa el [compendio](../README.md). No contiene servicios implementados.

## Objetivo y alcance

Construir una plataforma autoalojable de estacionamientos con componentes sustituibles. La primera instalación registra ocupación, disponibilidad, reservas y sesiones. Videovigilancia y modelos de IA aportan observaciones; un servicio transaccional mantiene las decisiones comerciales.

Alcance inicial: una instalación, plazas delimitadas, una cámara fija, un sensor de contraste, panel de operador y reservas de prueba. Cobros, reconocimiento de matrículas, identificación biométrica, flotas robóticas y actuación automática se evalúan después como módulos independientes.

## Responsabilidades y autoridad del dato

| Servicio | Responsabilidad y dato autorizado | Interfaz propuesta |
|---|---|---|
| Captura de vídeo | Capturar y conservar evidencia según política | RTSP de cámara; interfaz del NVR por verificar |
| Ocupación de borde | Estimar presencia en polígonos calibrados y su calidad | Salida al adaptador, no escritura directa de reservas |
| Adaptador | Validar esquema, mapear identificadores, deduplicar y almacenar pendientes | MQTT autenticado |
| Estacionamientos | Instalaciones, plazas, reservas, sesiones y decisiones de acceso | API HTTP versionada por desarrollar |
| PostgreSQL | Registro transaccional y auditoría durable | Acceso desde servicios, sin exposición al cliente |
| Ditto opcional | Proyección del estado de activos, zonas y calidad | Conector por desarrollar |
| IA e historial | Entrenamiento, evaluación y recomendaciones | Consultas de datos autorizadas; resultados versionados |
| Identidad y panel | Usuarios, roles, operación y explicación de decisiones | OIDC donde el cliente lo admita |

El servicio de estacionamientos es autoridad para reservas y sesiones. El adaptador produce observaciones físicas. Ditto es una proyección y no concede reservas. La disponibilidad se calcula con ocupación, reservas vigentes, plaza habilitada y calidad de observación; la falta de señal nunca se interpreta como plaza libre.

## Decisiones de arquitectura

- **Servicios separados:** interfaces públicas documentadas, exportación de datos y reemplazo de componentes sin rehacer el dominio.
- **Un gestor de reservas:** evaluar Ezy-Parking o desarrollar uno mínimo; TheParker sirve como cliente o referencia funcional tras adaptar sus dependencias.
- **Un NVR principal:** Kerberos Agent del compendio, con Frigate como alternativa. No añadir ambos sin un motivo operativo.
- **MQTT para observaciones:** usar autenticación, TLS y permisos por instalación; una cola durable del adaptador y un registro del consumidor cubren recuperación. No asumir que QoS aporta transacciones de negocio.
- **PostgreSQL para consistencia:** reservas y sesiones con transacciones; publicación mediante patrón outbox y consumo idempotente.
- **IA local y evaluable:** primero reglas y referencias estadísticas. Elegir un modelo complejo solo si mejora resultados medidos.
- **Gemelos después del flujo básico:** Ditto representa activos y facilita integración con JFXAI4CBS. Para simulación predictiva se requiere además un modelo validado; una proyección del estado no basta.

## Adaptación del compendio

| Entrada | Adaptación propuesta | Condición previa |
|---|---|---|
| Kerberos Agent | Conector de metadatos y referencias protegidas a vídeo | Inspeccionar interfaces y edición exacta; no asumir términos de otros productos Kerberos |
| Parker | Exportar observaciones por plaza hacia el adaptador | Revisar pesos y continuidad; una pestaña cerrada no sostiene operación |
| TheParker | Migrar identidad y datos hacia servicios autoalojados; revisar cliente Android | Renovar dependencias, verificar permisos y rehacer integración Firebase/Maps |
| Ezy-Parking | Evaluar servicio de reservas detrás de un contrato propio | Probar funciones descritas, despliegue y autenticación; revisar GPL de la versión |
| Parkingslot | Referencia de investigación, sin incorporación de código | Obtener licencia explícita y revisar datos y modelos |
| Java, P7, Smart Park y seguridad robótica | Conservar como pendientes de identificación | URL, repositorio, versión, licencia y evidencia funcional |

Para TheParker se propone sustituir Firebase por Keycloak, API propia, PostgreSQL y almacenamiento local protegido de archivos. Estas sustituciones requieren cambios de código; no son compatibles automáticamente con las APIs de Firebase. La base inicial puede omitir mapas y presentar zonas y plazas en un plano local. Si se requieren mapas geográficos, evaluar un cliente y datos con licencias adecuadas y una fuente de teselas; no presumir que un servidor público permite uso ilimitado.

## Contratos propuestos

Identificadores estables: `lot_id`, `zone_id`, `slot_id`, `camera_id`, `sensor_id`, `barrier_id`, `reservation_id` y `session_id`. Mantener correspondencias entre sistemas.

Eventos propuestos:
- `parking.slot.observed`: ocupación y calidad, sin información de identidad.
- `parking.reservation.created|cancelled|expired`: emitidos por el servicio transaccional después del commit.
- `parking.session.started|ended`: cambios de sesión, no deducidos únicamente del vídeo.
- `parking.device.health`: estado y frescura de cámara, sensor o controlador.
- `parking.prediction.updated`: horizonte, modelo y estimación separados de lo observado.

Topic propuesto: `parking/v1/{lot_id}/slots/{slot_id}/observed`. No es una interfaz nativa comprobada de los candidatos.

```json
{
  "schema_version": "1.0",
  "event_id": "evt-demo-001",
  "source": "occupancy-adapter",
  "lot_id": "lot-demo",
  "zone_id": "zone-a",
  "slot_id": "slot-001",
  "asset_id": "camera-001",
  "observed_at": "2026-10-13T15:00:00Z",
  "received_at": "2026-10-13T15:00:00.150Z",
  "type": "parking.slot.observed",
  "sequence": 17,
  "payload": {
    "occupancy": "occupied",
    "quality": "valid",
    "confidence": 0.92,
    "model_version": "pilot-001"
  }
}
```

La confianza no garantiza exactitud. Incorporar `free`, `occupied` y `unknown`; configurar caducidad y evitar que mensajes atrasados sobrescriban observaciones más recientes. Identificar una secuencia por origen y su reinicio de sesión; almacenar marcas de observación y recepción con relojes sincronizados.

API propuesta: consultas de instalaciones/plazas, creación y cancelación de reservas, apertura/cierre de sesiones y consulta de eventos. Documentar en OpenAPI antes de implementar. Usar claves de idempotencia y autorización por instalación. Dos reservas del mismo intervalo y plaza deben producir una sola aceptación; validar intervalos en transacción con una restricción de exclusión o mecanismo equivalente, no solamente con una comprobación previa.

## IA: datos, modelos y evaluación

| Función | Referencia inicial | Evaluación requerida |
|---|---|---|
| Ocupación | Sensor o etiquetado manual; comparación visual simple | Precisión/exhaustividad por plaza y condición; confusión libre/ocupado/desconocido; latencia |
| Demanda | Media histórica por franja horaria | MAE/RMSE por horizonte; prueba temporal e instalación distinta |
| Permanencia y anomalías | Reglas de tiempo y calidad de señal | Falsas alertas por hora e instalación; detección en escenarios conocidos |
| Recomendación | Reglas de distancia, tipo de vehículo y reserva | Disponibilidad real y cumplimiento de restricciones; comparación con reglas |

Registrar procedencia y permisos de datos, calibración, versión de etiquetas, divisiones de entrenamiento y prueba, código, artefactos y métricas. Evitar fuga temporal y evaluar cámaras o periodos no usados en entrenamiento. Reservas futuras son información disponible solo según el instante de predicción.

OpenCV apoya preprocesamiento; scikit-learn permite una referencia propia de predicción; ONNX y ONNX Runtime son opcionales cuando el modelo exportado sea compatible. MLflow registra evaluación y promoción. Revisar versiones, formatos, aceleradores y licencias antes de elegir imágenes de despliegue.

La promoción exige licencia del código y derechos suficientes de uso, modificación y distribución de pesos y datos. Un repositorio público sin licencia no entra en la base libre. Entrenar con datos propios autorizados es una ruta propuesta, no una garantía automática de ausencia de restricciones. Registrar deriva, volver a evaluar y permitir reversión a reglas o al modelo previo.

## Seguridad y operación

Separar redes de cámaras, usuarios y servicios; limitar acceso directo a cámaras. Roles propuestos: operador, administrador, auditor y cliente de reserva. Proteger credenciales, limitar consultas y conservar auditoría de cambios y acceso a evidencias. No copiar imágenes o matrículas en los topics generales.

Definir retención, eliminación y copias de seguridad por clase de dato. El piloto no requiere reconocimiento de identidad. Si se añade lectura de matrículas, evaluar finalidad, conservación y acceso; no usar su resultado como autorización única.

El controlador local de barreras conserva sensores antiatrapamiento, interbloqueos, apertura manual y respuesta definida ante fallo. Los modelos solo proponen observaciones o alertas. Las órdenes autorizadas requieren identificador único, expiración, confirmación y pruebas físicas supervisadas.

Ante fallo de cámara: ocupación desconocida. Ante caída de broker: cola local acotada y reproducción; marcar retraso. Ante caída de base de datos: suspender nuevas reservas y aplicar el procedimiento local de continuidad. Restaurar respaldos y probar conciliación después de recuperar servicios.

## Integración con JFXAI4CBS

Usar `lot_id`, `zone_id` y `asset_id` compartidos. Un adaptador publica observaciones y salud de dispositivos hacia los gemelos y la supervisión de JFXAI4CBS. Mantener el contrato de dominio de estacionamiento separado de los eventos de seguridad. No copiar reservas personales, vídeo ni plantillas biométricas en el gemelo general.

Versionar contratos, controlar permisos por instalación y probar lectura/escritura del activo correcto. La integración es propuesta: ni este repositorio ni JFXAI4CBS contienen el conector.

## Etapas y aceptación

| Etapa | Entregable | Criterio |
|---|---|---|
| Catálogo | Identidad, versión, licencias y restricciones | Referencias seleccionadas resueltas; exclusión de pendientes |
| Observación | Una cámara/plazas y contraste con sensor | Métricas por condición; estado desconocido y reconexión correctos |
| Eventos | Adaptador, MQTT y persistencia | Duplicados sin duplicar estado; orden y reproducción comprobados |
| Reservas | API, identidad y transacciones | Concurrencia, expiración, autorización y cancelación probadas |
| IA | Referencias y modelos versionados | Mejora frente a referencia y reversión demostradas |
| Gemelos | Adaptador de Ditto/JFXAI4CBS | Activos correctos, sincronización y frescura verificadas |
| Operación | Respaldo, recuperación y observabilidad | Restauración y carga objetivo probadas |
| Actuación opcional | Controlador de barrera independiente | Interbloqueos, fallo y operación manual verificados |

Fijar antes del piloto los umbrales y escenarios: número de plazas, cámaras, resolución, condiciones ambientales, latencia y retención. No inventar plazos ni rendimiento sin conocer hardware y carga.

## Estructura futura sugerida

```text
catalog/components.json       # presente: registro documental
docs/                        # presente: propuesta y original
schemas/                     # futuro: JSON Schema y OpenAPI
services/occupancy/           # futuro
services/adapters/            # futuro
services/parking/             # futuro
services/prediction/          # futuro
deploy/pilot/                 # futuro: versiones fijadas
tests/integration/            # futuro
tests/model-evaluation/       # futuro
```

Cada incorporación requiere versión fijada, licencia y SBOM, configuración reproducible, pruebas y responsable de actualización. Este diseño conserva capacidad de sustitución sin afirmar interoperabilidad ya implementada.

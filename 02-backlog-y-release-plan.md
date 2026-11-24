# Backlog y Release Plan — Talla Zapatos Android

## 1. Visión del Producto y Releases

| Release | Objetivo de negocio | Sprints | Funcionalidad clave |
|---|---|---|---|
| **Release 1 — MVP** | Usuario mide su pie y obtiene talla por marca sin internet | Sprint 0, 1, 2, 3 | Onboarding, captura A4, detección, conversión 5 marcas, historial básico |
| **Release 2 — Mejora UX** | Retención y confianza en la medición | Sprint 4, 5 | Historial completo, compartir, más marcas (Vans, Reebok), calibración manual mejorada |
| **Release 3 — Escala** | Actualización de tablas y analítica local | Sprint 6 | Importar tablas actualizadas, modo oscuro, métricas de precisión internas |

**Equipo:** 1-2 desarrolladores Android  
**Duración sprint:** 2 semanas  
**Fecha estimada Release 1:** 8 semanas desde kickoff (4 sprints × 2 semanas)

## 2. Configuración del Proceso

**Duración del Sprint:** 2 semanas (10 días hábiles)

**Definition of Ready (DoR):**
- Historia vinculada a un RF del documento de arquitectura
- Criterios de aceptación en formato Given/When/Then (mínimo 2)
- Sin dependencias bloqueantes sin plan de mitigación
- Estimación en story points acordada
- Mockups o descripción de pantalla definida para historias de UI

**Definition of Done (DoD):**
- Código mergeado a `main` con PR revisado
- Tests unitarios para lógica de dominio (si aplica a la historia)
- Sin regresiones en suite de tests existente
- Lint (`detekt` + `ktlint`) sin errores nuevos
- Funcionalidad verificada en emulador API 34 y dispositivo físico
- Criterios de aceptación validados manualmente o con test automatizado

## 3. Backlog por Sprint

### Sprint 0 — Setup e infraestructura

**Objetivo/Hito del sprint:** Proyecto Android compilable con CI, módulos base, OpenCV y CameraX integrados.

**Tareas secuenciales:**
1. Crear proyecto Android con Kotlin, Compose, Hilt, minSdk 26, targetSdk 34
2. Configurar módulos `app`, `domain`, `data` con dependencias Gradle
3. Integrar OpenCV Android SDK 4.9.0 y verificar carga nativa en arm64-v8a
4. Integrar CameraX Preview + ImageCapture básico
5. Configurar GitHub Actions: `./gradlew lint test assembleDebug`
6. Crear esquema Room vacío con entidades `MeasurementSession`, `FootMeasurement`, `BrandSizeRow`
7. Seed `brand_sizes.json` en assets con tablas Nike y Adidas (mínimo viable)

**Historias de Usuario del sprint:**
- **HU-001**: Como desarrollador, quiero un proyecto Android con arquitectura limpia configurada, para construir features de forma estructurada.
  - **Criterios de Aceptación:**
    - Dado un clone del repositorio, cuando ejecuto `./gradlew assembleDebug`, entonces el APK se genera sin errores.
    - Dado el módulo `domain`, cuando ejecuto tests, entonces JUnit 5 corre correctamente.
    - Dado la app instalada, cuando la abro, entonces muestra pantalla placeholder "Talla Zapatos" sin crash.
  - **Estimación:** 5 SP
  - **Prioridad:** Alta
  - **Dependencias:** ninguna

**Métricas del Sprint:**
- Velocity planeado: 5 SP
- Entregable: APK debug instalable + pipeline CI verde

---

### Sprint 1 — Onboarding y captura de cámara

**Objetivo/Hito del sprint:** Usuario completa onboarding y captura foto del pie con guía visual.

**Tareas secuenciales:**
1. Implementar flujo de permisos CAMERA con Accompanist Permissions o API nativa
2. Diseñar pantallas Compose: Splash → Onboarding (3 slides) → Home
3. Implementar `CameraPreviewScreen` con overlay de guía A4 + pie
4. Guardar bitmap capturado en memoria para procesamiento
5. Implementar navegación Navigation Compose entre pantallas

**Historias de Usuario del sprint:**
- **HU-002**: Como usuario nuevo, quiero ver instrucciones claras de cómo usar el A4 y posicionar mi pie, para preparar una buena captura.
  - **Criterios de Aceptación:**
    - Dado que abro la app por primera vez, cuando completo el onboarding de 3 pasos, entonces no vuelve a mostrarse en siguientes aperturas.
    - Dado el último slide, cuando toco "Empezar", entonces navego a la pantalla de captura.
    - Dado que salto el onboarding, cuando toco "Omitir", entonces también llego a captura.
  - **Estimación:** 3 SP
  - **Prioridad:** Alta
  - **Dependencias:** HU-001

- **HU-003**: Como usuario, quiero que la app solicite permiso de cámara de forma clara, para poder tomar la foto de mi pie.
  - **Criterios de Aceptación:**
    - Dado que no concedí permiso, cuando entro a captura, entonces veo diálogo explicando por qué se necesita la cámara.
    - Dado que deniego el permiso, cuando intento capturar, entonces veo mensaje con enlace a configuración del sistema.
    - Dado que concedo permiso, cuando entro a captura, entonces veo preview en vivo de la cámara.
  - **Estimación:** 3 SP
  - **Prioridad:** Alta
  - **Dependencias:** HU-001

- **HU-004**: Como usuario, quiero capturar una foto de mi pie sobre el A4 con una guía visual en pantalla, para encuadrar correctamente la toma.
  - **Criterios de Aceptación:**
    - Dado el preview activo, cuando veo la pantalla, entonces hay overlay semitransparente con contorno de A4 y zona de pie.
    - Dado el preview activo, cuando toco el botón capturar, entonces se congela la imagen y avanzo a procesamiento.
    - Dado mala iluminación, cuando la captura es borrosa, entonces veo advertencia "Mantén el teléfono estable y buena luz".
  - **Estimación:** 5 SP
  - **Prioridad:** Alta
  - **Dependencias:** HU-003

**Métricas del Sprint:**
- Velocity planeado: 11 SP
- Entregable: Flujo onboarding → captura foto funcional en dispositivo físico

---

### Sprint 2 — Motor de visión y conversión de tallas

**Objetivo/Hito del sprint:** Detección automática A4 + pie y conversión a tallas EU/US/UK.

**Tareas secuenciales:**
1. Implementar `OpenCvFootDetector.detectA4Corners()` con Canny + findContours + approxPolyDP
2. Calcular matriz de perspectiva y escala mm/px (A4 = 210×297 mm)
3. Implementar segmentación de pie por umbral adaptativo en región interior al A4
4. Implementar `ConvertToBrandSizeUseCase` con interpolación por longitud mm
5. Completar seed JSON para Puma, Converse, New Balance
6. Pantalla de resultados con selector de marca (dropdown)

**Historias de Usuario del sprint:**
- **HU-005**: Como usuario, quiero que la app detecte automáticamente el A4 y mida mi pie, para obtener longitud y anchura sin usar regla.
  - **Criterios de Aceptación:**
    - Dado una foto con A4 visible y pie centrado, cuando procesa, entonces muestra longitud en mm con error ≤ 3 mm vs. medición manual de referencia.
    - Dado que no detecta A4, cuando falla detección, entonces muestra pantalla de ajuste manual de 4 esquinas.
    - Dado procesamiento exitoso, cuando termina, entonces el tiempo total es menor a 3 segundos.
  - **Estimación:** 8 SP
  - **Prioridad:** Alta
  - **Dependencias:** HU-004

- **HU-006**: Como usuario, quiero ajustar manualmente las esquinas del A4 si la detección falla, para salvar capturas difíciles.
  - **Criterios de Aceptación:**
    - Dado la pantalla de ajuste, cuando arrastro cada esquina, entonces el cuadrilátero se actualiza en tiempo real.
    - Dado esquinas ajustadas, cuando confirmo, entonces recalcula medidas con la escala corregida.
    - Dado esquinas inválidas (auto-intersección), cuando confirmo, entonces muestra error y no avanza.
  - **Estimación:** 5 SP
  - **Prioridad:** Alta
  - **Dependencias:** HU-005

- **HU-007**: Como usuario, quiero ver mi talla en EU, US y UK para la marca que voy a comprar, para usar la equivalencia correcta en la tienda.
  - **Criterios de Aceptación:**
    - Dado medidas calculadas, cuando selecciono "Nike", entonces veo talla EU, US y UK según tabla Nike.
    - Dado cambio de marca, cuando selecciono "Adidas", entonces las tallas se actualizan sin recapturar.
    - Dado longitud fuera de rango tabulado, cuando convierte, entonces muestra "Fuera de rango — consulta tabla de la marca".
  - **Estimación:** 5 SP
  - **Prioridad:** Alta
  - **Dependencias:** HU-005

**Métricas del Sprint:**
- Velocity planeado: 18 SP
- Entregable: Captura → medición mm → talla por marca en pantalla de resultados

---

### Sprint 3 — Sesión bilateral, historial y release MVP

**Objetivo/Hito del sprint:** Flujo completo punta a punta con ambos pies, historial y pulido para Release 1.

**Tareas secuenciales:**
1. Flujo medir pie izquierdo → pie derecho → recomendación final (pie más largo)
2. Implementar `MeasurementRepository` con Room y pantalla Historial
3. Pantalla resultado final con resumen y botón "Nueva medición"
4. Manejo offline explícito (sin llamadas de red)
5. Pulido UI: tipografía, colores Material 3, estados de carga/error
6. Pruebas E2E del flujo completo en dispositivo físico
7. Generar APK release firmado y documentar instalación

**Historias de Usuario del sprint:**
- **HU-008**: Como usuario, quiero medir ambos pies en una sesión, para que la app recomiende la talla según mi pie más largo.
  - **Criterios de Aceptación:**
    - Dado que completé pie izquierdo, cuando avanzo, entonces la app solicita captura del pie derecho.
    - Dado ambos pies medidos, cuando veo resultado final, entonces indica cuál pie es más largo y la talla basada en ese pie.
    - Dado que solo quiero medir un pie, cuando selecciono "Solo un pie", entonces salta la segunda captura.
  - **Estimación:** 5 SP
  - **Prioridad:** Alta
  - **Dependencias:** HU-007

- **HU-009**: Como usuario, quiero guardar mi medición en historial local, para consultarla después sin repetir el proceso.
  - **Criterios de Aceptación:**
    - Dado resultado final, cuando toco "Guardar", entonces la sesión se persiste en Room con fecha y tallas.
    - Dado historial con entradas, cuando abro "Historial", entonces veo lista ordenada por fecha descendente.
    - Dado una entrada, cuando la selecciono, entonces veo detalle con medidas y tallas por marca.
  - **Estimación:** 5 SP
  - **Prioridad:** Alta
  - **Dependencias:** HU-008

- **HU-010**: Como usuario, quiero repetir la captura si el resultado no me convence, para corregir sin perder la sesión.
  - **Criterios de Aceptación:**
    - Dado pantalla de resultados, cuando toco "Repetir captura", entonces regreso a cámara manteniendo el pie seleccionado (izq/der).
    - Dado repetición, cuando capturo de nuevo, entonces reemplaza la medición anterior de ese pie en la sesión actual.
    - Dado repetición exitosa, cuando veo resultados, entonces las tallas reflejan la nueva medición.
  - **Estimación:** 3 SP
  - **Prioridad:** Alta
  - **Dependencias:** HU-008

- **HU-011**: Como usuario, quiero usar la app sin conexión a internet, para medir en cualquier lugar.
  - **Criterios de Aceptación:**
    - Dado modo avión activado, cuando uso el flujo completo, entonces todas las funciones MVP operan sin error.
    - Dado sin red, cuando abro historial, entonces veo datos guardados previamente.
    - Dado instalación inicial sin red, cuando abro la app, entonces tablas de marcas están disponibles desde assets locales.
  - **Estimación:** 2 SP
  - **Prioridad:** Alta
  - **Dependencias:** HU-009

**Métricas del Sprint:**
- Velocity planeado: 15 SP
- Entregable: **Release 1 MVP** — APK release con flujo completo offline

---

### Sprint 4 — Historial avanzado y compartir (Release 2)

**Objetivo/Hito del sprint:** Mejorar retención con compartir y UX de historial.

**Historias de Usuario del sprint:**
- **HU-012**: Como usuario, quiero compartir mi resultado de talla por WhatsApp o correo, para enviárselo a alguien que me compre zapatos.
  - **Criterios de Aceptación:**
    - Dado resultado guardado, cuando toco "Compartir", entonces se abre selector de apps con texto formateado (marca, EU, US, UK, fecha).
    - Dado compartir, cuando elijo WhatsApp, entonces el mensaje incluye medidas en mm y tallas.
    - Dado sin apps de compartir, cuando toco compartir, entonces muestra toast informativo.
  - **Estimación:** 3 SP
  - **Prioridad:** Media
  - **Dependencias:** HU-009

**Métricas del Sprint:**
- Velocity planeado: 3 SP
- Entregable: Intent ACTION_SEND funcional

---

### Sprint 5 — Marcas adicionales (Release 2)

**Objetivo/Hito del sprint:** Ampliar catálogo a Vans y Reebok.

**Historias de Usuario del sprint:**
- **HU-013**: Como usuario, quiero ver tallas de Vans y Reebok, para cubrir más tiendas online.
  - **Criterios de Aceptación:**
    - Dado selector de marcas, cuando despliego lista, entonces incluye Vans y Reebok además de las 5 originales.
    - Dado selección Vans, cuando convierte, entonces aplica tabla específica Vans con equivalencias correctas para pie de 260 mm (validación: EU 41).
    - Dado actualización de seed, cuando instalo nueva versión, entonces Room migra/actualiza filas sin perder historial.
  - **Estimación:** 3 SP
  - **Prioridad:** Media
  - **Dependencias:** HU-007

**Métricas del Sprint:**
- Velocity planeado: 3 SP
- Entregable: 7 marcas disponibles

---

### Sprint 6 — Modo oscuro y actualización de tablas (Release 3)

**Objetivo/Hito del sprint:** Pulido visual y mecanismo de actualización de datos.

**Historias de Usuario del sprint:**
- **HU-014**: Como usuario, quiero modo oscuro, para usar la app de noche sin molestar.
  - **Criterios de Aceptación:**
    - Dado configuración del sistema en oscuro, cuando abro la app, entonces usa tema Material 3 dark.
    - Dado preferencia manual en ajustes, cuando elijo "Claro", entonces ignora configuración del sistema.
    - Dado modo oscuro, cuando navego todo el flujo, entonces contraste cumple 4.5:1 en textos principales.
  - **Estimación:** 2 SP
  - **Prioridad:** Baja
  - **Dependencias:** HU-011

**Métricas del Sprint:**
- Velocity planeado: 2 SP
- Entregable: Release 3 publicable

## 4. Resumen de Release Plan

| Release | Sprints | Fecha estimada | Funcionalidad entregada | Hito de negocio |
|---|---|---|---|---|
| Release 1 MVP | Sprint 0–3 | Semana 8 | Medición completa offline, 5 marcas, historial | Primera versión usable en Play Store (beta cerrada) |
| Release 2 | Sprint 4–5 | Semana 12 | Compartir, 7 marcas | Mejora retención y cobertura de marcas |
| Release 3 | Sprint 6 | Semana 14 | Modo oscuro, tablas actualizables | Pulido pre-lanzamiento público |

## 5. Backlog priorizado (vista tabular consolidada)

| ID | Historia | Sprint | Prioridad | Story Points | Estado |
|---|---|---|---|---|---|
| HU-001 | Proyecto Android con arquitectura limpia configurada | 0 | Alta | 5 | Pendiente |
| HU-002 | Onboarding con instrucciones A4 y pie | 1 | Alta | 3 | Pendiente |
| HU-003 | Solicitud permiso de cámara | 1 | Alta | 3 | Pendiente |
| HU-004 | Captura con guía visual overlay | 1 | Alta | 5 | Pendiente |
| HU-005 | Detección automática A4 y medición de pie | 2 | Alta | 8 | Pendiente |
| HU-006 | Ajuste manual de esquinas A4 | 2 | Alta | 5 | Pendiente |
| HU-007 | Conversión a tallas EU/US/UK por marca | 2 | Alta | 5 | Pendiente |
| HU-008 | Sesión bilateral con recomendación pie más largo | 3 | Alta | 5 | Pendiente |
| HU-009 | Guardar y consultar historial local | 3 | Alta | 5 | Pendiente |
| HU-010 | Repetir captura sin perder sesión | 3 | Alta | 3 | Pendiente |
| HU-011 | Funcionamiento 100% offline | 3 | Alta | 2 | Pendiente |
| HU-012 | Compartir resultado vía intent Android | 4 | Media | 3 | Pendiente |
| HU-013 | Marcas Vans y Reebok | 5 | Media | 3 | Pendiente |
| HU-014 | Modo oscuro | 6 | Baja | 2 | Pendiente |

**Total story points:** 57 SP

## 6. Riesgos y Dependencias Técnicas

| Riesgo | Probabilidad | Impacto | Mitigación |
|---|---|---|---|
| Detección A4 falla con fondos no contrastantes | Alta | Alto | Ajuste manual HU-006; onboarding enfatiza fondo oscuro uniforme |
| Variabilidad de tallas entre modelos de una misma marca | Media | Medio | Disclaimer en UI; tablas basadas en guías oficiales 2025 |
| OpenCV aumenta tamaño APK | Media | Medio | Empaquetar solo arm64-v8a en release; ABI splits en Gradle |
| Diferencia entre talla medida y preferencia de calce | Alta | Medio | Mostrar rango ±0.5 talla; nota "considera media talla si prefieres calce holgado" |
| Fragmentación de cámaras en gama baja | Media | Alto | Probar en 3 dispositivos físicos; fallback resolución 1080p |
| Precisión insuficiente en pies con calcetín grueso | Media | Medio | Onboarding recomienda pie descalzo o calcetín fino |

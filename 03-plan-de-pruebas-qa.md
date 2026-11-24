# Plan de Pruebas y Estrategia de QA — Talla Zapatos Android

## 1. Objetivo del Plan de Pruebas

Validar que la aplicación **Talla Zapatos Android** cumple los requisitos funcionales y no funcionales definidos, con énfasis en la precisión de medición (±3 mm), el flujo offline completo y la conversión correcta de tallas por marca. El plan asegura trazabilidad desde RF → HU → TC para el Release 1 MVP.

## 2. Alcance — ¿Qué se va a probar?

**Dentro de alcance (Release 1):**
- Onboarding y permisos de cámara
- Captura con CameraX y overlay de guía
- Motor de visión OpenCV (detección A4, contorno de pie)
- Ajuste manual de esquinas
- Conversión de medidas a tallas (5 marcas)
- Sesión bilateral y recomendación
- Persistencia Room e historial
- Funcionamiento offline
- Flujo E2E completo en dispositivo físico

**Fuera de alcance (Release 1):**
- Compartir resultado (HU-012, Release 2)
- Marcas Vans/Reebok (HU-013, Release 2)
- Modo oscuro (HU-014, Release 3)
- Pruebas de carga (no aplica app local)
- Pruebas en iOS

## 3. Estrategia de Pruebas — Niveles

| Nivel | Objetivo | Responsable | Cobertura esperada |
|---|---|---|---|
| Pruebas Unitarias | Validar use cases de dominio (conversión, cálculo escala, selección pie más largo) | Dev Android | ≥ 80% en módulo `domain` |
| Pruebas de Integración | Validar Room DAO, seed JSON, pipeline OpenCV con imágenes fixture | Dev Android | 100% repositorios y detector con 10 imágenes golden |
| Pruebas de Sistema (E2E) | Validar flujos completos en emulador y dispositivo físico | Dev Android / QA manual | 100% historias Must del Release 1 |
| Pruebas de Estrés/Carga | No aplica MVP local | — | N/A |
| Pruebas de Seguridad | Verificar que imágenes no se escriben a almacenamiento externo sin consentimiento | Dev Android | Permisos y almacenamiento revisados |
| Pruebas de Regresión | Suite automatizada en CI ante cada PR | CI GitHub Actions | Unit + integración en cada merge |

## 4. Herramientas de QA

| Tipo de prueba | Herramienta | Justificación |
|---|---|---|
| Unitarias | JUnit 5 + MockK | Estándar Kotlin; mocking de repositorios |
| Integración Room | AndroidX Test + Room in-memory DB | Tests rápidos sin emulador |
| Integración OpenCV | Instrumented tests con imágenes en `androidTest/assets/` | Requiere librerías nativas |
| UI Compose | Compose UI Test + `createComposeRule` | Tests declarativos de pantallas |
| E2E | Espresso + UI Automator (permisos sistema) | Flujos con diálogos nativos Android |
| Lint estático | detekt + ktlint | Calidad de código Kotlin |
| CI | GitHub Actions | `./gradlew test connectedCheck` en PR |
| Medición precisión | Script Python comparativo + 20 fotos de referencia con regla | Validación RNF-002 offline del pipeline |

## 5. Ambientes de Prueba

| Ambiente | Descripción | Datos de prueba |
|---|---|---|
| Local dev | Android Studio emulador Pixel 6 API 34 | Imágenes fixture `test_a4_foot_*.jpg` en assets de test |
| Dispositivo físico | Mínimo 2 dispositivos: gama media (Snapdragon 6xx) y gama alta | Pie real + A4 + regla para validación ±3 mm |
| CI | GitHub Actions ubuntu-latest + emulador API 30 | Tests unitarios + instrumented headless |
| Staging | No aplica (sin backend) | — |
| Producción | Play Store beta cerrada (Release 1) | Usuarios internos del equipo |

**Datos de prueba requeridos:**
- 10 imágenes golden con medidas conocidas (longitud pie en mm anotada)
- 3 imágenes con A4 parcialmente oculto (casos de error)
- 1 imagen sin A4 visible (fallo detección)
- Seed `brand_sizes.json` con fila conocida: 260 mm → Nike EU 41

## 6. Casos de Prueba (Test Cases)

| TC-ID | HU relacionada | Descripción | Precondición | Pasos | Resultado Esperado | Tipo | Prioridad |
|---|---|---|---|---|---|---|---|
| TC-001 | HU-001 | Build debug exitoso | Repo clonado, JDK 17 | 1. Ejecutar `./gradlew assembleDebug` | APK generado en `app/build/outputs/apk/debug/` | Integración | Alta |
| TC-002 | HU-001 | Tests unitarios domain pasan | Proyecto configurado | 1. Ejecutar `./gradlew :domain:test` | 0 fallos | Unit | Alta |
| TC-003 | HU-002 | Onboarding primera vez | App instalada, primera apertura | 1. Abrir app 2. Avanzar 3 slides 3. Tocar Empezar | Llega a pantalla captura; onboarding no reaparece al reabrir | Sistema | Alta |
| TC-004 | HU-002 | Omitir onboarding | Primera apertura | 1. Abrir app 2. Tocar Omitir en slide 1 | Navega a captura directamente | Sistema | Media |
| TC-005 | HU-003 | Permiso cámara concedido | Permiso no concedido | 1. Ir a captura 2. Aceptar permiso | Preview de cámara visible | Sistema | Alta |
| TC-006 | HU-003 | Permiso cámara denegado | Permiso denegado permanentemente | 1. Ir a captura 2. Denegar 2 veces | Mensaje con enlace a configuración | Sistema | Alta |
| TC-007 | HU-004 | Captura con overlay visible | Permiso concedido | 1. Abrir captura | Overlay A4 y zona de pie visibles sobre preview | Sistema | Alta |
| TC-008 | HU-004 | Captura congela frame | Preview activo | 1. Tocar botón capturar | Imagen congelada, avanza a procesamiento | Sistema | Alta |
| TC-009 | HU-005 | Detección A4 exitosa | Imagen fixture `test_a4_foot_ok.jpg` (pie 260 mm) | 1. Procesar imagen en test instrumentado | longitudMm entre 257 y 263 | Integración | Alta |
| TC-010 | HU-005 | Fallo detección A4 | Imagen `test_no_a4.jpg` | 1. Procesar imagen | Retorna error DETECTION_FAILED, navega a ajuste manual | Integración | Alta |
| TC-011 | HU-005 | Tiempo procesamiento < 3s | Dispositivo Snapdragon 695 | 1. Capturar y procesar 2. Medir tiempo | Elapsed < 3000 ms | Sistema | Alta |
| TC-012 | HU-006 | Ajuste manual 4 esquinas | Pantalla ajuste tras fallo | 1. Arrastrar cada esquina 2. Confirmar | Recalcula medidas con nuevas esquinas | Sistema | Alta |
| TC-013 | HU-006 | Esquinas inválidas rechazadas | Pantalla ajuste | 1. Crear auto-intersección 2. Confirmar | Error visible, no avanza | Sistema | Media |
| TC-014 | HU-007 | Conversión Nike 260mm | FootMeasurement lengthMm=260 | 1. Ejecutar ConvertToBrandSizeUseCase(Nike) | sizeEU=41, sizeUS=8, sizeUK=7 (±0 según tabla) | Unit | Alta |
| TC-015 | HU-007 | Cambio de marca actualiza talla | Resultado con Nike mostrado | 1. Cambiar selector a Adidas | Tallas actualizadas sin recaptura | Sistema | Alta |
| TC-016 | HU-007 | Longitud fuera de rango | lengthMm=400 | 1. Convertir cualquier marca | Mensaje "Fuera de rango" | Unit | Media |
| TC-017 | HU-008 | Flujo bilateral completo | Captura izquierdo completada | 1. Medir pie izq 2. Medir pie der 3. Ver final | Indica pie más largo y talla basada en él | Sistema | Alta |
| TC-018 | HU-008 | Modo un solo pie | Pantalla post pie izquierdo | 1. Seleccionar "Solo un pie" | Salta captura derecho, muestra resultado con un pie | Sistema | Media |
| TC-019 | HU-009 | Guardar sesión en Room | Resultado final visible | 1. Tocar Guardar | Registro en DB con fecha y tallas | Integración | Alta |
| TC-020 | HU-009 | Listar historial | ≥1 sesión guardada | 1. Abrir Historial | Lista ordenada por fecha descendente | Sistema | Alta |
| TC-021 | HU-009 | Detalle de entrada historial | Historial con entradas | 1. Tocar una entrada | Muestra medidas mm y tallas por marca | Sistema | Alta |
| TC-022 | HU-010 | Repetir captura pie izquierdo | Sesión con pie izq medido | 1. Tocar Repetir 2. Recapturar | Nueva medición reemplaza anterior en sesión | Sistema | Alta |
| TC-023 | HU-011 | Flujo completo en modo avión | Modo avión ON, app instalada | 1. Onboarding 2. Capturar 3. Ver resultado 4. Guardar 5. Ver historial | Todo funciona sin error de red | Sistema | Alta |
| TC-024 | HU-011 | Tablas locales sin red | Instalación fresca sin red | 1. Instalar APK 2. Abrir 3. Llegar a selector marcas | 5 marcas disponibles | Sistema | Alta |
| TC-025 | HU-005 | Precisión 90% dataset | 20 fotos con medida regla conocida | 1. Procesar cada foto 2. Comparar mm | ≥18 de 20 dentro de ±3 mm | Integración | Alta |
| TC-026 | HU-004 | Advertencia foto borrosa | Imagen con varianza Laplacian < umbral | 1. Capturar imagen borrosa | Muestra advertencia de luz/estabilidad | Sistema | Media |
| TC-027 | HU-007 | Conversión 5 marcas seed completo | Seed JSON cargado | 1. Para cada marca, convertir 270mm | Retorna talla válida no vacía | Unit | Alta |
| TC-028 | HU-008 | Pie derecho más largo | Izq=255mm, Der=262mm | 1. Completar sesión bilateral | Recomendación basada en pie derecho 262mm | Unit | Alta |
| TC-029 | HU-009 | Historial vacío | Sin sesiones guardadas | 1. Abrir Historial | Mensaje "Sin mediciones aún" | Sistema | Baja |
| TC-030 | HU-001 | CI pipeline verde | PR abierto | 1. Push a rama 2. Esperar Actions | lint + test pasan | Integración | Alta |

## 7. Matriz de Trazabilidad

| Requisito (RF-ID) | Historia (HU-ID) | Caso(s) de Prueba (TC-ID) | Cubierto |
|---|---|---|---|
| RF-001 | HU-002 | TC-003, TC-004 | Sí |
| RF-002 | HU-003 | TC-005, TC-006 | Sí |
| RF-003 | HU-004 | TC-007, TC-008, TC-026 | Sí |
| RF-004 | HU-005 | TC-009, TC-010, TC-025 | Sí |
| RF-005 | HU-005 | TC-009, TC-025 | Sí |
| RF-006 | HU-006 | TC-012, TC-013 | Sí |
| RF-007 | HU-007 | TC-014, TC-016, TC-027 | Sí |
| RF-008 | HU-007 | TC-015, TC-027 | Sí |
| RF-009 | HU-008 | TC-017, TC-018, TC-028 | Sí |
| RF-010 | HU-008 | TC-017, TC-028 | Sí |
| RF-011 | HU-009 | TC-019, TC-020 | Sí |
| RF-012 | HU-009 | TC-020, TC-021, TC-029 | Sí |
| RF-013 | HU-012 | — | No (Release 2) |
| RF-014 | HU-011 | TC-023, TC-024 | Sí |
| RF-015 | HU-010 | TC-022 | Sí |
| RNF-001 | HU-005 | TC-011 | Sí |
| RNF-002 | HU-005 | TC-025 | Sí |
| RNF-008 | HU-001 | TC-002, TC-030 | Sí |
| RNF-009 | HU-011 | TC-023 | Sí |

## 8. Criterios de Salida (Exit Criteria)

**Release 1 MVP listo para beta cerrada cuando:**

1. **0 bugs críticos o bloqueantes** abiertos (crash en flujo principal, medición siempre falla, pérdida de datos historial)
2. **100% casos TC de prioridad Alta** del Release 1 en estado PASS
3. **Cobertura unitaria ≥ 80%** en módulo `domain` verificada por reporte JaCoCo
4. **Precisión RNF-002 validada:** TC-025 PASS (≥18/20 imágenes dentro de ±3 mm)
5. **Flujo E2E TC-023** (modo avión) PASS en 2 dispositivos físicos distintos
6. **APK release ≤ 45 MB** verificado
7. **Pipeline CI TC-030** verde en commit de tag de release
8. Máximo **3 bugs menores** abiertos documentados con workaround

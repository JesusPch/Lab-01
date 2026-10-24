# Documentación Técnica: Protección, Gobernanza y Seguridad de Datos en PostgreSQL

Este documento contiene las definiciones, criterios de seguridad, análisis de técnicas de protección de información (PII) y políticas de retención y control de acceso estructuradas para la base de datos.

---

## 1. Clasificación de Datos y Justificación de Protección

Todos los campos de la tabla `clientes` requieren medidas de seguridad diferenciadas según su naturaleza e impacto en la privacidad:

* **Identificadores Directos (`nombre`, `correo`, `telefono`, `rfc`):**  
  Requieren protección estricta porque contienen Información de Identificación Personal (PII) que vincula directamente los registros con una persona física específica. Su exposición no autorizada vulnera la privacidad del titular y genera riesgos directos de suplantación de identidad o contacto no deseado.

* **Cuasi-identificadores o Identificadores Indirectos (`edad`, `ciudad`, `codigo_posta`, `fecha_alta`, `id`):**  
  Requieren protección porque, al cruzarse entre sí o combinarse con bases de datos externas (censos, padrones, redes sociales), permiten reducir el espectro poblacional y reidentificar al individuo mediante técnicas de vinculación (*linkage attacks*).

---

## 2. Análisis del Enmascaramiento de Datos

* **Qué se ocultó:**  
  Se ocultaron de forma parcial los valores sensibles de los campos de contacto y tributarios:
  * **`correo`:** Se reemplazaron los caracteres centrales del usuario por asteriscos (`*****`), conservando solo las dos primeras letras y el dominio (`an*****@example.test`).
  * **`telefono`:** Se ocultaron los primeros 6 dígitos mediante asteriscos (`******`), dejando legibles únicamente los últimos 4 dígitos (`******4567`).
  * **`rfc`:** Se enmascararon los primeros 9 caracteres (iniciales y fecha de nacimiento), manteniendo visibles únicamente los 4 caracteres finales de la homoclave (`*********5ABC`).
* **Para quién:**  
  Exclusivamente para los usuarios asignados al rol operativo de **`soporte`**.
* **Por qué:**  
  Para aplicar el principio de necesidad de saber (*Need-to-Know*). El área de soporte requiere validar la identidad o el registro del cliente sin necesidad de visualizar la totalidad de la PII, reduciendo la exposición innecesaria de datos sensibles ante filtraciones o inspecciones de pantalla.
* **Mecanismo utilizado:**  
  Se utilizó **enmascaramiento dinámico a nivel de consulta** mediante la creación de una vista (`CREATE VIEW v_clientes_soporte`), construida con funciones de manipulación de cadenas en SQL (`CONCAT`, `LEFT`, `RIGHT`, `SUBSTRING` y `POSITION`).

---

## 3. Seudonimización y Mapa de Correspondencia

* **Cómo se generaron los tokens:**  
  Los tokens se generaron mediante la función `gen_random_uuid()` provista por la extensión `pgcrypto` de PostgreSQL, la cual asigna un Identificador Único Universal (UUID) aleatorio a cada cliente de la tabla base.
* **Dónde se encuentra el mapa de correspondencia:**  
  Se ubica en la tabla `mapa_cliente` dentro del esquema aislado `pii` (`pii.mapa_cliente`), relacionando el `token` (UUID) con el `cliente_id` original.
* **Quién debería poder acceder a él:**  
  Exclusivamente los administradores de seguridad de la base de datos, Oficiales de Protección de Datos (DPO) o procesos del sistema autorizados que requieran reidentificar al usuario con fines legítimos.
* **Por qué se separaron los permisos:**  
  Para aplicar la separación de funciones (*Separation of Duties - SoD*) y el principio de mínimo privilegio. Al dar acceso al rol `analista` únicamente sobre el esquema `analitica` y denegarlo explícitamente sobre los esquemas `pii` y `public`, se impide que el analista pueda revertir los tokens para identificar a las personas físicas.

---

## 4. Anonimización y Generalización de Datos

* **Qué identificadores fueron eliminados:**  
  Se removieron de los reportes los identificadores directos como `nombre`, `correo`, `telefono`, `rfc` y la clave primaria original `id`.
* **Qué atributos fueron generalizados:**  
  * **`edad`:** Convertida en rangos o intervalos categóricos (`18-30`, `31-45`, `46+`).
  * **`codigo_posta`:** Reducido a la región general mediante el truncado de los primeros dos dígitos (`LEFT(codigo_posta, 2)`).
* **Qué grupos fueron suprimidos:**  
  Aquellos grupos cuya frecuencia calculada con `COUNT(*)` no cumplió con un umbral mínimo de 5 registros (`HAVING COUNT(*) >= 5`), aplicando el criterio de $k$-anonimato ($k=5$).
* **Riesgo de reidentificación permanente:**  
  Permanece un riesgo residual por ataques de vinculación externa o inferencia si un atacante cruza las variables generalizadas (región + rango de edad) con bases de datos públicas o censos externos que contengan pocas personas en dicha demarcación.

---

## 5. Política de Retención de Datos

* **Qué criterio se utilizó:**  
  Un criterio temporal de antigüedad de registro, identificando aquellos clientes cuyo tiempo de registro (`fecha_alta`) superaba un intervalo de 5 años (`WHERE fecha_alta < CURRENT_DATE - INTERVAL '5 years'`).
* **Qué información debería conservarse:**  
  Datos agregados, métricas históricas de rendimiento empresarial o registros fiscales/contables cuyo periodo de conservación obligatoria siga vigente por ley.
* **Qué información podría eliminarse:**  
  Los datos personales identificables (PII) directos e indirectos de clientes que hayan terminado su relación contractual y hayan cumplido su plazo de retención legal.
* **Controles previos requeridos antes de eliminar:**  
  1. Verificación de ausencia de requerimientos judiciales, auditorías o procedimientos fiscales pendientes.
  2. Generación de bitácoras y registros de auditoría que documenten la purga.
  3. Aprobación y firma del Oficial de Privacidad / DPO.
  4. Respaldo previo de seguridad de la estructura y ejecución de la purga bajo transacciones de base de datos seguras.

---

## 6. Síntesis de Aplicación Técnica y Control de Acceso

| Escenario / Área | Técnica Recomendada | Justificación y Aplicación |
| :--- | :--- | :--- |
| **Soporte Operativo** | Enmascaramiento de datos (*Vistas Dinámicas*) | Permite al operador confirmar la identidad o realizar la atención mostrando únicamente caracteres parciales (ej. los últimos 4 dígitos del teléfono o del RFC). |
| **Analítica de Datos** | Seudonimización | Sustituye los identificadores directos por tokens (UUID) y restringe la lectura al esquema analítico, permitiendo realizar modelos y cruces sin exponer datos nominales. |
| **Estadísticas Públicas** | Anonimización y Agregación ($k$-anonimato) | Agrupa variables en rangos, elimina identificadores y suprime grupos pequeños para que la información publicada sea irreversiblemente anónima. |

### ¿Por qué ninguna de estas técnicas sustituye al control de acceso?
Porque las técnicas de transformación (enmascaramiento, seudonimización y anonimización) protegen el **contenido de los datos**, pero no la **autorización para acceder a ellos**. Sin un Control de Acceso Basado en Roles (RBAC), cualquier usuario no autorizado podría saltearse las vistas, consultar directamente la tabla base `public.clientes` o leer el esquema `pii`. El control de acceso es la barrera fundamental que dictamina quién puede ejecutar qué consultas dentro de la infraestructura.
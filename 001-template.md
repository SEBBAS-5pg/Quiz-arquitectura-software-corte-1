
# ADR-001: Refactorización del módulo de autenticación por fallas críticas de seguridad y problemas de mantenibilidad

## Contexto

El sistema actual implementa un módulo de autenticación simple que ofrece login y registro de usuarios mediante endpoints REST en Spring Boot. Durante la auditoría técnica realizada en la Fase 3 del proyecto, se identificaron múltiples vulnerabilidades graves que comprometen directamente la seguridad del sistema y la calidad del código. Entre los hallazgos destacan: exposición del hash MD5 de la contraseña en la respuesta del login, uso de MD5 sin salt para almacenamiento de contraseñas, SQL Injection debido al uso de `Statement` con concatenación de strings, validación débil en el registro de usuarios, manejo incorrecto de códigos HTTP, campos públicos en el modelo `User` sin encapsulación y logging de información sensible.  
Estos problemas afectan tanto a los usuarios (riesgo de robo de credenciales, manipulación de datos y suplantación) como al equipo de desarrollo (alta deuda técnica, baja mantenibilidad y dificultad para integrar nuevas funcionalidades). La combinación de vulnerabilidades hace urgente tomar decisiones de diseño para corregir el módulo antes de continuar el desarrollo.

***

## Decisión

Para abordar las fallas encontradas, se decidió realizar una **refactorización parcial pero estructurada** del módulo de autenticación, siguiendo principios SOLID, Clean Code y buenas prácticas de seguridad.

### 1. Eliminar SQL Injection usando PreparedStatement

Se reemplazarán todos los `Statement` por `PreparedStatement` (o `JdbcTemplate`) para impedir la manipulación de consultas. Esto responde directamente a la vulnerabilidad crítica demostrada en la Fase 3.

### 2. Sustituir MD5 por BCrypt para almacenar contraseñas

Se implementará `BCryptPasswordEncoder`, que provee hashing seguro con salt automático. Esto elimina la exposición de hashes predecibles y alinea el sistema con estándares modernos.

### 3. Mejorar la estructura del modelo y controladores mediante encapsulación y DTOs

Los campos del modelo `User` serán privados, y el controlador usará DTOs con validaciones `@Valid` para evitar parámetros ambiguos (`u`, `p`, `e`). Esto mejora legibilidad, robustez y reduce errores en el cliente.

### 4. Ajustar correctamente códigos HTTP según REST

El login fallido retornará **401 Unauthorized** y los errores de validación **400 Bad Request**, evitando el uso incorrecto de `200 OK`.

### 5. Introducir una interfaz para el repositorio (DIP)

Se creará `IUserRepository` para desacoplar `AuthService` de implementaciones concretas, facilitando pruebas unitarias y posibles migraciones a JPA.

Estas decisiones mejoran seguridad, claridad, mantenibilidad y permiten escalar el sistema sin acumular deuda técnica adicional.

***

## Consecuencias

### Consecuencias positivas

*   Eliminación real de la vulnerabilidad de SQL Injection.
*   Almacenamiento seguro de contraseñas mediante hashing robusto (BCrypt).
*   Código más limpio, legible y mantenible gracias a DTOs, encapsulación y nombres claros.
*   Comportamiento consistente con principios REST en el manejo de respuestas HTTP.
*   Menos exposición de datos sensibles en respuestas y logs.
*   Arquitectura más modular y testeable al introducir una interfaz de repositorio.

### Consecuencias negativas o riesgos

*   Requiere tiempo adicional de desarrollo y revisión por pares.
*   Puede romper integraciones existentes si clientes esperaban las respuestas antiguas.
*   Necesidad de migrar contraseñas ya guardadas en MD5 a un formato más seguro.
*   Introducción de nuevas dependencias y mayor complejidad inicial en la arquitectura.

***

## Alternativas consideradas

### 1. Reescribir todo el módulo de autenticación desde cero

Descartado porque implicaría mayor tiempo y riesgo, además de no ser necesario dado que el código actual es refactorizable. Añadiría esfuerzo sin mejorar la eficiencia del entregable académico.

### 2. Solo aplicar parches críticos de seguridad sin refactorizar nada más

Descartado porque resolvería el SQL Injection y el hashing débil, pero dejaría intactos problemas graves de diseño: nombres inadecuados, campos públicos, validaciones pobres, manejo incorrecto de HTTP y fuerte dependencia entre capas. Esto generaría deuda técnica futura aún más costosa.

### 3. Migrar completamente a Spring Security con autenticación JWT

Considerado, pero descartado para esta fase ya que ampliaría el alcance más allá del objetivo del quiz. Se podría proponer en un futuro ADR como mejora incremental.
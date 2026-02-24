| #   | Descripción del problema                                               | Archivo                 | Línea aprox.            | Principio violado                                       | Riesgo     |
| --- | ---------------------------------------------------------------------- | ----------------------- | ----------------------- | ------------------------------------------------------- | ---------- |
| 1   | SQL Injection por concatenación en query findByUsername                | UserRepository.java     | 18                      | Seguridad básica (SQLi)                                 | Alto       |
| 2   | SQL Injection por concatenación en save (INSERT)                       | UserRepository.java     | 33                      | Seguridad básica (SQLi)                                 | Alto       |
| 3   | Credenciales hardcodeadas (url/user/pass)                              | UserRepository.java     | 11–13                   | Seguridad (exposición de secretos), Clean Code (config) | Alto       |
| 4   | MD5 para contraseñas (inseguro, sin salt)                              | AuthService.java        | 58–66, 21,45            | Seguridad (hashing), OCP (difícil de cambiar)           | Alto       |
| 5   | Se expone el hash de la contraseña en la respuesta                     | AuthService.java        | 28, 33                  | Seguridad (exposición de datos)                         | Alto       |
| 6   | Conexiones JDBC sin cerrar (sin try-with-resources)                    | UserRepository.java     | 16–19, 31–34            | Clean Code (recursos), Robustez                         | Alto       |
| 7   | Logging de datos sensibles (usuario/email en System.out)               | AuthService.java        | 24–25, 30–31, 47–48, 52 | Seguridad (exposición), Clean Code (logging)            | Medio      |
| 8   | Campos public en User (sin encapsulación)                              | User.java               | 3–5                     | Clean Code (encapsulación), SOLID (SRP)                 | Medio      |
| 9   | Parámetros crípticos (u, p, e, s, r) y throws Exception en controlador | AuthController.java     | 14, 21, 27              | Clean Code (naming), Manejo de errores                  | Medio      |
| 10  | Siempre devuelve 200 OK incluso en login fallido                       | AuthController.java     | 21–30                   | Clean Code/REST (códigos HTTP)                          | Medio      |
| 11  | DIP débil: AuthService depende de clase concreta UserRepository        | AuthService.java        | 12                      | SOLID (DIP)                                             | Bajo/Medio |
| 12  | Secretos en application.properties incluyendo target/classes           | resources/\*.properties | —                       | Seguridad (secrets), Build hygiene                      | Medio      |



# 📄 **FASE 3 — Pruebas Funcionales (Resultados y Análisis)**

*Proyecto: LoginCaos — Auditoría Técnica*  
*Autor: Sebastián Puentes González*  
*Fecha:* {{FECHA}}

***

## 🧪 **Prueba 1 — Login válido**

### ✔ **Descripción**

Se verifica que el login funcione con credenciales correctas y se analiza si la respuesta expone información sensible.

***

### 📝 **Comando ejecutado**

```http
POST http://localhost:8080/login?u=admin&p=12345
```

***

### 📸 **Evidencia (respuesta Postman)**

> *Inserta aquí tu imagen*

    ./imagenes/login_valido_response.png

***

### 📸 **Evidencia (logs de servidor)**

> *Inserta aquí tu imagen*

    ./imagenes/login_valido_logs.png

***

### ✅ **Resultado observado**

*   El servidor retorna:
    ```json
    {
      "ok": true,
      "user": "admin",
      "hash": "827ccb0eea8a706c4c34a16891f84e7b"
    }
    ```
*   Status HTTP: **200 OK**

***

### ⚠ **Datos sensibles expuestos**

| Dato                          | Nivel de riesgo | Motivo                                                     |
| ----------------------------- | --------------- | ---------------------------------------------------------- |
| `user` (“admin”)              | Medio           | Permite **enumeración de usuarios**                        |
| `hash` (MD5 de la contraseña) | **Crítico**     | Exposición directa del hash → Puede crackearse en segundos |

***

### ❌ **¿Debería devolverse esta información?**

No.  
El hash nunca debe enviarse al cliente.  
La respuesta debería limitarse a:

```json
{ "ok": true }
```

O (idealmente) un **JWT o token de sesión**, pero **nunca** información derivada de la contraseña.

***

***

# 💣 **Prueba 2 — SQL Injection**

### ✔ **Descripción**

Evalúa si el endpoint es vulnerable a inyección SQL manipulando el parámetro `u`.

***

### 📝 **Comando ejecutado**

```http
POST http://localhost:8080/login?u=admin'--&p=cualquiercosa
```

***

### 📸 **Evidencia (respuesta Postman)**

> *Inserta aquí tu imagen*

    ./imagenes/sql_injection_response.png

***

### 📸 **Evidencia (logs del servidor)**

> *Inserta aquí tu imagen*

    ./imagenes/sql_injection_logs.png

***

### ✅ **Resultado observado**

*   La consulta generada es:

```sql
select username, email, password 
from users 
where username = 'admin'--'
```

El comentario (`--`) **modifica completamente la consulta**, permitiendo omitir contenido crítico.

*   El sistema:
    *   Sí encuentra al usuario `admin`
    *   Compare el hash MD5 de "cualquiercosa"
    *   El login falla, pero la inyección **fue exitosa**

***

### ⚠ **Por qué es extremadamente peligroso**

| Riesgo                        | Impacto                                                                    |
| ----------------------------- | -------------------------------------------------------------------------- |
| Manipulación de consultas SQL | Un atacante puede alterar la lógica de autenticación                       |
| Filtración de información     | Puede leer datos de usuarios                                               |
| Manipulación de datos         | En combinación con el endpoint `/register`, permite inyecciones más graves |
| Ataques escalados             | Puede causar borrado masivo, extracción de credenciales o RCE              |

***

### ❗ **Conclusión**

El endpoint `/login` es **vulnerable a SQL Injection**, lo cual representa una amenaza crítica en entornos productivos.

***

***

# 🧪 **Prueba 3 — Registro con contraseña débil**

### ✔ **Descripción**

Se valida si la API permite crear usuarios con contraseñas inseguras.

***

## 🔽 **Prueba A — Contraseña “123”**

### 📝 **Comando**

```http
POST http://localhost:8080/register?u=test&p=123&e=test@test.com
```

### 📸 **Evidencia**

    ./imagenes/registro_p123.png

### 🧩 **Resultado**

*   Respuesta:
    ```json
    { "ok": false }
    ```
*   La API **rechaza** la contraseña porque mide 3 caracteres y la regla actual es:
    ```java
    p.length() > 3
    ```

***

## 🔽 **Prueba B — Contraseña “1234”**

### 📝 **Comando**

```http
POST http://localhost:8080/register?u=test&p=1234&e=test@test.com
```

### 📸 **Evidencia**

    ./imagenes/registro_p1234.png

### 🧩 **Resultado**

*   Respuesta:
    ```json
    {
      "ok": true,
      "user": "test"
    }
    ```
*   La API **acepta** la contraseña porque tiene **4 caracteres**, suficiente según la validación actual.

***

## 📌 **Conclusión de la Prueba 3**

| Contraseña | Resultado   | ¿Correcto? | Comentario                      |
| ---------- | ----------- | ---------- | ------------------------------- |
| `123`      | ❌ Rechazada | ✔          | Validación mínima               |
| `1234`     | ✔ Aceptada  | ❌          | Contraseña extremadamente débil |

### ⚠ Validación actual NO es suficiente

Tu API **no valida nada más** que longitud > 3.  
No exige:

*   Longitud mínima adecuada (8+)
*   Mezcla de mayúsculas / minúsculas
*   Caracter especial
*   Un número
*   No estar en lista de contraseñas comunes
*   Hash seguro (solo usa MD5)
*   Salt criptográfico
*   Validación de email
*   Validación de usuario duplicado

***

# 🔚 **Resumen final de la FASE 3**

| Prueba               | Resultado                        | Riesgo      |
| -------------------- | -------------------------------- | ----------- |
| **Login válido**     | Expone hash MD5 y nombre usuario | **Crítico** |
| **SQL Injection**    | La consulta es alterada          | **Crítico** |
| **Contraseña débil** | Acepta contraseñas inseguras     | **Alto**    |

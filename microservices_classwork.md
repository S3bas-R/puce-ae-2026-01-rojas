# Deber — Microservicios autenticados con tu propio User Pool de Cognito

**Materia:** Arquitectura Empresarial · PUCE · Período 2026-01 · Paralelo 1462
**Modalidad:** individual
**Entregable:** un PDF (evidencia) + el enlace a tu fork del repositorio

---

## 1. Objetivo

Demostrar, con evidencia propia y verificable, que entendiste el mecanismo completo de
autenticación y autorización del sandbox de microservicios:

1. **Tú** creas y administras el proveedor de identidad (un User Pool de AWS Cognito **tuyo**).
2. Los dos microservicios (`users` y `diary_auth_practice`) validan los JWT que emite **tu** pool.
3. La identidad de cada petición sale **del token**, nunca del cliente: cambiar de token cambia
   lo que el sistema te deja crear y ver.

> ⚠️ **La parte más importante del deber es que el User Pool sea tuyo.**
> Entregar evidencia generada con el pool del docente (`us-east-1_yzwNALI2A`) o con el pool de
> un compañero **anula el deber** (ver [§7](#7-rúbrica-de-calificación) y [§8](#8-causas-de-anulación)).

---

## 2. Qué debes hacer

### 2.1 Fork del repositorio

1. Haz **fork** de este repositorio a tu cuenta personal de GitHub.
2. Clónalo en tu máquina.
3. El fork debe quedar **público** (o compartido con el docente) hasta la fecha de calificación.

### 2.2 Crea tu propio User Pool en AWS Cognito

En la consola de AWS → **Amazon Cognito** → *Create user pool*:

- Nombre sugerido: `puce-ae-2026-01-<tu_apellido>`.
- Inicio de sesión por **username** (puedes habilitar también email).
- Crea un **App client** (tipo *public client*, sin secret, es el que usará Postman).
- Configura el **Hosted UI / dominio de Cognito** del app client con:
  - **Allowed callback URL:** `https://oauth.pstmn.io/v1/callback`
  - **OAuth 2.0 grant type:** *Authorization code grant*
  - **Scopes:** `openid`, `email`, `profile`
- Crea **al menos dos usuarios** en el pool (ej. `ana` y `beto`), cada uno con su contraseña
  definitiva (sin estado *FORCE_CHANGE_PASSWORD* pendiente).

Anota estos datos, los vas a necesitar:

| Dato | Ejemplo | Dónde se usa |
|---|---|---|
| Región | `us-east-1` | issuer-uri |
| User Pool ID | `us-east-1_ABC123xyz` | issuer-uri |
| App client ID | `4f9k...` | Postman |
| Dominio de Cognito | `https://<tu-dominio>.auth.us-east-1.amazoncognito.com` | Postman |

### 2.3 Cambia **solo** el `issuer-uri` en tu fork

Lo único que se modifica en el código son estas dos líneas:

- [users/src/main/resources/application.yaml:27](users/src/main/resources/application.yaml#L27)
- [diary_auth_practice/src/main/resources/application.yaml:25](diary_auth_practice/src/main/resources/application.yaml#L25)

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://cognito-idp.<TU_REGION>.amazonaws.com/<TU_USER_POOL_ID>
```

Haz **commit y push** de ese cambio a tu fork. No modifiques controladores, servicios,
`SecurityConfig`, `nginx.conf` ni `docker-compose.yml`.

### 2.4 Levanta el entorno

```bash
docker compose up --build
```

Endpoints públicos a través del reverse proxy (puerto `8888`):

| Método | URL | Qué hace |
|---|---|---|
| `POST` | `http://localhost:8888/users/api/users/me` | crea el perfil del usuario del token |
| `GET`  | `http://localhost:8888/users/api/users/me` | consulta el perfil del usuario del token |
| `POST` | `http://localhost:8888/diary/entries` | crea una nota del usuario del token |
| `GET`  | `http://localhost:8888/diary/entries` | lista **solo** las notas de ese usuario |

Notas técnicas que te ahorran tiempo:

- Usa el **`access_token`**, no el `id_token`. `users` lee el claim `sub` y `diary` lee el claim
  `username`; ambos existen en el access token de Cognito.
- Sin header `Authorization: Bearer <token>` todo responde `401`.
- Las bases son H2 **en memoria**: si reinicias los contenedores, se borran los datos.
- Las colecciones de Postman ya vienen armadas en
  [users/users.postman_collection.json](users/users.postman_collection.json) y
  [diary_auth_practice/diary.postman_collection.json](diary_auth_practice/diary.postman_collection.json).
  Recuerda ajustar `base_url` al puerto del proxy (`http://localhost:8888/users` y
  `http://localhost:8888/diary`).

---

## 3. Estructura obligatoria del PDF

El PDF es la evidencia y se califica **en este orden exacto**.

### Encabezado (primera página)

- Universidad, materia, paralelo y período.
- Título del deber.
- Nombre completo del estudiante.
- Fecha de entrega.
- **URL de tu fork en GitHub.**
- **Tu User Pool ID** y tu región, escritos en texto (no solo en las capturas).

### Índice

Índice con las secciones numeradas y su página. Puede ser generado automáticamente
(Word/Google Docs) o escrito a mano, pero debe corresponder a las páginas reales.

### Secciones, en este orden

**1. Capturas de Cognito**
1.1. Captura del **User Pool** creado por ti, donde se lea el nombre y el **User Pool ID**.
1.2. Captura de la lista de usuarios del pool con **al menos dos usuarios creados**.

**2. Capturas de Postman**

2.1. **Obtención del token con el flujo OAuth 2.0 de Postman**
- Captura de la pestaña *Authorization* configurada como `OAuth 2.0` (grant type
  *Authorization Code*, Auth URL y Token URL de **tu** dominio de Cognito, tu Client ID,
  callback `https://oauth.pstmn.io/v1/callback`, scopes).
- Captura del token obtenido (pantalla *Manage Access Tokens* o el resultado del login).
- Repite el proceso para el **segundo usuario** (o muestra los dos tokens obtenidos).

2.2. **Inspección del JWT y sus claims**
- Captura del token decodificado en [jwt.io](https://jwt.io) (o herramienta equivalente) para
  el **usuario 1**, señalando `iss`, `sub`, `username`, `token_use`, `exp`.
- Captura equivalente para el **usuario 2**.
- Debe **evidenciarse que el `sub` (user id) es distinto** entre ambos usuarios, y que el
  `iss` apunta a **tu** User Pool.

2.3. **Endpoint `/api/users/me`**
- `POST /users/api/users/me` con el token del usuario 1 → captura de la respuesta con el
  `cognitoId` creado.
- `GET /users/api/users/me` con el token del usuario 1 → captura de la consulta.
- `POST` y `GET` de `/api/users/me` con el token del usuario 2 → captura donde se vea que la
  información devuelta **cambió** (otro `id`, otro `cognitoId`, otro `name`).

2.4. **Creación y listado de notas (diary)**
- `POST /diary/entries` con el token del usuario 1 (una o dos notas) → captura de la respuesta,
  donde se vea el `author`.
- `GET /diary/entries` con el token del usuario 1 → captura del listado.
- `POST /diary/entries` y `GET /diary/entries` con el token del usuario 2 → captura donde se
  evidencie que **cada usuario ve solo sus propias notas** (el listado cambia según el `owner`).

**3. Conclusiones**

Media página, con tus palabras, respondiendo:
- ¿Por qué los microservicios no necesitan guardar contraseñas?
- ¿Qué habría pasado si el `author`/`cognitoId` se enviara en el body en lugar de leerse del token?
- ¿Por qué basta cambiar el `issuer-uri` para que todo el sistema confíe en otro proveedor de identidad?

---

## 4. Requisitos de las capturas

- Pantalla completa o ventana completa; se debe poder leer la URL/endpoint y el status code.
- **No** recortes ni tapes el `iss`, el `sub` ni el User Pool ID: son la prueba de autoría.
- Sin editar ni retocar imágenes (nada de texto pegado encima).
- Puedes ocultar datos personales reales (correo personal, número de cuenta AWS).
- Cada captura debe tener un pie de figura corto: *“Figura 3 — GET /api/users/me con el token de Ana”*.

---

## 5. Entrega

1. Sube el PDF a la plataforma del curso.
2. El PDF debe contener el enlace al fork; el fork debe tener el commit con **tu** `issuer-uri`.
3. Nombre del archivo: `AE_2026-01_1462_<Apellido>_<Nombre>_microservicios.pdf`.

---

## 6. Verificación de autoría (cómo se revisa)

Al calificar se cruzan tres fuentes; las tres deben coincidir con el **mismo** User Pool:

| Fuente | Qué se compara |
|---|---|
| Captura de la consola de Cognito | User Pool ID |
| `application.yaml` de tu fork (ambos servicios) | `issuer-uri` |
| Claim `iss` del JWT en jwt.io | issuer del token |

Si las tres no apuntan al mismo pool, la evidencia no es válida.

---

## 7. Rúbrica de calificación

**Total: 100 puntos.**

| # | Criterio | Puntos |
|---|---|---:|
| 1 | **User Pool propio, creado y administrado por el estudiante.** El User Pool ID de la captura, el `issuer-uri` del fork y el claim `iss` del JWT coinciden entre sí y son distintos del pool del docente y de los de sus compañeros. | **35** |
| 2 | **Dos o más usuarios propios** creados en ese pool, evidenciados en la consola y usados realmente para generar los dos tokens. | **10** |
| 3 | Fork del repositorio, público y accesible, con el commit que cambia el `issuer-uri` en **ambos** `application.yaml` y **sin** otras modificaciones al código. | **10** |
| 4 | Obtención del token con el **flujo OAuth 2.0 de Postman** correctamente configurado (Authorization Code, endpoints de tu dominio, callback de Postman, scopes). | **10** |
| 5 | JWT decodificado para los dos usuarios, con claims señalados y **`sub` distinto** demostrado. | **10** |
| 6 | `/api/users/me`: creación y consulta con ambos tokens, evidenciando que la respuesta cambia según el token. | **10** |
| 7 | Notas del diary: creación y listado con ambos tokens, evidenciando el aislamiento por `owner`. | **8** |
| 8 | Formato: encabezado completo, índice correspondiente a las páginas reales, secciones en el orden exigido, figuras numeradas con pie de figura. | **5** |
| 9 | Conclusiones con análisis propio (no copiadas de la documentación del repo). | **2** |

### Escala de logro del criterio 1 (el de mayor peso)

| Situación | Puntos del criterio 1 |
|---|---:|
| Pool propio, coincidencia perfecta entre consola, fork y claim `iss`. | 35 |
| Pool propio pero el fork quedó con el `issuer-uri` del docente en uno de los dos servicios. | 20 |
| Pool propio pero las capturas no permiten leer el User Pool ID / `iss`. | 12 |
| No hay evidencia de un pool propio (o se usó el del docente / de un compañero). | **0 en todo el deber** |

---

## 8. Causas de anulación

El deber se califica con **0** si:

- Se usó el User Pool del docente (`us-east-1_yzwNALI2A`) o el de otro estudiante.
- Dos entregas comparten el mismo User Pool ID.
- Las capturas están editadas, recortadas para ocultar el `iss`/`sub`, o son de otra persona.
- No existe fork, o el fork no tiene el cambio de `issuer-uri`.
- Se modificó el código de los servicios para saltarse la validación del token
  (por ejemplo, permitir peticiones sin `Authorization` o aceptar el `author` desde el body).

---

## 9. Checklist antes de entregar

- [ ] Mi User Pool existe en mi cuenta de AWS y sé su ID.
- [ ] Tengo al menos dos usuarios con contraseña definitiva.
- [ ] Mi app client tiene Hosted UI, callback de Postman, Authorization Code y scopes.
- [ ] Cambié el `issuer-uri` en **los dos** `application.yaml` y hice push al fork.
- [ ] `docker compose up --build` levanta los tres contenedores.
- [ ] Sin token obtengo `401` (lo probé).
- [ ] Obtuve **dos** access tokens, uno por usuario, con el flujo OAuth 2.0 de Postman.
- [ ] En jwt.io se ve `iss` = mi pool y `sub` distinto entre los dos usuarios.
- [ ] Las respuestas de `/api/users/me` y de `/diary/entries` cambian al cambiar el token.
- [ ] El PDF tiene encabezado, índice y las secciones **en el orden exigido**.
- [ ] El PDF incluye la URL de mi fork y mi User Pool ID en texto.

---

## 10. Material de apoyo

- [README.md](README.md) — cómo levantar el entorno.
- [DOCUMENTACION.md](DOCUMENTACION.md) — arquitectura, seguridad con Cognito, reverse proxy.
- [users/guia_microservicio_users.md](users/guia_microservicio_users.md) — el microservicio de perfiles.
- [diary_auth_practice/README.md](diary_auth_practice/README.md) — autorización sin roles (IDOR).

# Notas y explicaciones sobre la aplicación

Fijarme que le llegue al backend desde el frontend el token en cada petición. Repasar toda la aplicacion en general

El flujo general es:

1. Usuario se loguea → backend devuelve token (JWT) + setea cookie refreshToken
2. Frontend guarda el token en localStorage
3. En cada request, el interceptor de axios agrega el token en el header Authorization: Bearer <token>
4. El backend recibe el request, verifyToken extrae el payload del JWT y lo pone en req.user
5. Los controllers usan req.user.id y req.user.rol para saber quién hace la petición

El token no va en el payload — va en el header Authorization. El payload solo lleva los datos del pedido (items, cantidades). El token va aparte en la cabecera HTTP.

El interceptor ya lo hace automáticamente:

```typescript
api.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

Cuando el frontend hace:

```typescript
await api.post("/pedidos", payload);
```

Esto es en la página del carrito

Esto se añade automáticamente al header como info en el Bearer

El backend recibe ese request, verifyToken lee el header Authorization, verifica el JWT, y pone el payload en req.user. El controller entonces usa req.user.id para saber qué usuario está creando el pedido.

## Otras partes de la aplicación

### 1. Auth

**Login:** el frontend manda `email` + `contrasenia`. El backend busca el usuario, compara la contraseña con `bcrypt.compare` contra el `passwordHash` guardado en la DB. Si coincide, genera dos tokens con `jsonwebtoken`:

- `accessToken` (15 min) — se devuelve en el body de la respuesta
- `refreshToken` (7 días) — se setea como cookie `httpOnly` (no accesible desde JS, más seguro)

**¿Por qué bcrypt?** Las contraseñas nunca se guardan en texto plano. bcrypt aplica un hash irreversible con salt — aunque alguien robe la DB, no puede recuperar las contraseñas.

**Refresh:** cuando el `accessToken` expira, el interceptor de axios detecta el 401, llama a `POST /auth/refresh`, el backend lee la cookie `httpOnly` con el `refreshToken`, lo verifica, y devuelve un nuevo `accessToken`. El usuario no nota nada.

---

### 2. Autorización

**`verifyToken`:** lee el header `Authorization: Bearer <token>`, verifica la firma JWT con `JWT_SECRET`, y si es válido pone `{ id, rol }` en `req.user`. Si no hay token o es inválido → 401.

**`requireRole('admin')`:** verifica que `req.user.rol === 'admin'`. Si no → 403.

**`requireSelfOrAdmin`:** verifica que `req.user.id === req.params.id` o que el rol sea admin. Evita que un usuario acceda a recursos de otro.

**Flujo completo de un request protegido:**

```
Frontend → axios interceptor agrega token → Backend recibe request
→ verifyToken verifica JWT → req.user se puebla
→ requireRole/requireSelfOrAdmin verifica permisos
→ controller ejecuta lógica con req.user.id
```

---

### 3. Validación con Zod

Zod valida el body antes de que llegue al controller mediante el middleware `validate`:

```ts
export function validate(schema) {
  return (req, res, next) => {
    const result = schema.safeParse(req.body);
    if (!result.success)
      return res.status(400).json({ errors: result.error.issues });
    req.body.validated = result.data; // datos limpios y tipados
    next();
  };
}
```

El controller siempre usa `req.body.validated`, nunca `req.body` directamente. Esto evita que lleguen campos inesperados o con tipos incorrectos.

**`RegistroSchema` vs `UsuarioSchema`:** el registro excluye `rol` y `fondos` — el backend los fuerza a `"cliente"` y `0`. Así un usuario no puede registrarse como admin mandando `rol: "admin"` en el body.

---

### 4. Lógica de negocio

**Pedidos — transiciones de estado:**

```
pendiente → confirmado → pagado → enviado → entregado
               ↘ cancelado
```

El estado `pagado` solo puede llegar por el webhook de MercadoPago — el admin no puede setearlo manualmente. Cada transición se valida contra un mapa antes de persistir.

**Stock:** al crear un pedido, el controller verifica que `cantidad <= mueble.stock`. Si no hay stock → 400. Al cancelar un pedido, el stock se devuelve.

**Pagos:** el frontend llama a `POST /api/pagos/:pedidoId`, el backend crea una preferencia en MercadoPago con los items del pedido y devuelve una `checkoutUrl`. El usuario paga en MercadoPago, que notifica al backend via webhook → el backend actualiza el pedido a `pagado`.

---

### 5. Seguridad general

**CORS:** solo acepta requests de `localhost:5173` y la URL de Netlify. Bloquea cualquier otro origen.

**Helmet:** agrega headers de seguridad HTTP automáticamente (CSP, X-Frame-Options, etc.).

**Rate limiting:** máximo 100 requests por IP cada 15 minutos. Protege contra ataques de fuerza bruta.

**Cookie httpOnly:** el `refreshToken` no es accesible desde JavaScript — solo el browser lo envía automáticamente. Protege contra XSS.

**`sameSite: 'none'` + `secure: true`:** necesario en producción porque el frontend (Netlify) y el backend (Railway) están en dominios distintos. Sin esto el browser bloquearía la cookie.

---

El punto central que el profesor preguntó: **el token viaja en el header HTTP, no en el body**. El interceptor de axios lo agrega automáticamente en cada request. El backend lo lee con `verifyToken` y lo convierte en `req.user`. Nunca hay que mandarlo manualmente desde los componentes.

## Manejo del JWT token

Está bien manejado porque:

1. **`isRefreshing` + `failedQueue`** — si el token expira y hay varios requests en paralelo (por ejemplo, la página carga muebles, pedidos y perfil al mismo tiempo), solo se hace **un** refresh. Los demás requests se encolan y esperan. Cuando el refresh termina, todos se reintentan con el nuevo token. Sin esto, harías 3 refreshes en paralelo y el segundo y tercero fallarían.
2. **Exclusión de rutas** — `url !== "/auth/login"` y `url !== "/auth/refresh"` evitan loops infinitos. Si el refresh falla con 401, no intenta refrescarse a sí mismo indefinidamente.
3. **Limpieza al fallar** — si el refresh también falla (refreshToken expirado después de 7 días), limpia el localStorage y redirige al login. El usuario no queda en un estado roto.

Lo único que se podría cuestionar es que guarda el `token` en `localStorage` en lugar de una cookie httpOnly — localStorage es vulnerable a XSS. Pero el `refreshToken` sí está en cookie httpOnly, que es el más crítico porque dura 7 días. El `accessToken` dura solo 15 minutos, así que el riesgo es acotado. Es un tradeoff aceptable para un proyecto académico.

El `refreshToken` vive en una **cookie httpOnly** en el browser — no en el código del frontend ni en `localStorage`. Lo setea el backend al hacer login:

```ts
// auth.controller.ts
res.cookie(REFRESH_COOKIE, refreshToken, {
  httpOnly: true, // no accesible desde JS
  secure: true, // solo HTTPS
  sameSite: "none", // cross-origin (Netlify → Railway)
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7 días
});
```

Cuando el frontend llama a `POST /auth/refresh`, el browser **automáticamente** incluye esa cookie en el request gracias a `withCredentials: true` en axios — sin que el código JavaScript lo maneje explícitamente. Por eso en el interceptor el refresh es simplemente:

```ts
const { data } = await api.post("/auth/refresh");
```

Sin mandar nada en el body — el `refreshToken` ya viaja en la cookie automáticamente. El backend lo lee con `req.cookies.refreshToken`.

### Archivo adminCrear

`adminCrear.tsx` está bien estructurado. Puntos importantes a saber para la defensa:

**Protección de ruta:**

```ts
if (localStorage.getItem("rol") !== "admin") {
  navigate("/");
  return;
}
```

Redirige al home si no es admin. Es protección del lado del cliente — el backend igual rechaza con 403 si alguien intenta crear muebles sin el rol correcto.

**Tres formularios en un componente:**

- Crear material → `POST /api/materiales`
- Crear categoría → `POST /api/categorias`
- Crear mueble → `POST /api/muebles`

Las tres rutas requieren token de admin — el interceptor lo agrega automáticamente.

**Validación doble:**
Hay validación en el frontend (longitudes, números no negativos, URL válida) y también en el backend con Zod. Si alguien bypasea el formulario y manda el request directamente, el backend igual lo valida.

**Un bug menor:** en `handleCreateCategoria` usa `setCreatingMaterial(true)` en lugar de un estado separado `creatingCategoria`. Funciona igual pero deshabilita el botón de material mientras se crea una categoría. No es crítico.

**Carga de catálogos:**
Al montar, hace dos requests en paralelo con `Promise.all` para cargar categorías y materiales disponibles para los selects. Más eficiente que hacerlos secuencialmente.

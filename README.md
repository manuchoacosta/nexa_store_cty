# — Documentación del sistema

## 1. Descripción de la aplicación

**Nexa Store** es una aplicación web de gestión comercial (CRUD) construida con:

- **Backend**: Java 21 + Spring Boot 4.1.0 (stack **WebMVC** con controladores + Thymeleaf).
- **Base de datos**: PostgreSQL (`nexa_store_db`), accedida mediante Spring Data JPA / Hibernate.
- **ORM**: Entidades JPA mapeadas a un esquema existente (`spring.jpa.hibernate.ddl-auto=none`; el esquema NO se modifica ni se crea).
- **Frontend**: Thymeleaf con vistas HTML autocontenidas (CSS embebido) + JavaScript vanilla en el módulo de ventas (AJAX, cálculos dinámicos). Iconos de [Tabler Icons](https://tabler-icons.io/) en la página de inicio.

La aplicación gestiona **11 tablas** (10 módulos CRUD + 1 enum) que representan una cadena de tiendas: países, familias/departamentos/categorías de producto, tiendas, ciudades, vendedores, clientes, productos, ventas y sus detalles.

- Los 10 módulos CRUD (País, ProductoFamilia, Tienda, Ciudad, ProductoDepartamento, Vendedor, Cliente, ProductoCategoria, Producto, Venta) ofrecen: **listar** (con búsqueda por término), **crear**, **editar** y **cancelar/eliminar**.
- El módulo **Venta** integra cabecera + detalles (tipo POS/facturación) en un solo formulario, con cálculo automático de valor vendido y filtro AJAX de vendedores por tienda.

---

## 2. Módulos en orden de dependencia

Los módulos están ordenados del más "base" (sin dependencias) al más "final" (mayor número de relaciones). Un módulo solo puede crearse si existen registros de los módulos de los que depende (sus claves foráneas).

| # | Módulo | Tabla | Clave primaria | Dependencias (FK) |
|---|--------|-------|----------------|-------------------|
| 1 | País | `pais` | `id_pais` (Double) | — |
| 2 | Familia de producto | `producto_familia` | `id_producto_familia` (Double) | — |
| 3 | Tienda | `tienda` | `id_tienda` (Double) | — |
| 4 | Ciudad | `ciudad` | `id_ciudad` (Integer) | País |
| 5 | Departamento de producto | `producto_departamento` | `id_producto_departamento` (Integer) | Familia de producto |
| 6 | Vendedor | `vendedor` | `id_vendedor` (Integer) | Tienda (obligatoria) |
| 7 | Cliente | `cliente` | `id_cliente` (Integer) | Ciudad |
| 8 | Categoría de producto | `producto_categoria` | `id_producto_categoria` (Double) | Departamento de producto |
| 9 | Producto | `producto` | `id_producto` (Integer) | Categoría de producto |
| 10 | Venta | `venta` | `id_venta` (Integer) | Tienda, Cliente, Vendedor (obligatorias) |
| 11 | Detalle de venta | `venta_detalle` | `id_venta_detalle` (Integer) | Venta, Producto |

> Nota: los tipos de PK son `Double` en algunas tablas y `Integer` en otras (así están definidas en la base de datos; se respetaron).

### Enum `EstadoVenta`

| Valor | Descripción |
|-------|-------------|
| `Completada` | Estado por defecto al crear una venta |
| `Cancelada` | Solo por acción explícita (botón "Cancelar" en listado) |

El formulario de ventas nuevas muestra un badge "Pendiente" (cosmético, no se guarda). Al guardar, se persiste como `Completada`.

### Orden sugerido de carga de datos

```
Pais  →  ProductoFamilia  →  Tienda        (nivel 1, sin dependencias)
Ciudad                     (nivel 2, depende de Pais)
ProductoDepartamento       (nivel 2, depende de ProductoFamilia)
Vendedor  →  Cliente       (nivel 2, dependen de Tienda / Ciudad)
ProductoCategoria          (nivel 3, depende de ProductoDepartamento)
Producto                   (nivel 4, depende de ProductoCategoria)
Venta                      (nivel 4, depende de Tienda + Cliente + Vendedor)
VentaDetalle               (nivel 5, depende de Venta + Producto)
```

---

## 3. Relaciones entre entidades

Relaciones **@OneToMany** (lado "padre", carga perezosa `LAZY`):

| Entidad padre | Colección | Entidad hija (FK que apunta al padre) |
|---------------|-----------|---------------------------------------|
| `Pais` | `ciudades` | `Ciudad.pais` → `ciudad.id_pais` |
| `ProductoFamilia` | `productoDepartamentos` | `ProductoDepartamento.productoFamilia` → `producto_departamento.id_producto_familia` |
| `Tienda` | `vendedores`, `ventas` | `Vendedor.tienda` → `vendedor.id_tienda`; `Venta.tienda` → `venta.id_tienda` |
| `Ciudad` | `clientes` | `Cliente.ciudad` → `cliente.id_ciudad` |
| `ProductoDepartamento` | `productoCategorias` | `ProductoCategoria.productoDepartamento` → `producto_categoria.id_producto_departamento` |
| `Vendedor` | `ventas` | `Venta.vendedor` → `venta.id_vendedor` |
| `Cliente` | `ventas` | `Venta.cliente` → `venta.id_cliente` |
| `ProductoCategoria` | `productos` | `Producto.productoCategoria` → `producto.id_producto_categoria` |
| `Producto` | `ventaDetalles` | `VentaDetalle.producto` → `venta_detalle.id_producto` |
| `Venta` | `ventaDetalles` | `VentaDetalle.venta` → `venta_detalle.id_venta` (cascade = ALL, orphanRemoval = true) |

Relaciones **@ManyToOne** (lado "hijo"):

| Entidad hija | FK (columna) | Entidad padre |
|--------------|--------------|---------------|
| `Ciudad` | `id_pais` | `Pais` |
| `ProductoDepartamento` | `id_producto_familia` | `ProductoFamilia` |
| `Vendedor` | `id_tienda` (NOT NULL) | `Tienda` |
| `Cliente` | `id_ciudad` | `Ciudad` |
| `ProductoCategoria` | `id_producto_departamento` | `ProductoDepartamento` |
| `Producto` | `id_producto_categoria` | `ProductoCategoria` |
| `Venta` | `id_tienda`, `id_cliente`, `id_vendedor` (NOT NULL) | `Tienda`, `Cliente`, `Vendedor` |
| `VentaDetalle` | `id_venta`, `id_producto` | `Venta`, `Producto` |

Las FKs `vendedor.id_tienda` y las tres FKs de `venta` son **obligatorias** (`NOT NULL`); el resto son opcionales.

---

## 4. Funcionalidades de cada módulo

### Módulos CRUD estándar (10 módulos)

1. **Listar** (`GET /<recurso>`) — tabla con los registros, ordenados por id descendente (máx. 50). Cuando se muestra una relación, la columna muestra la descripción/nombre del padre.
2. **Buscar** (`GET /<recurso>?termino=...`) — filtra por coincidencia parcial (case-insensitive) sobre los campos textuales y numéricos relevantes. Incluye botones *Buscar* y *Limpiar*.
3. **Crear** (`POST /<recurso>/guardar`) — el formulario oculta el campo de id; el sistema lo asigna automáticamente.
4. **Editar** (`GET /<recurso>/formulario?id=X`) — el formulario muestra el id de forma fija y permite modificar el resto; los desplegables quedan preseleccionados.
5. **Eliminar** (`POST /<recurso>/eliminar/{id}`) — elimina el registro; si no existe, se ignora sin error.

### Módulo Venta (integrado con detalles)

El formulario de ventas es un **POS/facturación** integrado que gestiona cabecera y detalles en una sola vista:

- **Cabecera**: fecha (automática), tienda, cliente, vendedor (filtrado por tienda vía AJAX).
- **Detalles**: selección de producto (con precio de costo visible), unidades; el campo **Valor Vendido** se calcula automáticamente como `precioCosto × unidades`.
- **Badge de estado**: "Pendiente" (amarillo) al crear; "Completada" (verde) o "Cancelada" (rojo) al editar.
- **Cancelar**: `POST /ventas/cancelar/{id}` — cambia el estado a `Cancelada` (no elimina registros).

### Página de inicio (`/`)

- Tarjetas de acceso a los **10 módulos** con iconos de Tabler Icons (SVG inline).

### Menú de tienda (`/tienda/{id}/menu`)

- Datos de la tienda.
- **Vendedores** de esa tienda (tabla).
- **Ventas** de esa tienda (tabla con fecha y estado).
- Accesos directos a "Gestionar vendedores" y "Gestionar ventas".

---

## 5. Detalles de implementación importantes

### 5.1 Estructura del proyecto

```
src/main/java/py/edu/nexa_store/
├── entidades/          → 11 entidades JPA + 1 enum (EstadoVenta)
│                          Pais, ProductoFamilia, Tienda, Ciudad,
│                          ProductoDepartamento, Vendedor, Cliente, ProductoCategoria,
│                          Producto, Venta, VentaDetalle, EstadoVenta
├── repositorios/       → 11 interfaces *Repositorio (Spring Data JPA)
├── servicios/          → 10 servicios *Servicio (lógica de negocio + transacciones)
├── dto/                → VentaFormDTO (DTO integrado venta + detalles)
├── excepciones/        → RecursoNoEncontradoException
└── controladores/      → 10 *Controlador (MVC) + InicioControlador

src/main/resources/templates/
├── <recurso>/form.html → vista de formulario de cada módulo
├── venta/list.html     → listado de ventas con búsqueda
├── venta/form.html     → formulario integrado venta + detalles (POS)
├── index.html          → página de inicio (iconos Tabler Icons)
└── tienda/menu.html    → menú por tienda
```

### 5.2 Generación de ids (sin tocar la base de datos)

Como el esquema no se modifica (sin `GENERATED BY DEFAULT` en las PKs), los ids se generan en la aplicación:

- Cada repositorio define `getNextId()`:

  ```java
  @Query("SELECT COALESCE(MAX(p.idPais), 0) + 1 FROM Pais p")
  Double getNextId();
  ```

- Cada servicio expone `obtenerSiguienteId()`.
- En el controlador, al **crear** un registro se asigna el id ANTES de guardar:

  ```java
  if (entidad.getId() == null) {
      entidad.setId(servicio.obtenerSiguienteId());
      servicio.guardar(entidad);
  } else {
      // edición: actualizar
  }
  ```

- El formulario **no tiene input de id** en modo creación; al editar, el id se muestra como texto fijo.

### 5.3 Binding de claves foráneas en formularios

**Patrón estándar de todo el proyecto**:

1. La entidad declara un campo `@Transient` con el id de la FK:

   ```java
   @Transient
   private Double paisId;
   ```

2. El `<select>` se vincula con `th:field`:

   ```html
   <select th:field="*{paisId}">
   ```

3. Al **editar**, el controlador prellena el campo transitorio desde la relación:

   ```java
   if (ciudad.getPais() != null) {
       ciudad.setPaisId(ciudad.getPais().getIdPais());
   }
   ```

4. Al **guardar**, el controlador resuelve la relación buscando el padre:

   ```java
   if (ciudad.getPaisId() != null) {
       ciudad.setPais(paisServicio.buscarPorId(ciudad.getPaisId()));
   } else {
       ciudad.setPais(null);
   }
   ```

### 5.4 Carga perezosa y OSIV

- Todas las colecciones y relaciones son `FetchType.LAZY`.
- En WebMVC, **OSIV está habilitado por defecto**: el `EntityManager` permanece abierto durante el renderizado de Thymeleaf, por lo que el lazy loading funciona en templates.
- Se mantienen `LEFT JOIN FETCH` en las consultas de listado/búsqueda como buena práctica para evitar queries N+1:

  ```java
  @Query("SELECT v FROM Venta v LEFT JOIN FETCH v.tienda "
       + "LEFT JOIN FETCH v.cliente LEFT JOIN FETCH v.vendedor "
       + "ORDER BY v.idVenta DESC LIMIT 50")
  List<Venta> listarTodos();
  ```

### 5.5 Búsqueda por término

- Endpoint GET con query param `termino`.
- El controlador decide entre lista completa o filtrada:

  ```java
  model.addAttribute("ciudades",
      (termino != null && !termino.isBlank())
          ? ciudadServicio.buscarPorTermino(termino.trim())
          : ciudadServicio.listarTodos());
  ```

- Las consultas usan `LIKE CONCAT('%', :termino, '%')` con `LOWER(...)` para búsqueda sin distinguir mayúsculas. Los números se comparan con `CAST(campo AS string)`.
- Los formularios usan `th:value="${termino}"` para conservar el término al limpiar/refrescar.
- Todos los listados y búsquedas usan `LIMIT 50` como máximo.

### 5.6 Venta integrada (POS/Facturación)

El módulo de ventas usa un **DTO integrado** (`VentaFormDTO`) que agrupa la cabecera (entidad `Venta`) y los detalles (lista de `DetalleForm`) en un solo formulario:

```java
@Data
public class VentaFormDTO {
    private Venta venta;
    private List<DetalleForm> detalles = new ArrayList<>();

    @Data
    public static class DetalleForm {
        private Integer productoId;
        private String productoNombre;
        private Integer unidadesVendidas;
        private BigDecimal valorVendido;
    }
}
```

**API AJAX de vendedores**: `GET /ventas/api/vendedores?tiendaId=X` retorna JSON con `[{idVendedor, nombre, apellido}]`. JavaScript escucha el `change` del select de tienda y filtra vendedores dinámicamente.

**Cálculo automático de valor vendido**: cada `<option>` del select de productos lleva un atributo `data-precio` con el `precioCosto`. Al cambiar el producto o la cantidad, JavaScript calcula `precioCosto × unidades` y actualiza el campo Valor Vendido.

### 5.7 Estado de ventas (enum)

La entidad `Venta` usa el enum `EstadoVenta` (`Completada`, `Cancelada`) con `@Enumerated(EnumType.STRING)`:

```java
@Enumerated(EnumType.STRING)
@Column(name = "estado", columnDefinition = "VARCHAR(20) DEFAULT 'Completada'")
private EstadoVenta estado;
```

- Al **crear** una venta: estado = `Completada` (asignado automáticamente en el controlador).
- El form muestra un badge "Pendiente" (cosmético) para ventas nuevas no guardadas.
- Al **editar**: badge verde `Completada` o rojo `Cancelada` (solo lectura).
- **Cancelar**: `POST /ventas/cancelar/{id}` cambia el estado a `Cancelada` sin eliminar registros.

### 5.8 Validación y errores

- Las FKs obligatorias de `Venta` y `Vendedor` se validan en el controlador; si faltan, se muestra un mensaje de error y se re-renderiza la vista sin perder la lista.
- `RecursoNoEncontradoException` se lanza al buscar ids inexistentes y se captura en el controlador (mensajes amigables, sin pantalla de error).
- `application.properties` activa logs DEBUG de Hibernate (`show-sql`, `format_sql`, `BasicBinder=TRACE`) para depuración.

### 5.9 Configuración

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/nexa_store_db
spring.datasource.username=postgres
spring.datasource.password=postgres
spring.jpa.hibernate.ddl-auto=none
spring.mvc.format.date=yyyy-MM-dd
```

---

## 6. Rutas principales

| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/` | Página de inicio (10 módulos con iconos Tabler Icons) |
| GET | `/tienda/{id}/menu` | Menú de una tienda (vendedores y ventas) |
| GET | `/paises`, `/producto-familias`, `/tiendas`, `/ciudades`, `/producto-departamentos`, `/vendedores`, `/clientes`, `/producto-categorias`, `/productos` | Listar (con `?termino=` para buscar) |
| GET | `/ventas` | Listado de ventas con búsqueda |
| GET | `/ventas/formulario` | Formulario para nueva venta (POS integrado) |
| GET | `/ventas/formulario?id=X` | Formulario para editar venta con detalles |
| GET | `/ventas/api/vendedores?tiendaId=X` | API JSON: vendedores filtrados por tienda |
| GET | `/paises/editar/{id}` (y equivalentes) | Ver formulario para editar |
| POST | `/paises/guardar` (y equivalentes) | Crear o actualizar |
| POST | `/ventas/guardar` | Guardar venta + detalles (via VentaFormDTO) |
| POST | `/ventas/cancelar/{id}` | Cancelar venta (estado → Cancelada) |
| POST | `/paises/eliminar/{id}` (y equivalentes) | Eliminar registro |

# Arquitectura — Simple Stock Flow

> **Primera pasada**, reconstruida **hacia atrás** desde `spec/data-model.md` (único insumo del reto).
> Cada afirmación lleva su fuente (`§n`, `FK-n`, `D-nn`). Lo que no sale del modelo se marca como
> **supuesto** (`S-nn`) y se lista en §10. Esta carpeta se **cierra al final** (paso 6), contrastándola
> con `04`, `03`, `02` y `01`.

---

## 1. Estilo arquitectónico

**Arquitectura hexagonal (puertos y adaptadores).** El dominio no conoce la base de datos ni el
almacenamiento: habla con el exterior solo a través de puertos.

| Evidencia en el modelo | Fuente |
|---|---|
| "El dominio nunca ve la clave en claro; el hash lo produce un puerto" | §1, §2.5, D-09 |
| El reporte se calcula en el motor por un **puerto de lectura** | §1, D-06 |
| La traducción entre colecciones de C# y tablas es responsabilidad del **adaptador de persistencia**, "donde vive el mapeo" | §0 |
| Si `stock >= 0` salta, "algo escribió fuera del adaptador" | §1, ADR-002 |
| El binario de imagen vive en un **almacenamiento externo**, referido por clave opaca | §1, D-08 |
| La base **no** participa en la transacción del almacenamiento externo | §7.1 |

**No es un sistema distribuido.** Hay una sola base (`simple_stock_flow`, esquema `sales`) y el
modelo no describe ningún otro servicio con datos propios (§0, §3). **S-01:** se asume un único
backend desplegable.

---

## 2. Piezas del sistema

| Pieza | Responsabilidad | Fuente |
|---|---|---|
| **Dominio** (`src/domain/`) | Agregados, objetos de valor (`Money`, `Quantity`), invariantes de negocio | §2, §12 |
| **Aplicación** | Casos de uso; objetos de valor propios de consulta (*Rango de fechas*); orquesta puertos | §1 (Rango de fechas) |
| **Adaptador saliente de persistencia** (`src/adapters/outbound/persistence/Configurations/`) | Mapeo EF Core ↔ tablas; propiedades sombra (`deleted_at`, `xmin`, FK); filtro global de baja lógica | §0, §3, §12 |
| **Adaptador entrante (API)** | Expone el contrato HTTP. El contrato vive en `api-contract.md`, **fuera de este modelo** | §12 |
| **Base de datos** | PostgreSQL 16, esquema `sales`. Dueño del DDL: **las migraciones de EF y nada más** | §3.2, ADR-001 |
| **Infraestructura** | Levanta el motor; **no define el esquema** | §6.2 |

Los tres repositorios del proyecto citados (`simple-stock-flow-api`, `-infra`, `-docs`) confirman la
separación API / infraestructura / documentación (§10, §12).

---

## 3. Agregados y sus fronteras

| Agregado | Raíz | Contiene | Ciclo de vida | Fuente |
|---|---|---|---|---|
| **Catálogo** | `Product` | Nombre, precio, stock, categoría, imagen (clave opaca) | Baja **lógica**, nunca física | §2.2, ADR-003 |
| **Ventas** | `Sale` | `SaleItem` (composición, constructor `internal`) | **Inmutable** tras registrarse | §2.3, §2.4 |
| **Identidad** | `User` | `username`, `password_hash`, `role` | Sin borrado | §2.5, §7.1 |
| *(referencia)* | `Category` | `name` | **No es agregado**; repositorio de solo lectura; 5 filas sembradas | §2.1, §9.1 |

**Regla de frontera:** los agregados se cruzan **solo por identidad de la raíz** (`category_id`,
`product_id`, `sold_by_user_id`), nunca por navegación de objetos (§5, columna *Naturaleza*).
`SaleItem` no existe fuera de su venta (FK-2 con `CASCADE`, §5).

---

## 4. Puertos

Los **nombres son propuestos** (S-02); lo que fija el modelo es la necesidad de cada uno.

| Puerto | Tipo | Patrón de acceso que sirve | Fuente |
|---|---|---|---|
| Repositorio de productos | Saliente | Q1 buscar (texto, categoría, activos, paginado), Q2 por id, Q3 por lote | §6.1 |
| Repositorio de categorías | Saliente, **solo lectura** | Q4 listar, Q5 por id | §2.1, §6.1 |
| Repositorio de ventas | Saliente | Q6 venta con líneas, Q7 por rango paginado | §6.1 |
| **Puerto de lectura del reporte** | Saliente, **modelo de lectura** | Q9 agregación por producto en el motor. No se persiste | §1, D-06, ADR-004 |
| Repositorio de usuarios | Saliente | Q10 por nombre exacto, en cada inicio de sesión | §6.1 |
| **Puerto de hash de clave** | Saliente | Producir y verificar `password_hash` | D-09, §7 |
| **Puerto de almacenamiento de imágenes** | Saliente | Guardar/borrar binario; el dominio solo ve `image_key` | D-08, §7.1 |

**Q8** (ventas por rango sin paginar) "no tiene consumidor" y el modelo recomienda **retirarlo del
puerto** en vez de dejarlo como trampa (§6.1).

---

## 5. Dónde vive cada regla

Este es el eje de la arquitectura: el modelo clasifica **cada regla** en una de tres marcas (§ "Cómo
se lee"). Una invariante que solo vive en C# **protege a la aplicación, no a los datos**: un `psql`
manual la salta sin ruido.

### 5.1 En el motor (hoy)

| Regla | Objeto | Fuente |
|---|---|---|
| Claves primarias de las 5 tablas | `PK_*` | §4 |
| `category.name` único · `user.username` único | `IX_category_name`, `IX_user_username` (índices únicos) | §4 |
| `product.stock >= 0` | `ck_product_stock_non_negative` — "última barrera" | §2.2, ADR-002 |
| Categoría obligatoria y existente | FK-1 `RESTRICT` | §5 |
| Línea no existe sin venta | FK-2 `CASCADE` | §5 |
| Baja lógica de producto | `product.deleted_at` + filtro global | §2.2, §13 D-1, ADR-003 |
| Línea → producto no se borra | FK-3 `RESTRICT` (T-20) | §5, §13 D-2 |
| Un producto no se repite en una venta | Índice único `(sale_id, product_id)` (T-20) | §2.3, §13 D-2 |

### 5.2 Solo dominio (deuda declarada: T-20 las baja al motor)

`price > 0` · `quantity > 0` · `category.name` no vacío · `user.role ∈ {admin, seller}` ·
`username` en minúsculas y recortado (§4).

### 5.3 Solo dominio **por naturaleza** (no expresables en un `CHECK`)

- **Retirar más stock del disponible falla** — regla de proceso (`Product.Withdraw`, §2.2).
- **Una venta confirmada tiene al menos una línea** — exigiría un disparador diferido (§2.3).
- **Descontar stock y añadir la línea son una sola operación** (`Sale.AddItem` → `Product.Withdraw`, §2.3).
- **Inmutabilidad de la venta** — se garantiza **por ausencia** de operación de edición/borrado (§2.3).

### 5.4 Pendiente

`sale.sold_by` → `sold_by_username` y `sold_by_user_id` con FK-4 `RESTRICT` (T-12) · índices
parciales y de trigramas (T-13) · `category_name` congelado en `sale_item` (T-11) (§3, §5, §6.2).

---

## 6. Flujo crítico: registrar una venta

1. El adaptador entrante recibe la solicitud y la traduce a un caso de uso.
2. El caso de uso **lee los productos por lote** (Q3) — "el punto de contención de D-04" (§6.1).
3. `Sale.AddItem` llama a `Product.Withdraw` (descuenta stock) y **copia nombre, precio y categoría
   congelados** a la línea (§1 *Nombre congelado*, §2.3, §2.4).
4. `Sale.EnsureConfirmable` exige al menos una línea (§2.3).
5. El adaptador persiste **en una sola transacción**; EF detecta conflicto con `xmin` (D-04, §3).
6. Si, pese a todo, algo escribió stock negativo, el `CHECK` del motor falla ruidosamente (ADR-002).

**Qué se garantiza y qué no:** la atomicidad vale **dentro de la base**. El almacenamiento externo
**no participa** de la transacción (§7.1), así que ninguna operación que mezcle ambos promete
atomicidad.

---

## 7. Concurrencia

**Optimista, con `xmin` como testigo** (D-04, T-10). `xmin` es columna de sistema de Postgres,
expuesta como propiedad sombra, y "no aparece en `information_schema`" (§3). La última barrera ante
una carrera que se escape del control optimista es el `CHECK` de `stock` (ADR-002).

---

## 8. Persistencia y esquema

- **Las migraciones son dueñas del DDL**, sin excepción (ADR-001). Incluye extensiones: `pg_trgm`
  se instalaría **dentro** de la migración del índice (§6.2).
- **Ninguna columna tiene `DEFAULT`:** los valores los pone el dominio (§3).
- **Marcas de tiempo `timestamptz`**, servidor en UTC (§3).
- **Monomoneda:** ninguna columna de moneda (D-05).
- **Sin columnas de auditoría** `created_at` / `updated_at` (decisión cerrada, §8 del modelo).
- **Semilla:** 5 categorías con id fijo en la migración inicial; **el administrador inicial no**: lo
  crea el arranque de la aplicación con credenciales de entorno (§9.1, §9.2, D-09, D-10).
- **Nombres:** tablas en singular, esquema `sales` intacto (§0).

---

## 9. Seguridad y privacidad (decisiones con impacto arquitectónico)

| Decisión | Fuente |
|---|---|
| `password_hash` jamás en logs, respuestas ni proyecciones, y **nunca se indexa** | §7 |
| El dominio nunca ve la clave en claro | D-09, §2.5 |
| Roles cerrados: `admin`, `seller` | §1, §2.5 |
| **Nadie otorga `admin` en ejecución:** lo provisiona el despliegue desde el entorno; un administrador da de alta vendedores | §11 H-3, DP-04 |
| `username` y `sold_by` son **datos personales**; acceso restringido | §7 |
| El reporte **no se desglosa por vendedor** | DP-02 |
| No hay entidad cliente ni comprador | §1 |

---

## 10. Supuestos

| # | Supuesto | Por qué hace falta |
|---|---|---|
| S-01 | Un único backend desplegable (no distribuido) | El modelo describe una sola base y ningún otro servicio |
| S-02 | Los nombres de los puertos son propuestos | El modelo fija la necesidad, no el identificador |
| S-03 | La autenticación entrega un token/sesión por el adaptador entrante | El modelo habla de "inicio de sesión" (Q10) pero no del mecanismo |
| S-04 | No hay interfaz de usuario dentro del alcance de esta documentación | El modelo no menciona frontend |

---

## 11. Puntos a verificar en el cierre (paso 6)

El modelo tiene incoherencias internas que la arquitectura **no debe heredar sin decirlo**:

1. **Conteo de columnas:** §3 dice 22; el enlace y §10 dicen 21. Aquí no se fija un número.
2. **Foto vieja vs. foto nueva:** la salida de §10 es del **2026-09-19** y §13 mide el **2026-09-20**.
   Esta arquitectura toma §13 como estado vigente, porque "gana el motor" y es la medición posterior.
3. **Índice `(sale_id, product_id)`:** §6.2 lo da por faltante (T-13); §2.3, §4 y §13 lo dan por
   existente (T-20). Aquí se trata como **existente**.
4. **`category_name`:** aparece en la tabla de `product` en §3 (error: pertenece a `sale_item`), y
   §2.4 lo marca *motor* mientras §3 lo marca *pendiente (T-11)*. Aquí se trata como **pendiente**.

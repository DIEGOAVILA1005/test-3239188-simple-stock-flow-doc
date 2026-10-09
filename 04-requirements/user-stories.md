# Historias de usuario — Simple Stock Flow

> Reconstruidas **desde `spec/data-model.md`**: cada historia nace de un patrón de acceso (Q1–Q10,
> §6.1), una invariante (§2) o una decisión cerrada (§11). La columna *Fuente* permite rastrear cada
> criterio al modelo. Lo que no sale del modelo se marca **supuesto** (`S-nn`) y se lista al final.

**Actores** (§1, §2.5): **administrador** (`role = admin`) y **vendedor** (`role = seller`). No existe
actor *cliente* ni *comprador* (§1, §7).

---

## US-01 · Iniciar sesión
**Como** operador **quiero** autenticarme con mi usuario y clave **para** usar el sistema.

| # | Criterio de aceptación | Fuente |
|---|---|---|
| 1 | El sistema busca al usuario por nombre exacto (patrón de alta frecuencia, en cada inicio de sesión) | Q10, §6.1 |
| 2 | El nombre de usuario se compara en minúsculas y recortado | §2.5 (`NormalizeUsername`) |
| 3 | La clave se verifica contra `password_hash` **a través del puerto de hash**; el dominio nunca ve la clave en claro | D-09, §2.5 |
| 4 | `password_hash` no aparece en ninguna respuesta, log ni mensaje de error | §7 |

## US-02 · Dar de alta vendedores
**Como** administrador **quiero** crear usuarios vendedor **para** que registren ventas.

| # | Criterio de aceptación | Fuente |
|---|---|---|
| 1 | Solo un administrador autenticado puede dar de alta usuarios; el alta anónima **no** es válida | §11 H-3, DP-04, §9.2 (defecto A-1) |
| 2 | El rol asignable en ejecución es `seller`; **nadie otorga `admin` en ejecución** (lo provisiona el despliegue desde el entorno) | §11 H-3, DP-04, §9.2 |
| 3 | `username` obligatorio y **único**; se guarda en minúsculas y recortado | §2.5, §4 |
| 4 | La clave se guarda solo como hash | D-09, §1 |

## US-03 · Crear producto
**Como** administrador **quiero** registrar un producto **para** venderlo.

| # | Criterio de aceptación | Fuente |
|---|---|---|
| 1 | Un producto tiene **nombre, precio, stock, categoría e imagen opcional, y nada más** | §1, DP-03 |
| 2 | Nombre obligatorio, no vacío, guardado recortado | §2.2 |
| 3 | `precio > 0` | §2.2 |
| 4 | `stock >= 0` | §2.2, `ck_product_stock_non_negative` |
| 5 | La categoría es obligatoria y debe existir | §2.2, FK-1 |
| 6 | La imagen es opcional; ausente se representa con `NULL`, nunca con cadena vacía | §1, §2.2 |

## US-04 · Buscar productos
**Como** vendedor o administrador **quiero** buscar productos **para** encontrarlos rápido.

| # | Criterio de aceptación | Fuente |
|---|---|---|
| 1 | Filtra por **texto parcial** en el nombre y por **categoría** | Q1, §6.1 |
| 2 | Devuelve solo productos **activos** (no dados de baja) | Q1, §6.1, ADR-003 |
| 3 | Ordena por nombre y **pagina**, devolviendo también el total de elementos | Q1, §6.1 |
| 4 | Se puede consultar un producto por identificador | Q2, §6.1 |

## US-05 · Modificar un producto
**Como** administrador **quiero** renombrar, repreciar, reponer stock y recategorizar **para** mantener el catálogo.

| # | Criterio de aceptación | Fuente |
|---|---|---|
| 1 | Aplican las mismas invariantes de US-03 | §2.2 |
| 2 | Reponer stock suma unidades; **retirar más stock del disponible falla** | §2.2 (`Restock`, `Withdraw`) |
| 3 | Si dos cambios compiten sobre el mismo producto, el segundo se detecta como **conflicto** y no pisa al primero | D-04, ADR-002, §3 (`xmin`) |
| 4 | Cambiar nombre, precio o categoría **no reescribe** ventas ya registradas | §1 *Nombre congelado*, ADR-004 |

## US-06 · Dar de baja un producto
**Como** administrador **quiero** retirar un producto del catálogo **para** que no se siga vendiendo.

| # | Criterio de aceptación | Fuente |
|---|---|---|
| 1 | La baja es **lógica** (`deleted_at`); **nunca** se borra físicamente | §2.2, §7.1, ADR-003 |
| 2 | Un producto dado de baja no aparece en búsquedas ni puede venderse | Q1, Q3, §5 |
| 3 | Las ventas históricas que lo incluyen se conservan intactas | §7.1, FK-3 |
| 4 | Un `DELETE` físico manual **falla ruidosamente** | FK-3 `RESTRICT`, §5 |

## US-07 · Gestionar la imagen de un producto
**Como** administrador **quiero** asociar, reemplazar o quitar la imagen **para** identificar el producto.

| # | Criterio de aceptación | Fuente |
|---|---|---|
| 1 | El sistema guarda solo una **clave opaca** (`image_key`), nunca la ruta ni los bytes | §1, D-08 |
| 2 | Al reemplazar o dar de baja, primero se anula `image_key` y **se confirma** la transacción; **después** se borra el binario | §7.1 |
| 3 | No se promete atomicidad con el almacenamiento externo: un binario huérfano es aceptable, una clave rota no | §7.1, H-2 |

## US-08 · Consultar categorías
**Como** operador **quiero** ver las categorías **para** clasificar y filtrar productos.

| # | Criterio de aceptación | Fuente |
|---|---|---|
| 1 | Hay **cinco** categorías fijas, sembradas en la migración inicial | §1, §9.1, D-10 |
| 2 | Se listan ordenadas por nombre | Q4, §6.1 |
| 3 | **No** se pueden crear, renombrar ni borrar categorías | §2.1 |

## US-09 · Registrar una venta
**Como** vendedor **quiero** registrar una venta de varios productos **para** descontar stock y dejar constancia.

| # | Criterio de aceptación | Fuente |
|---|---|---|
| 1 | La venta registra **quién** la hizo y **cuándo** (`sold_at`) | §2.3, §3 |
| 2 | Tiene **al menos una línea** para confirmarse | §2.3 (`EnsureConfirmable`) |
| 3 | **Un producto no se repite** dentro de la misma venta | §2.3, §13 D-2 |
| 4 | Cada línea tiene `cantidad > 0` | §2.4 (`Quantity`) |
| 5 | Descontar stock y añadir la línea son **una sola operación**; si el stock no alcanza, la venta falla | §2.3, §2.2 |
| 6 | La línea **congela** nombre de producto, precio unitario y nombre de categoría del momento | §1, §2.4, D-06 |
| 7 | Solo se pueden vender productos existentes y **no dados de baja** | §5 |
| 8 | El total y los subtotales **se calculan, no se almacenan** | §1, art. VII |

## US-10 · Consultar una venta
**Como** operador **quiero** ver una venta con sus líneas **para** revisar qué se vendió.

| # | Criterio de aceptación | Fuente |
|---|---|---|
| 1 | Se muestra la venta con todas sus líneas | Q6, §6.1 |
| 2 | Los valores mostrados son los **congelados** en la venta, no los vigentes del catálogo | §1 |
| 3 | La venta **no se edita ni se borra** | §2.3, §7.1 |

## US-11 · Listar ventas por rango de fechas
**Como** administrador **quiero** ver las ventas de un período **para** revisarlas.

| # | Criterio de aceptación | Fuente |
|---|---|---|
| 1 | Se filtra por **rango de fechas**; el fin no puede ser anterior al inicio | §1 (*Rango de fechas*), Q7 |
| 2 | Ordena por fecha descendente y **pagina**, con el total de elementos | Q7, §6.1 |

## US-12 · Reporte de ventas por producto
**Como** administrador **quiero** un reporte agregado por producto en un rango **para** saber qué se vende más.

| # | Criterio de aceptación | Fuente |
|---|---|---|
| 1 | Agrega **por producto** sobre un rango de fechas, ordenado por importe descendente | §1, Q9, §6.1 |
| 2 | Se calcula **en el motor** por un puerto de lectura; **no se persiste** | §1, D-06 |
| 3 | Si un producto se recategorizó dentro del rango, se **agrupa por la etiqueta congelada**: aparecen **dos filas**, no una | §11.1 (H-1 cerrado) |
| 4 | Un reporte de un rango cerrado **no cambia** aunque se registren ventas nuevas con otras etiquetas | §11.1 |
| 5 | El reporte **no se desglosa por vendedor** | DP-02, §7.1 |

---

## Fuera de alcance (decidido en el modelo)

Cliente o comprador (§1) · mantenimiento de categorías (§2.1, §4.1) · editar o anular ventas (§2.3) ·
varias monedas (D-05) · auditoría `created_at/updated_at` (§8) · atributos extra de producto, como
descripción, SKU o código (DP-03) · desglose del reporte por vendedor (DP-02).

## Supuestos

| # | Supuesto | Motivo |
|---|---|---|
| S-05 | **Permisos por rol:** el administrador gestiona catálogo, usuarios y reportes; el vendedor registra ventas y consulta | El modelo fija el *alta de vendedores* por admin (DP-04), pero no el resto de permisos |
| S-06 | **Columnas del reporte:** producto, etiqueta de categoría congelada, cantidad total e importe total | El modelo indica agregación por producto con orden por importe; el índice incluye `quantity` y `unit_price` (§6.2) |
| S-07 | **Mensajes de error de login** no distinguen usuario inexistente de clave incorrecta | El modelo no lo especifica; es práctica estándar |

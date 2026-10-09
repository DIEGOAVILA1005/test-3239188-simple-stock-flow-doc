# Requisitos no funcionales — Simple Stock Flow

> Derivados de `spec/data-model.md`. Cada requisito cita su fuente. El modelo **no fija umbrales
> numéricos** (latencia, volumen, disponibilidad): donde un requisito sería cuantitativo, se declara
> como **supuesto** y no se inventa una cifra.

---

## Integridad de los datos

| ID | Requisito | Fuente |
|---|---|---|
| **NFR-01** | `product.stock` **nunca** es negativo, y esa garantía vive **en el motor** (`ck_product_stock_non_negative`), no solo en la aplicación | §2.2, ADR-002 |
| **NFR-02** | Las invariantes marcadas *solo dominio* (`price > 0`, `quantity > 0`, nombre de categoría no vacío, `role ∈ {admin, seller}`, `username` en minúsculas) **bajan al motor** como `CHECK`. Mientras tanto, se declaran como deuda: un `INSERT` por `psql` las salta | §4, T-20 |
| **NFR-03** | Toda relación entre tablas tiene política explícita de borrado: FK-1 y FK-3 `RESTRICT`, FK-2 `CASCADE`, FK-4 `RESTRICT` (pendiente). `ON UPDATE NO ACTION` en todas | §5 |
| **NFR-04** | Nada se borra físicamente salvo el binario de imagen: productos con baja lógica; ventas, líneas, usuarios y categorías sin borrado | §7.1, ADR-003 |
| **NFR-05** | Las ventas son **inmutables**: no existe operación de edición ni de borrado | §2.3 |

## Concurrencia y consistencia

| ID | Requisito | Fuente |
|---|---|---|
| **NFR-06** | La concurrencia sobre el stock es **optimista**: el conflicto se detecta por `xmin` y el segundo escritor no pisa al primero | D-04, ADR-002, §3 |
| **NFR-07** | Registrar una venta descuenta stock y añade la línea como **una sola operación**, dentro de una transacción de base | §2.3 |
| **NFR-08** | **No se promete** atomicidad entre la base y el almacenamiento externo de imágenes; el orden es: anular la clave, confirmar, y **después** borrar el binario | §7.1 |
| **NFR-09** | Todas las marcas de tiempo son `timestamptz` y el servidor corre en UTC | §3 |
| **NFR-10** | El sistema es **monomoneda**: ninguna tabla tiene columna de moneda. `Money` redondea a 2 decimales (`AwayFromZero`) y coincide con `numeric(18,2)`; si uno cambia, cambia el otro en la misma migración | D-05, §2.2 |

## Seguridad y privacidad

| ID | Requisito | Fuente |
|---|---|---|
| **NFR-11** | `password_hash` es un **secreto**: jamás en logs, respuestas, proyecciones ni mensajes de error, y **nunca se indexa** | §7 |
| **NFR-12** | El dominio **nunca ve la clave en claro**; el hash lo produce un puerto | D-09, §2.5 |
| **NFR-13** | `username` y `sold_by` son **datos personales**: acceso restringido, no admisibles en respuestas anónimas | §7 |
| **NFR-14** | La superficie de privacidad es **mínima**: no hay datos de cliente final, pagos ni salud, y el reporte no se desglosa por vendedor | §7, DP-02 |
| **NFR-15** | El rol `admin` **no se otorga en ejecución**; se provisiona desde el entorno. El administrador inicial no se siembra desde SQL ni con un hash literal versionado | §9.2, DP-04, art. IX |

## Rendimiento

| ID | Requisito | Fuente |
|---|---|---|
| **NFR-16** | Los patrones de **alta frecuencia** (búsqueda de producto Q1, lectura por lote Q3, ventas por rango Q7, reporte Q9, usuario por nombre Q10) están **respaldados por índices** que existen o tienen tarea | §6.1, §6.2 |
| **NFR-17** | La **búsqueda de texto parcial** usa un índice de trigramas, porque ningún árbol B sirve un comodín a la izquierda. La extensión `pg_trgm` se instala **dentro** de la migración del índice | §6.2, T-13 |
| **NFR-18** | El **reporte agregado se calcula en el motor**, no en memoria, y su agregación no toca la tabla (índice con columnas incluidas) | D-06, §6.2 |
| **NFR-19** | Los índices que no se justifican **se descartan por escrito** (`stock`, `role`, `image_key`, `password_hash`…): un índice de más encarece cada escritura | §6.3 |

> **S-08.** El modelo no define metas cuantitativas (milisegundos, usuarios concurrentes, volumen).
> Aquí no se fijan; si se necesitan, las define el propietario.

## Mantenibilidad y operación

| ID | Requisito | Fuente |
|---|---|---|
| **NFR-20** | El esquema lo poseen **solo las migraciones de EF** (incluidas las extensiones); infraestructura levanta el motor, no define el DDL | ADR-001, §3.2, §6.2 |
| **NFR-21** | **Ninguna columna tiene `DEFAULT`**: los valores los pone el dominio, nunca el motor | §3 |
| **NFR-22** | Tablas en **singular**, esquema `sales` intacto; los objetos que EF genera conservan su estilo y los `CHECK` van en `snake_case` (`ck_{tabla}_{regla}`) | §0, §3.1 |
| **NFR-23** | **Sin columnas de auditoría** (`created_at`/`updated_at`); si aparece un requisito real, vuelve al propietario | §8 |
| **NFR-24** | Las categorías se siembran con **identificadores fijos** en la migración inicial; el administrador inicial lo crea el arranque con credenciales de entorno | §9.1, §9.2 |

## Verificabilidad

| ID | Requisito | Fuente |
|---|---|---|
| **NFR-25** | El modelo de datos debe poder **contrastarse con el motor** repitiendo las consultas de §10; si no coinciden, **gana el motor** y el documento está roto | §10, art. X |
| **NFR-26** | Toda regla se **clasifica** como *motor*, *solo dominio* o *pendiente*; lo que no está aplicado se declara, no se promete | § "Cómo se lee", §13 |
| **NFR-27** | Las mentiras conocidas del documento se registran como **deuda declarada** (D-1, D-2, D-3) en vez de dejarse silenciosas | §13 |

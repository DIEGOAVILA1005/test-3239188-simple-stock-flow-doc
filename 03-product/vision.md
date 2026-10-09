# Visión — Simple Stock Flow

> Reconstruida desde `spec/data-model.md`. Refleja lo que el modelo **ya compromete**; no añade
> funcionalidades. Lo no sustentado se marca como **supuesto**.

---

## Declaración de visión

> **Para** el equipo operativo de un negocio con catálogo (vendedores y administrador),
> **que necesita** vender sin pasarse del stock y saber después qué se vendió,
> **Simple Stock Flow** es un **sistema de control de catálogo, stock y ventas**
> **que** hace cumplir las reglas en el dato mismo, deja ventas inmutables y entrega un reporte
> estable por producto.
> **A diferencia de** una hoja de cálculo o de reglas que solo vive en la aplicación,
> **nuestro sistema** garantiza en el motor lo que puede garantizarse y **declara por escrito** lo que
> aún no.

## Principios

| # | Principio | Qué significa | Fuente |
|---|---|---|---|
| P-1 | **Simple a propósito** | Cinco entidades, cinco tablas, "sin excedente"; un producto tiene nombre, precio, stock, categoría e imagen y nada más | §2, DP-03 |
| P-2 | **El motor manda** | Si el documento contradice a la base, el documento está roto | Intro, art. X |
| P-3 | **Lo que no se garantiza, se declara** | Cada regla: *motor*, *solo dominio* o *pendiente*. Deuda declarada ≠ trampa silenciosa | § "Cómo se lee", §13 |
| P-4 | **El pasado no cambia** | Ventas inmutables y valores congelados; el reporte de un rango cerrado no se mueve | §1, §2.3, §11.1 |
| P-5 | **Nunca se pierde historia** | Baja lógica de productos; nada de borrado físico | §7.1, ADR-003 |
| P-6 | **Datos personales mínimos** | Sin cliente final; el reporte no se desglosa por vendedor | §7, DP-02 |
| P-7 | **Una sola fuente de verdad** | Sin `DEFAULT` en el motor; los valores los pone el dominio | §3 |

## Qué entrega el producto

1. **Catálogo** con búsqueda, categorías fijas, baja lógica e imagen opcional (US-03 a US-08).
2. **Ventas** que descuentan stock de forma atómica, congelan lo vendido y no se editan (US-09 a US-11).
3. **Reporte** agregado por producto y rango, calculado en el motor y estable (US-12).
4. **Acceso** con dos roles, sin alta anónima y sin otorgar `admin` en ejecución (US-01, US-02).

## Qué NO es

Sin clientes ni compradores · sin pagos ni multimoneda · sin CRUD de categorías · sin edición ni
anulación de ventas · sin auditoría de cambios · sin desglose por vendedor.
*(Todo decidido y registrado: §1, §2.1, §2.3, §7, §8, D-05, DP-02, DP-03.)*

## Cómo sabremos que funciona

El modelo **no define métricas numéricas** (ver S-08 en `04-requirements/non-functional.md`). En su
lugar fija criterios **verificables**:

| Criterio | Cómo se comprueba | Fuente |
|---|---|---|
| Ninguna venta deja el stock negativo | `ck_product_stock_non_negative` rechaza el `INSERT`/`UPDATE` | §2.2 |
| El reporte de un rango cerrado no cambia | Misma consulta, mismo resultado tras ventas nuevas | §11.1 |
| El documento no miente | Las consultas de §10 devuelven lo que §3, §4 y §6.2 afirman | §10 |
| Cada deuda está registrada | El registro §13 cuadra con las marcas del documento | §13 |

## Supuestos

| # | Supuesto |
|---|---|
| S-10 | El producto es **de uso interno** del negocio; no tiene tienda en línea ni acceso del comprador (coherente con la ausencia de entidad cliente, §1) |
| S-09 | El rubro del negocio no se declara; ver `problem-framing.md` |

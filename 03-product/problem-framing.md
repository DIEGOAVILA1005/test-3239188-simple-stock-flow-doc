# Planteamiento del problema — Simple Stock Flow

> Reconstruido **hacia atrás** desde `spec/data-model.md`: el problema es el que ese modelo hace
> necesario resolver. Lo que el modelo no afirma se marca como **supuesto**.

---

## 1. El problema

Un negocio que vende artículos de un catálogo necesita **saber cuántas unidades tiene**, **vender sin
pasarse de lo que tiene** y **saber después qué vendió y cuánto**. Sin una herramienta que haga
cumplir esas reglas, el stock se desajusta, las ventas no dejan huella confiable y el reporte
depende de a quién se le pregunte.

El modelo de datos delata el problema en tres frentes:

| Dolor | Cómo lo resuelve el modelo | Fuente |
|---|---|---|
| **Vender lo que no hay** | `stock >= 0` garantizado por el motor; la venta falla si el stock no alcanza | §2.2, ADR-002 |
| **Que el pasado cambie** | La línea de venta **congela** nombre, precio y categoría; la venta es inmutable | §1 *Nombre congelado*, §2.3, §2.4 |
| **No saber qué se vende más** | Reporte agregado por producto sobre un rango de fechas, estable y calculado en el motor | §1, D-06, §11.1 |

## 2. Quién lo padece

| Persona | Qué necesita | Fuente |
|---|---|---|
| **Vendedor** (`seller`) | Registrar ventas rápido y con el catálogo correcto | §1, §2.5 |
| **Administrador** (`admin`) | Mantener catálogo y usuarios; ver reportes | §1, §11 H-3 |

**Quién no es parte del problema:** el comprador. El sistema **no tiene entidad cliente**; registra al
**operador interno** que hizo la venta (§1, §7).

> **S-09.** El tipo de negocio no se declara. Las cinco categorías sembradas (*General, Herramientas,
> Electricidad, Fontanería, Pinturas*, §9.1) sugieren un comercio de suministros o ferretería, pero
> es una **inferencia**, no un dato del modelo.

## 3. Por qué los remedios obvios no bastan

| Remedio obvio | Por qué falla | Fuente |
|---|---|---|
| Validar el stock solo en la aplicación | Un `psql` o una migración futura lo salta **sin ruido**; protege a la aplicación, no a los datos | § "Cómo se lee", §1 |
| Leer precio y nombre del catálogo al consultar una venta | Renombrar o repreciar reescribe el histórico | §1 *Nombre congelado* |
| Borrar productos que ya no se venden | Rompe líneas de venta y reportes | §7.1, ADR-003, FK-3 |
| Elegir "la etiqueta más reciente" en el reporte | Una venta nueva cambiaría lo ya leído de un rango cerrado | §11.1 |

## 4. Qué NO es el problema

Se decidió dejar fuera, y no se reabre sin un requisito real:

- **Clientes y compradores** (§1) · **pagos o tarjetas** (§7) · **varias monedas** (D-05)
- **Mantenimiento de categorías** (§2.1, §4.1) · **editar o anular ventas** (§2.3)
- **Auditoría de cambios** (§8) · **atributos extra de producto** (DP-03)
- **Análisis por vendedor** (DP-02)

## 5. Restricciones que condicionan la solución

- **Honestidad sobre lo que está garantizado:** cada regla se declara *motor*, *solo dominio* o
  *pendiente*; no se promete lo que no existe (§ "Cómo se lee", §13).
- **Superficie de privacidad pequeña** (§7).
- **El motor manda:** si el documento contradice a la base, el documento está roto (§ intro, art. X).

## 6. Pregunta que el producto debe responder

> *¿Puedo registrar una venta con la certeza de que el stock es correcto, que el registro no
> cambiará nunca, y que luego puedo saber qué se vendió?*

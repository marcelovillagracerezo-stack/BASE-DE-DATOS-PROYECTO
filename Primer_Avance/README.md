# Práctica — Sistema para Ferretería (Desarrollo)

## 1) Narración de requisitos (Descripción del problema y solución)

Al analizar las necesidades de una ferretería local, identifiqué un problema crítico: el negocio lleva el registro de su inventario y ventas en cuadernos, lo que provoca un descontrol total. Se desconoce el stock real de los artículos, se pierden los datos de contacto de quienes suministraron la mercadería, y el cálculo manual al momento de cobrar retrasa la atención al cliente.

Para resolver esto, la base de datos gestionará a los **Proveedores** (para tener sus datos exactos); estructurará el inventario agrupando los **Productos** en **Categorías** (registrando precio y stock real); registrará las **Compras** para aumentar el inventario cuando llega mercadería nueva; y registrará cada **Venta** al mostrador, almacenando el detalle de los artículos vendidos para generar un ticket rápido. El sistema calculará los totales automáticamente y actualizará el stock (sumando en compras y restando en ventas), asegurando que los datos siempre coincidan con la cantidad física en los estantes.

## 2) Suposiciones (decisiones para aclarar ambigüedades)

* Las ventas son al contado. Se registrará el nombre y, de forma opcional, el NIT del cliente en la tabla Venta para el ticket.
* Un producto pertenece a una sola Categoría.
* Asumiremos que un producto específico es suministrado por un único Proveedor principal (Relación 1:N entre Proveedor y Producto) para simplificar el modelo de esta práctica.
* Las relaciones de Venta y Compra con los Productos son de Muchos a Muchos (N:M), por lo que se requieren tablas intermedias (`DETALLE_VENTA` y `DETALLE_COMPRA`).
* Las claves primarias de las tablas de detalle serán compuestas (ej. `id_venta` + `id_producto`) para evitar registrar el mismo producto en líneas separadas de un mismo ticket; en su lugar, se suma la cantidad en una sola línea.

## 3) Identificación de Entidades, Atributos, Tipos y PK

```text
┌─────────────────────────────────────────────────┐
│ PROVEEDOR                                       │
├─────────────────────────────────────────────────┤
│ + id_proveedor: INTEGER PK (AUTOINCREMENT)      │
│ + razon_social: VARCHAR(100) NOT NULL           │
│ + nombre_contacto: VARCHAR(100)                 │
│ + telefono: VARCHAR(20) NOT NULL                │
│ + email: VARCHAR(100)                           │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ CATEGORIA                                       │
├─────────────────────────────────────────────────┤
│ + id_categoria: INTEGER PK (AUTOINCREMENT)      │
│ + nombre: VARCHAR(50) UNIQUE NOT NULL           │
│ + descripcion: VARCHAR(255)                     │
└─────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│ PRODUCTO                                                                 │
├──────────────────────────────────────────────────────────────────────────┤
│ + id_producto: INTEGER PK (AUTOINCREMENT)                                │
│ + codigo_barras: VARCHAR(50) UNIQUE NOT NULL                             │
│ + nombre: VARCHAR(100) NOT NULL                                          │
│ + id_categoria: INTEGER FK → CATEGORIA(id_categoria) NOT NULL            │
│ + id_proveedor: INTEGER FK → PROVEEDOR(id_proveedor) NOT NULL            │
│ + precio_unitario: DECIMAL(10,2) NOT NULL CHECK (precio_unitario > 0)    │
│ + stock_actual: INTEGER NOT NULL DEFAULT 0 CHECK (stock_actual >= 0)     │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│ COMPRA                                                                   │
├──────────────────────────────────────────────────────────────────────────┤
│ + id_compra: INTEGER PK (AUTOINCREMENT)                                  │
│ + id_proveedor: INTEGER FK → PROVEEDOR(id_proveedor) NOT NULL            │
│ + fecha_hora: DATETIME DEFAULT CURRENT_TIMESTAMP                         │
│ + total_compra: DECIMAL(10,2) NOT NULL DEFAULT 0                         │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│ DETALLE_COMPRA                                                           │
├──────────────────────────────────────────────────────────────────────────┤
│ + id_compra: INTEGER FK → COMPRA(id_compra)                              │
│ + id_producto: INTEGER FK → PRODUCTO(id_producto)                        │
│ + cantidad: INTEGER NOT NULL CHECK (cantidad > 0)                        │
│ + costo_unitario: DECIMAL(10,2) NOT NULL                                 │
│ + subtotal: DECIMAL(10,2) NOT NULL                                       │
│ ** PK COMPUESTA: (id_compra, id_producto) **                             │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│ VENTA                                                                    │
├──────────────────────────────────────────────────────────────────────────┤
│ + id_venta: INTEGER PK (AUTOINCREMENT)                                   │
│ + fecha_hora: DATETIME DEFAULT CURRENT_TIMESTAMP                         │
│ + nombre_cliente: VARCHAR(100) DEFAULT 'Consumidor Final'                │
│ + nit_cliente: VARCHAR(20)                                               │
│ + total_venta: DECIMAL(10,2) NOT NULL DEFAULT 0                          │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│ DETALLE_VENTA                                                            │
├──────────────────────────────────────────────────────────────────────────┤
│ + id_venta: INTEGER FK → VENTA(id_venta)                                 │
│ + id_producto: INTEGER FK → PRODUCTO(id_producto)                        │
│ + cantidad: INTEGER NOT NULL CHECK (cantidad > 0)                        │
│ + precio_unitario: DECIMAL(10,2) NOT NULL                                │
│ + subtotal: DECIMAL(10,2) NOT NULL                                       │
│ ** PK COMPUESTA: (id_venta, id_producto) **                              │
└──────────────────────────────────────────────────────────────────────────┘
```

## 4) Relaciones y cardinalidades

* **PROVEEDOR (0..N) — (1) PRODUCTO:** Un proveedor suministra varios productos.
* **CATEGORIA (0..N) — (1) PRODUCTO:** Una categoría agrupa varios productos.
* **PROVEEDOR (0..N) — (1) COMPRA:** Un proveedor nos puede realizar muchas ventas (que para nosotros son compras).
* **COMPRA (1..N) — (1) DETALLE_COMPRA:** Una compra tiene múltiples detalles.
* **PRODUCTO (0..N) — (1) DETALLE_COMPRA:** Un producto puede ser reabastecido en muchas compras.
* **VENTA (1..N) — (1) DETALLE_VENTA:** Una venta incluye múltiples productos (detalles).
* **PRODUCTO (0..N) — (1) DETALLE_VENTA:** Un producto puede aparecer en muchas ventas.

## 5) Reglas de negocio y restricciones importantes

* **Validación de Stock previo a la Venta:** Antes de insertar un registro en `DETALLE_VENTA`, el sistema verificará que exista stock. Si `cantidad > stock_actual`, la venta será rechazada.
* **Actualización Bidireccional del Inventario (Triggers):**
  * Al confirmar un `DETALLE_VENTA` (salida), se restará la cantidad del `stock_actual`.
  * Al confirmar un `DETALLE_COMPRA` (ingreso), se sumará la cantidad al `stock_actual`.
  * En implementaciones futuras, operaciones de `UPDATE` o `DELETE` sobre los detalles deberán ajustar la diferencia del stock para mantener la consistencia.
* **Congelamiento de Precios:** Los precios (`costo_unitario` en compras y `precio_unitario` en ventas) se copian al momento de la transacción para mantener el historial intacto frente a futuros cambios de tarifa.
* **Cálculo Automático de Atributos Derivados:** El usuario no ingresa los totales a mano. El `subtotal` se calcula estrictamente como (`cantidad` × `precio`). Posteriormente, `total_venta` y `total_compra` se actualizan automáticamente sumando los subtotales de sus respectivos detalles.

## 6) Diagrama Entidad-Relación (DER)

El modelo conceptual de la base de datos se ha diseñado utilizando la Notación de Chen clásica, contemplando las entidades de compras, ventas, inventario y respetando las reglas de normalización (3NF).

📄 **[Haz clic aquí para ver el Diagrama Entidad-Relación completo en PDF](./Diagrama_ER_Ferreteria.pdf)**

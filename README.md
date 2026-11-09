# ERP para la Bodega de Alimentos "Frescura Total"

Sistema de planificación de recursos empresariales (ERP) de consola, desarrollado en
**Java 17 + Maven** con persistencia en **MySQL** a través de **JDBC**, pensado para la
gestión integral de una bodega de alimentos perecederos.

---

## Descripción

**Frescura Total S.A.S.** es una comercializadora de alimentos frescos cuya operación gira
en torno a un activo crítico y altamente perecedero: el inventario. Este ERP automatiza y
conecta los cuatro procesos centrales del negocio en una sola aplicación de escritorio
(consola):

- **Inventario y Calidad** (módulo central): control de lotes con su fecha de vencimiento,
  estados del lote (Disponible, Próximo a Vencer, Vencido, En Cuarentena), registro de
  cuarentena, inspecciones de calidad y registro de mermas.
- **Proveedores y Compras**: maestro de proveedores, catálogo de productos y la relación
  producto–proveedor (qué proveedor surte qué producto y a qué precio).
- **Ventas y Facturación**: clientes (mayoristas/minoristas), registro de ventas con
  selección de lotes por política **FEFO** (*First-Expired, First-Out*), aplicación
  automática de **promociones del 50 %** a lotes próximos a vencer, cálculo de **IVA**
  diferenciado y **generación de la factura con un clic** en formato **PNG y PDF**, cada una
  con **código QR** y un **CUFE simulado**.
- **Logística y Entregas**: conductores, vehículos, rutas de entrega, asignación de rutas y
  la relación ventas–ruta para el despacho.

El inventario es el eje: las ventas descuentan existencias de los lotes, la logística
despacha esas ventas y la calidad retira del inventario lo que no es apto. Todo el sistema
está construido sobre una arquitectura limpia que separa responsabilidades y facilita el
mantenimiento.

---

## Autores

- **Nicolás Sierra Bermúdez**

---

## Requerimientos Previos

Para compilar y ejecutar el proyecto necesitas tener instalado:

| Herramienta        | Versión recomendada |
|--------------------|---------------------|
| **Java JDK**       | 17 (LTS)            |
| **Apache NetBeans**| 12 o superior       |
| **MySQL Server**   | 8.x                 |
| **Apache Maven**   | 3.8+ (NetBeans ya incluye uno embebido) |

> El proyecto compila sin necesidad de una base de datos en ejecución; MySQL solo es
> necesario para **usar** la aplicación en tiempo de ejecución.

---

## Configuración de la Base de Datos

1. **Instala y arranca MySQL 8** en tu máquina local (`localhost`, puerto `3306`).

2. **Crea la base de datos y carga el esquema + los datos de prueba.** El archivo
   `docs/database.sql` contiene **dos secciones**:
   - **(A) DDL**: `CREATE DATABASE erp_frescura_total`, todas las tablas y los catálogos
     esenciales ya poblados (estados de lote, tipos de cliente, estados de venta,
     categorías de producto e información de la empresa).
   - **(B) DML**: más de 50 registros de prueba coherentes por cada tabla principal
     (proveedores, productos, clientes, lotes, ventas, etc.), con integridad referencial
     garantizada y fechas de vencimiento variadas (lotes disponibles, próximos a vencer y
     vencidos).

   Ejecuta el script completo desde la terminal:

   ```bash
   mysql -u root -p < docs/database.sql
   ```

   O bien, desde el cliente de MySQL / MySQL Workbench, abre `docs/database.sql` y ejecútalo.
   El script crea la base de datos `erp_frescura_total` desde cero (no necesitas crearla a
   mano).

3. **Credenciales de conexión.** Los valores de conexión están centralizados en la clase
   **`com.frescuratotal.erp.common.DatabaseConfig`**. Los valores de ejemplo son:

   | Parámetro  | Valor por defecto        |
   |------------|--------------------------|
   | Host       | `localhost`              |
   | Puerto     | `3306`                   |
   | Base datos | `erp_frescura_total`     |
   | Usuario    | `root`                   |
   | Contraseña | `root`                   |

   Si tu MySQL usa otro usuario o contraseña, edita únicamente esa clase y vuelve a compilar.
   Si la base de datos no está disponible, la aplicación **no se cae**: muestra un mensaje
   claro en español indicando que no se pudo conectar y cómo solucionarlo.

---

## Instalación y Ejecución

### Opción A — Apache NetBeans (recomendada)

1. Clona el repositorio:
   ```bash
   git clone https://github.com/Nicolas0129/frescura-total-erp.git
   ```
2. Abre NetBeans → **File → Open Project** y selecciona la carpeta del proyecto (NetBeans lo
   reconoce como *Java with Maven → Java Application*).
3. Compila: **Run → Clean and Build** (o clic derecho sobre el proyecto → *Clean and Build*).
4. Ejecuta: **Run → Run Project** o pulsa **F6**. El menú principal aparecerá en la ventana
   de salida (*Output*) de NetBeans.

### Opción B — Terminal (Maven)

```bash
# 1. Compilar y empaquetar el JAR ejecutable (fat JAR con dependencias)
mvn -q clean package

# 2. Ejecutar la aplicación
java -jar target/frescura-total-erp.jar
```

El `maven-shade-plugin` genera `target/frescura-total-erp.jar`, un JAR autocontenido
(incluye el driver JDBC de MySQL, ZXing para el QR y PDFBox para el PDF) cuya clase principal
es `com.frescuratotal.erp.Main`.

> Las facturas generadas (PNG/PDF) se guardan en la carpeta `invoices/` creada
> automáticamente en tiempo de ejecución.

---

## Estructura del Proyecto

El proyecto sigue el patrón **MVC organizado por componentes (módulos de negocio)**. En lugar
de agrupar todos los *models*, *views* y *controllers* en tres carpetas gigantes, cada módulo
funcional es autocontenido y expone sus propios sub-paquetes `controller/`, `model/` y
`view/`. Esto mantiene alta cohesión dentro de cada dominio y bajo acoplamiento entre
módulos.

```
com/frescuratotal/erp/
├── common/                 Utilidades transversales
│   ├── DatabaseConnection      (Singleton de conexión JDBC)
│   ├── DatabaseConfig          (credenciales y URL de conexión)
│   ├── DaoFactory              (Factory de DAOs)
│   ├── GenericDao              (contrato DAO base genérico)
│   ├── DaoException            (excepción de capa de datos)
│   ├── NumberGenerator         (números de lote / venta / factura)
│   ├── ConsoleUtils            (entrada/salida de consola)
│   ├── AppLogger               (logging a archivo logs/erp.log)
│   ├── CompanyInfo / CompanyInfoDao
│   └── invoice/                (generación de factura: Factory PNG/PDF, QR, CUFE)
│
├── suppliers/              Proveedores y Compras
│   ├── controller/  SupplierController
│   ├── model/       Supplier, Product, Category (+ DAO: interfaz + impl JDBC)
│   └── view/        SupplierView
│
├── inventory/             Inventario y Calidad (módulo central)
│   ├── controller/  InventoryController
│   ├── model/       Lot, LotStatus, QuarantineRecord, QualityInspection, WasteRecord (+ DAOs)
│   └── view/        InventoryView
│
├── sales/                 Ventas y Facturación
│   ├── controller/  SalesController
│   ├── model/       Customer, CustomerType, Sale, SaleDetail, Invoice, Promotion (+ DAOs)
│   └── view/        SalesView
│
├── logistics/            Logística y Entregas
│   ├── controller/  LogisticsController
│   ├── model/       Driver, Vehicle, DeliveryRoute, DeliveryAssignment, RouteSale (+ DAOs)
│   └── view/        LogisticsView
│
└── Main.java             Punto de entrada; menú principal de consola
```

**Convención de idioma:** todo el **código** (clases, métodos, variables y comentarios) está
en **inglés** con `PascalCase` para clases y `camelCase` para métodos/variables. Los DAOs se
encargan de **mapear los nombres de columna en español** del esquema MySQL a los **campos en
inglés** de las entidades. Todos los **mensajes dirigidos al usuario** (menús de consola,
avisos, errores) están en **español**.

Los diagramas del diseño están en `docs/`:
- `docs/diagrama_er.png` — diagrama entidad-relación de la base de datos.
- `docs/diagrama_clases.png` — diagrama de clases UML de las entidades del paquete `model/`.

---

## Principios y Patrones de Diseño

### Principios SOLID

El diseño aplica los cinco principios SOLID de forma transversal. El **Principio de
Responsabilidad Única (SRP)** se refleja en la separación MVC: los *models* representan datos
y reglas de la entidad, los *controllers* orquestan los casos de uso y los *views* solo se
ocupan de la interacción por consola; ninguna clase mezcla acceso a datos con lógica de
presentación. El registro de eventos también respeta SRP al quedar centralizado en una única
utilidad (`AppLogger`), de modo que los controladores no se ocupan de cómo ni dónde se escribe
la bitácora. El **Principio de Inversión de Dependencias (DIP)** y el **Principio de
Sustitución de Liskov (LSP)** se materializan en la capa DAO: los controladores dependen de
**interfaces** (`SupplierDao`, `LotDao`, `SaleDao`, …), no de las implementaciones JDBC
concretas, de modo que cualquier implementación que respete el contrato es intercambiable.

El **Principio Abierto/Cerrado (OCP)** se aprecia en la generación de facturas: se pueden
añadir nuevos formatos de documento implementando la interfaz `InvoiceDocumentGenerator` sin
modificar el código existente que la consume. Por último, el **Principio de Segregación de
Interfaces (ISP)** mantiene los contratos pequeños y enfocados (un `GenericDao` con las
operaciones CRUD esenciales y DAOs específicos que añaden solo las consultas que su entidad
necesita), evitando que las clases dependan de métodos que no usan.

### Patrón DAO (Data Access Object)

Toda la persistencia se encapsula en la capa DAO. Para cada entidad existe una **interfaz**
(por ejemplo `SupplierDao`) que define las operaciones disponibles y una **implementación
JDBC** (`SupplierDaoImpl`) que contiene el SQL concreto. Un contrato base genérico,
`GenericDao<T, K>`, normaliza las operaciones CRUD comunes. Este patrón aísla por completo la
lógica de negocio del detalle de SQL y JDBC: si se cambiara el motor de base de datos o la
forma de consultar, solo se tocarían las implementaciones. Además, es aquí donde ocurre el
**mapeo español→inglés**: el DAO lee las columnas en español (`nombre_proveedor`, `nit`,
`fecha_vencimiento`, …) desde el `ResultSet` y las asigna a los campos en inglés de las
entidades (`name`, `taxId`, `expirationDate`, …).

### Patrón Singleton

La clase **`DatabaseConnection`** implementa el patrón Singleton para garantizar que exista
**una única instancia** responsable de administrar la conexión JDBC a MySQL durante toda la
vida de la aplicación. Centralizar la conexión evita abrir conexiones redundantes, reutiliza
los parámetros definidos en `DatabaseConfig` y ofrece un punto único para gestionar errores
de conexión de forma controlada (mostrando un mensaje claro en español en lugar de propagar
una excepción que tumbe el programa).

### Patrón Factory

El patrón Factory se usa en dos lugares. Primero, **`DaoFactory`** centraliza la creación de
los DAOs: los controladores piden un DAO a la fábrica (`DaoFactory.supplierDao()`,
`DaoFactory.lotDao()`, …) y reciben la implementación adecuada, sin acoplarse a los
constructores concretos. Segundo, **`InvoiceGeneratorFactory`** decide, según el formato
solicitado, qué generador de documento instanciar (`PngInvoiceGenerator` o
`PdfInvoiceGenerator`), ambos implementando la interfaz común `InvoiceDocumentGenerator`.
Esto permite ofrecer la **factura con un clic en PNG y PDF** (con QR y CUFE simulado) y, en el
futuro, agregar nuevos formatos sin modificar el código que solicita la factura — una
combinación directa de los patrones Factory y Open/Closed.

---

## Programación funcional y manejo de logs

### Stream API y expresiones lambda

La lógica de negocio de los controladores aprovecha de forma intensiva la **Stream API** y las
**expresiones lambda** de Java, además de las interfaces funcionales estándar (`Predicate`,
`Function`, `Comparator`, `Consumer`). Algunos ejemplos reales dentro del código:

- **Inventario (`InventoryController`)**:
  - `runDailyReview()` clasifica los lotes con `stream().filter(...)` usando un `Predicate`
    para separar los que deben marcarse *Vencido* o *Próximo a Vencer*, y aplica el cambio con
    `forEach(...)`.
  - `countLotsByStatus()` agrupa y cuenta los lotes por estado con
    `Collectors.groupingBy(...)` + `Collectors.counting()`.
  - `lotsExpiringWithin(dias)` filtra por fecha y ordena en modo FEFO con
    `sorted(Comparator.comparing(Lot::getExpirationDate))`.
  - `totalStockValue()` y `totalWasteUnits()` agregan valores con `map(...).reduce(...)`.
- **Ventas (`SalesController`)**:
  - `suggestFefoLot(...)` selecciona el lote que primero vence con
    `min(Comparator.comparing(Lot::getExpirationDate))` (regla FEFO explícita).
  - `confirmSale(...)` calcula subtotal, descuentos, IVA y total con un helper funcional
    `sumBy(..., SaleDetail::getLineTotal)` basado en `map(...).reduce(...)`.
  - `totalSalesAmount()` y `countSalesByStatus()` producen reportes agregados por streams.
- **Proveedores (`SupplierController`)**: `productsOfSupplier(...)` resuelve el catálogo con
  `map(productDao::findById).flatMap(Optional::stream)`, y `countProductsByCategory()` agrupa
  productos por categoría.
- **Logística (`LogisticsController`)**: `completeRoute(...)` marca y cuenta las ventas
  entregadas con `filter(...).count()`, y `pendingDispatchValue()` suma el valor pendiente de
  despacho con `map(Sale::getTotalAmount).reduce(...)`.

Estos reportes son visibles para el usuario en las nuevas opciones de menú **"Reporte de
inventario"** y **"Reporte de ventas"**.

### Manejo de archivos y registro de eventos (logging)

La clase **`com.frescuratotal.erp.common.AppLogger`** centraliza el registro de eventos a
**archivo** usando `java.util.logging` sobre un `FileHandler` en modo *append*. La bitácora se
guarda en **`logs/erp.log`**; la carpeta `logs/` se crea automáticamente con `java.nio.file.Files`
si no existe. Cada línea incluye **marca de tiempo y nivel** (`INFO`, `WARNING`, `SEVERE`), por
ejemplo:

```
[2026-11-02 09:15:04] INFO    Venta confirmada: V-000123 cliente=12 lineas=3 total=85000.00
[2026-11-02 09:16:40] WARNING Inspeccion #7 RECHAZADA para lote 55. Merma de 20 unidades.
```

Se registran las operaciones clave del negocio: recepción y loteo de mercancía, cambios de
estado del *job* de revisión diaria, cuarentenas, inspecciones y mermas, confirmación de
ventas, generación de facturas, creación y cierre de rutas, y los **errores de base de datos**
capturados en los módulos. `AppLogger` también ofrece `tail(n)` para leer las últimas líneas
del archivo con `java.nio.file.Files` (lectura de archivos). El archivo `logs/erp.log` está
excluido del control de versiones (ver `.gitignore`); lo que se versiona es el **código** que
lo genera.

---

## Historial de versiones

| Fecha | Cambio |
|-------|--------|
| 2026-10-30 | Versión inicial con los cuatro módulos y creación del CHANGELOG. |
| 2026-10-31 | Javadoc en las entidades `Lot`, `Product` y `Supplier`. |
| 2026-11-01 | Validaciones numéricas con mensajes en español. |
| 2026-11-02 | Stream API, lambdas, logging a archivo y uniformización de mensajes. |
| 2026-11-03 | Entrega final: CHANGELOG y README actualizados. |

Consulta el detalle completo en [CHANGELOG.md](CHANGELOG.md).

## Licencia

Proyecto académico desarrollado por Nicolás Sierra Bermúdez. Uso educativo.

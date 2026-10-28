# 🚀 Proyecto RappiPlus: De Datos a Decisiones de Negocio (End-to-End Analytics)

## 📌 Descripción del Proyecto
Este proyecto desarrolla una solución analítica integral (**End-to-End**) para evaluar la salud financiera, el comportamiento de los usuarios dentro de la plataforma y el impacto de experimentos de producto en el servicio **RappiPlus** (con operaciones en México, Colombia y Argentina).

El flujo de trabajo combina **Python (Pandas, SciPy)**, **SQL (PostgreSQL)** y **Business Intelligence (Power BI)** a través de 6 etapas críticas:
1. 🔍 **Calidad de Datos y ETL (Python):** Auditoría, imputación y limpieza de registros transaccionales y de marketing.
2. 💰 **Rentabilidad y Unit Economics (Python):** Evaluación de ingresos (*Revenue*), costo de ventas (*COGS*), inversión publicitaria y utilidad neta (*Profit*).
3. 🛒 **Embudo de Conversión / Funnel Analysis (SQL):** Identificación de puntos de abandono en el flujo de compra.
4. 🔁 **Retención por Cohortes (SQL):** Seguimiento semanal de la permanencia de usuarios tras su registro.
5. 🧪 **Experimentación A/B (Estadística Inferencial):** Validación de impacto en cambios de interfaz (*Checkout UI*) mediante prueba Chi-cuadrado ($\chi^2$).
6. 📊 **Visualización Ejecutiva (Power BI):** Construcción de un dashboard interactivo con vistas *Overview* y *Drill-through*.

---

## 🛠️ Tecnologías y Librerías Utilizadas
* **Lenguajes:** Python 3, SQL (PostgreSQL)
* **Librerías de Python:** `pandas`, `numpy`, `scipy.stats` (`chi2_contingency`), `sqlalchemy`, `matplotlib`, `seaborn`
* **Visualización y BI:** Microsoft Power BI Desktop (Modelado en Estrella, DAX y tablas de fechas dinámicas)

---

## 🧹 1. Auditoría y Limpieza de Datos (ETL en Python)
Se procesaron tres fuentes principales (`rappiplus_orders_raw.csv`, `rappiplus_catalog.csv` y `rappiplus_marketing_spend.csv`), aplicando las siguientes reglas de negocio para pasar de **25,100 registros crudos a 24,906 órdenes limpias y validadas**:

* **Eliminación de duplicados:** Detección y remoción de 100 pedidos duplicados en `id_pedido`.
* **Conversión temporal:** Transformación de `fecha_hora_pedido` y `fecha` a formato `datetime64[ns]` (periodo enero – junio 2025).
* **Homologación de variables categóricas:** Estandarización de nombres de países (`str.title()` para unificar `"mexico"` y `"Mexico"`) y normalización de acentos en `categoria_producto` (`"Electrónica"` ➔ `"Electronica"`) para garantizar un *join* exacto entre pedidos y catálogo.
* **Imputación inteligente en Marketing:** Recuperación de **101 registros con canal nulo** extrayendo el prefijo directamente desde `id_campaña` (*organic, paid_search, social*).
* **Saneamiento numérico y control de *Outliers*:** Filtrado de cantidades y precios negativos o en cero, y eliminación de valores atípicos extremos en `cantidad` (pedidos erróneos de 10,000 y 20,000 unidades, acotando a `cantidad <= 50`).
* **Recálculo de consistencia financiera:** Validación matemática de la columna `monto_total` mediante la fórmula `(cantidad * precio_unitario) - monto_descuento`.

---

## 💰 2. Análisis de Rentabilidad y Ventas

Al integrar las órdenes limpias con los costos unitarios del catálogo de proveedores y el gasto de marketing, se obtuvieron los siguientes indicadores financieros del semestre:

### 📊 Desempeño Financiero y Operativo
| Métrica / Indicador (KPI) | Resultado |
| :--- | :--- |
| **Ingreso Total (Revenue)** | **$9,610,019.78** |
| **Costo de Productos (COGS)** | **$3,828,869.01** |
| **Inversión Total en Marketing** | **$2,871,843.53** |
| **Costo Total Operativo (COGS + Mkt)** | **$6,700,712.54** |
| **Utilidad Neta (Profit)** | **$2,909,307.24** *(Margen neto: ~30.27%)* |
| **Ticket Promedio por Orden (AOV)** | **$385.85** |
| **Cantidad Promedio por Orden** | **1.50 unidades** |
| **Producto Líder en Ventas** | **`Vacuum-Pro-Black`** (6,284 unidades vendidas) |

### 📣 Distribución del Gasto en Marketing por Canal
* **Social:** $976,818.37 (34.01%)
* **Organic:** $972,650.96 (33.87%)
* **Paid Search:** $922,374.20 (32.12%)

> **Diagnóstico Financiero:** El negocio es **altamente rentable**, generando más de **$2.90 millones en utilidad neta** tras cubrir el costo de mercancía y la inversión publicitaria en los tres países.

---

## 🛒 3. Análisis del Funnel de Conversión (SQL)
Mediante consultas SQL con funciones de ventana (`LAG() OVER`) sobre la tabla `events` en PostgreSQL, se evaluó el recorrido de los usuarios a través de las etapas clave del proceso de compra:

| Etapa del Funnel (`nombre_evento`) | Usuarios Únicos | Conversión Paso a Paso (%) | Conversión Acumulada Total (%) |
| :--- | :---: | :---: | :---: |
| **1. `first_visit`** | 7,796 | 100.00% | 100.00% |
| **2. `add_to_cart`** | 7,634 | 97.92% | 97.92% |
| **3. `add_payment_info`** | 6,250 | **81.87%** ⚠️ | 80.17% |
| **4. `purchase`** | 6,240 | 99.84% | **80.04%** |

*(Nota: Eventos intermedios complementarios registrados en plataforma: `select_item` con 7,582 usuarios y `begin_checkout` con 7,208 usuarios).*

* **Hallazgo Clave:** La tasa de conversión global es sumamente sólida (**80.04%**). Sin embargo, el **principal cuello de botella ocurre en la transición hacia `add_payment_info`**, donde se pierde el **18.13%** de los usuarios que ya habían agregado productos al carrito (1,384 usuarios abandonan antes de registrar su método de pago). Una vez ingresados los datos de pago, el **99.84%** finaliza la compra.

---

## 🔁 4. Análisis de Retención por Cohortes (SQL)
Se cruzaron las tablas `users` y `user_activity` en PostgreSQL utilizando *Common Table Expressions (CTEs)* y truncamiento de fechas (`DATE_TRUNC`) para evaluar la recurrencia semanal de las cohortes adquiridas entre enero y mayo de 2025:

* **Estructura de Cohortes:** Se analizaron 5 cohortes mensuales (con un volumen de entre **1,444 y 1,687 usuarios registrados por mes**).
* **Comportamiento longitudinal:** El seguimiento durante las primeras 3 semanas posteriores al registro (`semana_1`, `semana_2`, `semana_3`) muestra una cobertura constante de actividad registrada en la base de seguimiento para los usuarios dados de alta en cada periodo.

---

## 🧪 5. Experimentación A/B: Rediseño de Checkout UI
Para evaluar si una nueva interfaz en la pantalla de pago mejoraba la conversión final (`experiment_checkout_ui.csv`), se diseñó y ejecutó una prueba de hipótesis rigurosa:

* **Hipótesis Nula ($H_0$):** La tasa de conversión es igual entre el grupo **Control** y el grupo **Tratamiento** (el cambio en la UI no tiene efecto significativo).
* **Hipótesis Alternativa ($H_1$):** La tasa de conversión difiere significativamente entre ambos grupos.
* **Prueba Estadística:** Prueba **Chi-cuadrado de independencia ($\chi^2$)** mediante `scipy.stats.chi2_contingency`, con un nivel de significancia $\alpha = 0.05$.
* **Resultado y Decisión de Producto:** No se encontró evidencia estadísticamente significativa para rechazar $H_0$ ($p \ge 0.05$). **Se recomienda NO desplegar el cambio de interfaz en producción**, ahorrando costos de ingeniería y redirigiendo los esfuerzos de UX a simplificar la fricción específica en la carga de métodos de pago (`add_payment_info`).

---

## 📊 6. Arquitectura del Dashboard en Power BI
Los datasets depurados (`orders_clean.csv`, `catalog_clean.csv` y `marketing_clean.csv`) se integraron en **Power BI** bajo un modelo relacional conectado a una tabla calendario dinámica (`dim_fecha`), estructurado en dos vistas:

1. **Vista 1 — Overview Ejecutivo:**
   * Tarjetas de KPIs principales: *Revenue Total, Profit Total, Gasto en Marketing, Ticket Promedio* y *Unidades por Orden*.
   * Gráficos de líneas con la evolución mensual de *Revenue / Profit* y desempeño acumulado **YTD (Year-to-Date)**.
   * Comparativa de ingresos y rentabilidad desglosada por categoría (*Hogar, Moda, Electrónica*) y producto.
2. **Vista 2 — Detalle Operativo y Drill-Through:**
   * Tabla granular a nivel orden/producto con **formato condicional tipo semáforo** sobre el margen de beneficio (*Profit*).
   * Ranking de volumen de unidades vendidas por SKU (liderado por `Vacuum-Pro-Black` y `Blender-XL-Red`).
   * Funcionalidad **Drill-Through** que permite seleccionar cualquier producto en la vista general y navegar directamente al historial detallado de sus pedidos.

---

## 🎯 7. Recomendaciones Estratégicas de Negocio
1. **Optimización de la Etapa de Pago (`add_payment_info`):** Dado que la mayor fuga del funnel (18.13%) ocurre al ingresar los datos de pago y el rediseño visual general (A/B test) no alteró la conversión, se recomienda integrar pasarelas de pago en un clic (*Apple Pay, Google Pay, billeteras locales*) y auditar posibles errores técnicos de validación de tarjetas.
2. **Reasignación Basada en ROAS por Canal:** Con un gasto de marketing distribuido en partes casi iguales (~$922k – $976k por canal), se sugiere cruzar el costo de adquisición contra el margen neto generado por cada `fuente_referencia` para escalar el presupuesto en los productos de mayor margen unitario.
3. **Estrategia de *Cross-Selling* para Elevar Unidades por Orden:** Actualmente el promedio es de **1.50 productos por pedido**. Implementar paquetes promocionales (*bundles*) alrededor de los productos ancla (`Vacuum-Pro-Black` y `Blender-XL-Red`) permitirá incrementar el ticket promedio ($385.85) diluyendo el costo fijo de adquisición.

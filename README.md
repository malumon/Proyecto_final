# Proyecto final - Data Analytics

# Análisis del Marketplace Olist

## 1. Introducción

Este proyecto analiza el comportamiento del marketplace brasileño Olist a partir de datos de ventas, clientes, productos, vendedores, pagos, valoraciones y captación de nuevos vendedores.

El análisis combina procesos de extracción y revisión de datos, modelado relacional, limpieza y transformación con Python, análisis exploratorio de datos (EDA) y construcción de un dashboard interactivo en Power BI.

### Objetivo del proyecto

El objetivo es obtener una visión global del funcionamiento del marketplace, identificar patrones relevantes en las ventas y el comportamiento de los clientes, analizar el impacto de la logística sobre la satisfacción y estudiar el proceso de captación y conversión de nuevos vendedores.

Los resultados del análisis se utilizan como base para la construcción de un dashboard ejecutivo que permita visualizar los principales indicadores y hallazgos del negocio.

---

## 2. Fuentes de datos

1. **Brazilian E-Commerce Public Dataset by Olist** — https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce?resource=download
2. **Marketing Funnel by Olist** — https://www.kaggle.com/datasets/olistbr/marketing-funnel-olist

Para cada tabla se siguió siempre el mismo esquema de revisión: shape, `info()`, nulos, duplicados, valores únicos de las claves y primeras conclusiones.

### 2.1. Dataset Ecommerce — resumen por tabla

| Tabla                     | Registros                                        | Notas clave                                                                                                                                                                                                                                                                                                                                              |
| ------------------------- | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **customers**             | 99.441, 5 variables                              | Sin nulos ni duplicados. `customer_id` es único (99.441). `customer_unique_id` tiene 96.096 valores únicos → un mismo cliente puede tener varios `customer_id`; será la clave para identificar clientes únicos.                                                                                                                                          |
| **orders**                | 99.441 pedidos, 8 variables                      | `order_id` único. Columnas de fecha en texto (a convertir a datetime). Nulos en `order_approved_at`, `order_delivered_carrier_date`, `order_delivered_customer_date`, asociados al estado del pedido (no error de calidad). Mayoría `delivered`; resto: shipped, canceled, processing, invoiced, approved, created, unavailable.                         |
| **order_items**           | 112.650, 7 variables                             | Sin nulos ni duplicados. `shipping_limit_date` a convertir a datetime. Más filas que `orders` → un pedido puede tener varios productos (tabla de detalle). 98.666 pedidos únicos (menos que en `orders`, a revisar). Contiene precio, coste de envío y vendedor de cada artículo.                                                                        |
| **order_payments**        | 103.886, 5 variables                             | Sin nulos ni duplicados. 99.440 pedidos únicos (casi la totalidad). Más registros que pedidos → un pedido puede tener varios pagos. Método predominante: tarjeta de crédito, luego boleto, voucher y tarjeta de débito (`not_defined`: solo 3 casos).                                                                                                    |
| **order_reviews**         | 99.224, 7 variables                              | Sin duplicados. `review_creation_date` y `review_answer_timestamp` a convertir a datetime. Muchos nulos en título/mensaje de la reseña (cliente valoró sin comentar, no es problema de calidad). 98.673 pedidos únicos → no todos los pedidos fueron valorados. Predominan valoraciones de 5 estrellas.                                                  |
| **products**              | 32.951 productos, 9 variables                    | Sin duplicados. `product_id` único. 610 productos con nulos en categoría, longitud de nombre/descripción y nº de fotos (mismas filas → información incompleta en origen). Solo 2 productos con nulos en variables físicas (peso/dimensiones). 73 categorías. Algunas variables numéricas en float por presencia de nulos, aunque son cantidades enteras. |
| **sellers**               | 3.095 vendedores, 4 variables                    | Sin nulos ni duplicados. `seller_id` único. Fuerte concentración en São Paulo (SP), y a distancia Paraná (PR), Minas Gerais (MG), Santa Catarina (SC), Río de Janeiro (RJ) y Río Grande do Sul (RS).                                                                                                                                                     |
| **geolocation**           | 1.000.163 registros, 5 variables (la más grande) | Sin nulos. 261.831 duplicados completos. Solo 19.015 códigos postales únicos → un mismo código postal tiene múltiples registros. No es clave primaria, requiere agregación previa antes de integrarse al modelo.                                                                                                                                         |
| **product_category_name** | 71 registros, 2 variables                        | Sin nulos ni duplicados. Tabla de correspondencia portugués–inglés de categorías.                                                                                                                                                                                                                                                                        |

### 2.2. Dataset Marketing — resumen por tabla

| Tabla                               | Registros                  | Notas clave                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ----------------------------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **marketing_qualified_leads (MQL)** | 8.000 leads, 4 variables   | Sin duplicados. `mql_id` único. `first_contact_date` a convertir a datetime. 60 nulos en `origin`. Principales canales: organic_search, paid_search y social.                                                                                                                                                                                                                                                                              |
| **closed_deals**                    | 842 acuerdos, 14 variables | `mql_id` único y relaciona con MQL. `won_date` a convertir a datetime. Pocos nulos en `business_segment`, `lead_type`, `business_type`; muchos nulos en `has_company`, `has_gtin`, `average_stock`, `declared_product_catalog_size` (a evaluar su utilidad). Segmentos principales: home_decor, health_beauty, car_accessories. Predomina `online_medium`, luego `online_big` e `industry`. Mayoría tipo `reseller`, luego `manufacturer`. |

---

## 3. Modelo relacional propuesto

### 3.1. Tabla principal: `order_items`

Representa el mayor nivel de detalle de una venta (una fila = un producto dentro de un pedido). Permite analizar simultáneamente ventas por producto, categoría y vendedor, ingresos, costes de envío, comportamiento de clientes, métodos de pago y valoraciones. Si se usara `orders` como base, se perdería el detalle de productos en pedidos con varios artículos.

```text
CUSTOMERS (customer_id)
   │ 1:N
   ▼
ORDERS (order_id)
   │ 1:N
   ▼
ORDER_ITEMS (tabla principal)
   ├── PRODUCTS (product_id) → PRODUCT_CATEGORY_TRANSLATION
   ├── SELLERS (seller_id)
   ├── ORDER_PAYMENTS (order_id)
   └── ORDER_REVIEWS (order_id, vía orders)
```

### 3.2. Relaciones y estrategia de integración (LEFT JOIN, base `order_items`)

| Tabla origen | Tabla destino                | Clave                 | Relación | Unión |
| ------------ | ---------------------------- | --------------------- | -------- | ----- |
| order_items  | orders                       | order_id              | N:1      | LEFT  |
| orders       | customers                    | customer_id           | N:1      | LEFT  |
| order_items  | products                     | product_id            | N:1      | LEFT  |
| products     | product_category_translation | product_category_name | N:1      | LEFT  |
| order_items  | sellers                      | seller_id             | N:1      | LEFT  |
| order_items  | order_payments               | order_id              | 1:N      | LEFT* |
| orders       | order_reviews                | order_id              | 1:0..1   | LEFT  |

*`order_payments` requiere tratamiento previo (un pedido puede tener varios pagos), por lo que se mantiene como tabla independiente.

### 3.3. Geolocation

No se incorpora directamente: 1.000.163 registros pero solo 19.015 códigos postales, con múltiples coordenadas por código postal → un merge directo duplicaría masivamente el dataset. Se agregó previamente a un registro por código postal (ver más abajo).

### 3.4. Bloque de marketing

```text
Marketing Qualified Leads (mql_id) → Closed Deals (seller_id) → Sellers
```

`closed_deals` conecta ambos datasets vía `seller_id`. Representa un proceso de negocio distinto al de las ventas, por lo que se mantiene como modelo complementario (embudo de marketing).

---

## 4. Limpieza y transformación

* Todas las transformaciones se hicieron sobre **copias** de los DataFrames originales, para preservar los datos fuente y poder reproducir el proceso desde cero.

### 4.1. Conversión de fechas (`pd.to_datetime()`, `errors="coerce"`)

| Tabla                     | Columnas convertidas                                                                                                                    |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| orders                    | order_purchase_timestamp, order_approved_at, order_delivered_carrier_date, order_delivered_customer_date, order_estimated_delivery_date |
| order_items               | shipping_limit_date                                                                                                                     |
| order_reviews             | review_creation_date, review_answer_timestamp                                                                                           |
| marketing_qualified_leads | first_contact_date                                                                                                                      |
| closed_deals              | won_date                                                                                                                                |

### 4.2. Revisión de tipos de datos

En `closed_deals`, `has_company`, `has_gtin` y `average_stock` **no se convirtieron** a booleano/categórica: los dos primeros tienen >92% de nulos (convertir a `False` distorsionaría el análisis) y `average_stock` solo tiene información para una pequeña proporción de registros. Criterio general: solo se transforma lo que aporta valor real.

### 4.3. Tratamiento de nulos por tabla

* **products:** 610 registros con nulos en categoría y variables descriptivas (aparecen en 1.603 líneas de `order_items`, por lo que no se eliminan). Se imputó `product_category_name` nulo como **"Unknown"**; el resto de nulos descriptivos se mantuvo sin imputar (sin base objetiva para estimarlos). 2 productos adicionales con nulos en peso/dimensiones se conservaron igual.
* **orders:** nulos en fechas logísticas corresponden mayormente a pedidos cancelados, en proceso o no entregados (no es error de calidad). Se detectaron 8 pedidos `delivered` sin fecha de entrega —inconsistencia puntual documentada, sin imputar.
* **order_reviews:** nulos en comentario/título no son error de calidad, sino que el cliente valoró sin escribir opinión (se dan en todos los rangos de `review_score`). No se imputaron.
* **geolocation → geolocation_master:** se creó una tabla maestra con **un registro por código postal** (19.015 filas), calculando latitud/longitud media y la moda de ciudad/estado por código postal, para evitar duplicados en las uniones.

---

## 5. Construcción del dataset analítico

### 5.1. Integración progresiva sobre `order_items`

1. **+ orders** (order_id, 1:N, left): se mantienen las 112.650 líneas originales.
2. **+ customers** (customer_id, 1:N, left): se añade info geográfica del cliente; las 112.650 filas quedan con customer_id no nulo (integridad referencial correcta).
3. **+ products** (product_id, 1:N, left): se usa la categoría ya imputada como "Unknown"; aumento esperado de nulos descriptivos al replicarse por cada venta de un mismo producto incompleto.
4. **+ product_category_translation** (product_category_name, 1:N, left): 2 categorías sin traducción original (`pc_gamer` y `portateis_cozinha_e_preparadores_de_alimentos`, 24 nulos) se completaron manualmente.
5. **+ sellers** (seller_id, 1:N, left): se añade info geográfica del vendedor.
6. **+ geolocation_master** (seller_zip_code_prefix = geolocation_zip_code_prefix): se detectaron discrepancias puntuales entre ciudad/estado de `sellers` y de `geolocation_master` (errores de escritura, abreviaturas, códigos postales sin match). Se conservaron **ambas** variables al no poder determinar cuál es correcta en todos los casos.

### 5.2. Selección de variables del dataset de ventas

Se eliminaron variables que no respondían a las preguntas de negocio del proyecto (no por "quitar por quitar", sino para mantener un dataset analítico enfocado):

* **Fechas/logística interna:** order_approved_at, order_delivered_carrier_date, shipping_limit_date (fuera del foco comercial del proyecto).
* **Categoría original:** product_category_name (se sustituyó por su traducción al inglés).
* **Descriptivas de producto:** product_name_lenght, product_description_lenght, product_photos_qty (sin relación con las preguntas de negocio).
* **Físicas de producto:** product_weight_g, product_length_cm, product_height_cm, product_width_cm (orientadas a logística, fuera de alcance).
* **Códigos postales:** customer_zip_code_prefix, seller_zip_code_prefix, geolocation_zip_code_prefix (ciudad/estado y coordenadas ya dan el detalle geográfico necesario).
* **Geolocalización redundante:** geolocation_city, geolocation_state (se prefirió conservar la ciudad/estado declarados por cliente/vendedor y usar solo las coordenadas para análisis geográfico).

**Variables finales del dataset de ventas (19 variables):**

| Grupo                 | Variables                                                                                            |
| --------------------- | ---------------------------------------------------------------------------------------------------- |
| Identificación        | order_id, order_item_id, product_id, seller_id, customer_id, customer_unique_id                      |
| Información económica | price, freight_value                                                                                 |
| Estado y fechas       | order_status, order_purchase_timestamp, order_delivered_customer_date, order_estimated_delivery_date |
| Producto              | product_category_name_english                                                                        |
| Cliente               | customer_city, customer_state                                                                        |
| Vendedor              | seller_city, seller_state                                                                            |
| Geolocalización       | geolocation_lat, geolocation_lng                                                                     |

### 5.3. Tablas mantenidas independientes (distinta granularidad)

* **order_payments:** una fila por pago (puede haber varios por pedido). Sin nulos ni tipos erróneos. Se conservaron sin modificar 2 registros con `payment_installments = 0` y 9 con `payment_value = 0` (proporción mínima, sin evidencia de error). Se eliminó `payment_sequential` por no aportar valor analítico (se puede reconstruir agrupando por `order_id`).
* **order_reviews:** una fila por reseña de pedido. Algunos `review_id` asociados a más de un `order_id` correspondían a pedidos distintos y simultáneos con la misma valoración (se conservaron). Se eliminaron `review_comment_title` y `review_comment_message` (no hay análisis de texto/NLP en el proyecto) y `review_answer_timestamp` (proceso interno de la plataforma).

### 5.4. Bloque de marketing

* **marketing_qualified_leads:** `mql_id` único, sin duplicados. Se eliminó `landing_page_id` (identificador técnico sin tabla de referencia para interpretarlo). Se conservó `origin` pese a sus 60 nulos (0,75%) por su relevancia para analizar canales de captación.
* **closed_deals:** `mql_id` único, sin duplicados. Se eliminaron `sdr_id` y `sr_id` (empleados internos de Olist, fuera de alcance), y `has_company`, `has_gtin`, `average_stock`, `declared_product_catalog_size` (>90% nulos). También se eliminó `declared_monthly_revenue` (~95% de los valores en cero, poco útil para segmentar). Se conservaron `business_segment`, `lead_type`, `lead_behaviour_profile` y `business_type` (perfil de vendedores), con sus nulos sin imputar.
* **Integración → `df_marketing`:** LEFT JOIN por `mql_id`, base = marketing_qualified_leads (se conservan los 8.000 leads). Tras la unión se mantienen 8.000 registros. Los nulos en las columnas provenientes de `closed_deals` indican que ese lead no se convirtió en vendedor (no son errores).

---

## 6. Análisis Exploratorio de Datos (EDA)

**Objetivo:** comprender el comportamiento comercial del marketplace Olist (ventas, clientes, vendedores, métodos de pago, valoraciones y embudo de captación de vendedores), como base para el dashboard.

### 6.1. Bloque 1. Rendimiento de ventas

* **Evolución temporal:** comparando enero–agosto de 2017 vs. 2018 (se excluyó septiembre, con solo 1 pedido en 2018): los pedidos subieron **136,96%** (22.614 → 53.574) y las ventas **139,42%** (3.610.270,15 € → 8.643.531,14 €). El ticket medio apenas varió (159,09 € → 160,74 €, +1,03%). El crecimiento viene del **volumen de pedidos**, no del gasto medio.
* **Categorías:** mayor venta en health_beauty (1.441.248,07 €, 9,22%), watches_gifts (1.305.541,61 €, 8,35%) y bed_bath_table (1.241.681,72 €, 7,94%). `computers` tiene el ticket medio más alto (1.286,18 €) pero poca participación (1,49%) por bajo volumen. Correlación pedidos–ventas: **r = 0,97**. Concentración: top 5 = 39,68% de ventas, top 10 = 63,15%, top 17 = 80,83%.
* **Geografía:** São Paulo (SP) lidera con 41.375 pedidos y 5,92M € (37,38% de ventas). Junto a Río de Janeiro (13,44%) y Minas Gerais (11,72%) suman 62,53%; los 7 estados principales llegan a 80,89%. El ticket medio varía por estado (SP: 143,12 €; Bahía: 182,10 €) pero no determina el volumen. Por ciudad, la distribución es mucho más diversificada (se necesitan 386 ciudades para el 80%). Correlación pedidos–ventas: **r = 0,9985** (estados) / **r = 0,9966** (ciudades).
* **Vendedores:** el mayor vendedor facturó 249.640,70 €; otro, con solo 358 pedidos, logró un ticket medio de 658,82 €. Los 10 principales representan el 12,80% de ventas, los 20 principales el 20,85%; se necesitan 562 vendedores (18,16% del total) para el 80%. Correlación pedidos–ventas: **r = 0,83**. Ticket medio: mediana 128,82 € vs. media 221,39 € (desv. est. 358,62 €, máx. 6.922,21 €) → distribución sesgada por vendedores con tickets muy altos.
* **Métodos de pago:** tarjeta de crédito 75,24% de pedidos, boleto 19,46% (94,70% entre ambos). En importe: tarjeta ≈12,54M €, boleto ≈2,87M €. Importe medio: tarjeta 163,94 €, boleto 145,03 €, voucher 98,15 €. 2.246 pedidos con dos métodos de pago; 97.194 con uno solo (el importe se analizó por método, no por pedido único).

### 6.2. Bloque 2. Comportamiento de los clientes

* **Distribución y recurrencia:** SP concentra el 41,88% de los clientes (correlación clientes–ventas por estado: **r = 0,9985**), aunque no de forma perfectamente proporcional (SP tiene más peso en clientes que en ventas; en RJ ocurre lo contrario). El **96,95%** de los clientes hizo un solo pedido; solo el 3,05% repitió (mediana y P75 = 2 pedidos entre recurrentes; máximo 16 pedidos de un mismo cliente).
* **Valor por cliente:** media de ventas por cliente 166,04 € vs. mediana 107,94 € (asimetría = 9,09, por clientes de valor muy alto). No hay concentración extrema: se necesitan 46.506 clientes (48,74% de la base) para el 80% de ventas. Los clientes recurrentes generan de media 310,49 € (casi el doble de los 161,49 € de los de un solo pedido), pese a tener un ticket medio algo menor (146,85 € vs. 161,49 €) — el mayor valor viene de comprar más veces, no de gastar más por compra.

### 6.3. Bloque 3. Productos

* **Precio vs. rendimiento:** correlación precio–unidades vendidas prácticamente nula (**r = -0,0323**); precio–facturación positiva pero débil (**r = 0,279**). Productos entre 50–500 € concentran el 68,58% de la facturación (destaca 100–200 €, con 28,14%); >500 € suma 19,69%. El rendimiento depende de la combinación precio-volumen, no de una única estrategia.
* **Productos de alto desempeño** (top 25% simultáneo en unidades vendidas y facturación): 5.392 productos. Mayor proporción en el rango 100–200 € (26,33%), luego 200–500 € (21,76%) y 500–1000 € (18,74%); solo 7,38% en 0–50 €. El segmento 100–200 € destaca como especialmente interesante para el negocio.

### 6.4. Bloque 4. Experiencia y satisfacción del cliente

* **Satisfacción general:** media 4,09/5, mediana 5; 77,1% de valoraciones son de 4-5 estrellas. Pedidos `delivered`: valoración media 4,16. Correlación tiempo de entrega–valoración: **r = -0,3337** (a mayor tiempo, menor satisfacción). Los estados con menor tiempo de entrega tienden a mejor valoración. El importe del pedido y el número de unidades no muestran relación relevante con la satisfacción.
* **Satisfacción vs. comportamiento económico:** correlaciones prácticamente nulas entre valoración media y nº de pedidos (r = 0,01), ventas totales (r = -0,05) y ticket medio (r = -0,05). Clientes con 5 estrellas: 1,03 pedidos, 160,96 € de venta media; con 4 estrellas: 1,02 pedidos, 158,04 €; con 1 estrella: 198,81 € (¡superior!) — no hay patrón creciente. Recurrentes vs. no recurrentes: satisfacción casi igual (4,15 vs. 4,10), pero ventas medias muy distintas (310,10 € vs. 161,26 €), por mayor frecuencia de compra, no por ticket medio (que es inferior en recurrentes: 146,63 € vs. 161,26 €).

### 6.5. Bloque 5. Operaciones / Logística

* **Desempeño de entregas:** de 96.476 pedidos con fecha de entrega, tiempo medio 12,09 días (mediana 10; P75 = 15 días; máximo 209 días). El 93,38% se entregó a tiempo o antes; el 6,62% con retraso (retraso medio 10,62 días, mediana 7; 43,8% de los retrasos supera los 7 días y 18,7% supera los 15).
* **Factores de retraso:** mismo estado cliente-vendedor: 7,48 días de media (mediana 6) vs. estados distintos: 14,68 días (mediana 12). Por estado del cliente: de 8,30 días (SP) a 28,98 días (Roraima). Coste de transporte: relación positiva débil con el tiempo de entrega (**r = 0,17**). Nº de unidades del pedido: sin relación relevante (r = -0,02). La localización (envíos interestatales) es el factor más asociado al tiempo de entrega.
* **Impacto en satisfacción:** pedidos a tiempo o antes: valoración media 4,23; con retraso: 2,27 (diferencia de 1,96 puntos). Entre los retrasados, correlación duración del retraso–valoración: **r = -0,15** (débil). El cumplimiento del plazo estimado pesa más en la satisfacción que la magnitud concreta del retraso.

### 6.6. Bloque 6. Marketing (Vendedores)

* **Conversión global:** 8.000 leads, 842 convertidos → **10,52%** de conversión. Organic search: mayor volumen (2.296 leads), conversión 11,80% (32,73% de todas las conversiones). Paid search: 1.586 leads, conversión 12,30% (23,55% de conversiones). Entre ambos: 56,28% de las conversiones. Social: buen volumen (1.350 leads) pero solo 5,56% de conversión. Origen "unknown": mayor tasa de conversión (16,29%, 21,62% de conversiones), pero sin trazabilidad de canal, por lo que se interpreta con cautela.
* **Evolución temporal:** en 2017 la conversión sube gradualmente (0,84% en julio → 5,50% en diciembre). Desde enero de 2018 hay un salto (13,32% enero, 14,49% febrero, 14,22% marzo), coincidiendo con un fuerte aumento de volumen de leads (de cientos/mes en 2017 a >1.000/mes en 2018). Los últimos meses (abril–mayo 2018) deben leerse con cautela: el tiempo de conversión tiene mediana de 14 días, por lo que leads recientes aún no tuvieron tiempo de convertir (la caída a 9,98% en mayo no implica necesariamente peor conversión real). Se detectó un caso aislado con fecha de conversión anterior al primer contacto (inconsistencia puntual, no afecta la interpretación general).

---

## 7. Hallazgos principales

1. **El crecimiento del negocio viene del aumento de pedidos**, no del ticket medio (pedidos +136,96%, ventas +139,42%, ticket medio +1,03% entre 2017 y 2018).
2. **Las ventas se concentran en pocas categorías**: top 5 = 39,68%, top 10 = 63,15%, top 17 = 80,83%.
3. **Fuerte concentración geográfica a nivel estatal**: SP + RJ + MG = 62,53% de las ventas; top 7 estados = 80,89%.
4. **Las ventas por vendedor están relativamente diversificadas**: top 10 vendedores = solo 12,80%; se necesitan 562 vendedores para el 80% (contrapunto a la fuerte concentración geográfica).
5. **La recurrencia de clientes es el principal reto comercial**: 96,95% compra una sola vez, pero los recurrentes generan casi el doble de valor (310,49 € vs. 161,49 €) — gran potencial de fidelización.
6. **El precio no determina por sí solo el rendimiento**: correlación precio–unidades casi nula (r = -0,0323); el segmento 100–200 € concentra la mayor proporción de productos de alto desempeño.
7. **La logística es clave en la satisfacción**: entregas a tiempo 4,23/5 vs. retrasadas 2,27/5 — el cumplimiento del plazo pesa más que el valor económico del pedido.
8. **La distancia geográfica duplica el tiempo de entrega**: mismo estado 7,48 días vs. estados distintos 14,68 días.
9. **La satisfacción no se traduce en mayor valor económico**: correlaciones con pedidos (0,01), ventas (-0,05) y ticket medio (-0,05) prácticamente inexistentes — mejorar la satisfacción es importante, pero no hay evidencia de que aumente directamente las ventas.
10. **Marketing tiene margen de optimización**: conversión global 10,52%; organic_search y paid_search combinan volumen y eficiencia, social convierte poco (5,56%) pese al volumen; "unknown" tiene la mejor conversión pero sin trazabilidad de canal; fuerte mejora de conversión desde enero de 2018.

---

## 8. Dashboard Olist en Power BI

**Objetivo:** convertir los resultados del EDA en una herramienta ejecutiva, dividida en 4 páginas temáticas (para evitar un único dashboard sobrecargado):

* ¿Cómo está funcionando el negocio?
* ¿Cómo se comportan los clientes?
* ¿Cómo afecta la logística a la satisfacción?
* ¿Cómo convierten los leads en vendedores?

### 8.1. Construcción y corrección del modelo

* **Importes mal interpretados:** `price` y `freight_value` se importaron como texto y se convirtieron mal, generando ventas de ~644 millones (frente a los ~15,8M reales del EDA). Se corrigió eliminando el paso automático de cambio de tipo y convirtiendo con configuración regional English (United States) a número decimal. Tras la corrección, Ventas Totales = 15,8M, coincidente con el EDA.
* **Modelo temporal:** la relación inicial entre `Dim_Calendar[Date]` y `order_purchase_timestamp` fallaba porque esta última incluía hora. Se creó la columna `Fecha Compra` (solo fecha) y se estableció la relación `Dim_Calendar[Date] → df_ventas[Fecha Compra]`, habilitando filtrado por año/mes y el segmentador global.
* **Fechas de logística:** `order_purchase_timestamp`, `order_delivered_customer_date` y `order_estimated_delivery_date` estaban como texto, impidiendo usar `DATEDIFF()`. Se convirtieron en Power Query a Fecha/Hora y Fecha respectivamente.

### 8.2. Modelo analítico — tablas auxiliares

| Tabla                | Propósito                                                                |
| -------------------- | ------------------------------------------------------------------------ |
| Dim_Calendar         | Análisis temporal, segmentación por año, evolución mensual               |
| Dim_Orders           | Tabla puente entre pedidos y otras entidades                             |
| Clientes_Recurrencia | Granularidad 1 cliente = 1 fila, para frecuencia de compra y recurrencia |
| TipoCliente          | Comparar clientes Recurrentes vs. No recurrentes                         |
| TipoEnvio            | Comparar Mismo estado vs. Estado distinto                                |
| TipoEntrega          | Comparar A tiempo vs. Retrasada                                          |
| EmbudoMarketing      | Construcción del embudo Lead → Convertido                                |

Se desarrollaron ~30 medidas DAX, siguiendo la metodología: validar numerador → validar denominador → construir el indicador final.

### 8.3. Página 1 — Resumen Ejecutivo

**Pregunta:** ¿Cómo está funcionando el negocio?

* KPIs: Ventas Totales 15,8M | Pedidos Totales 99mil | Ticket Medio 160,58 | % Clientes Recurrentes 3,05% | % Entregas a Tiempo 91,89%.
* Evolución mensual de ventas y pedidos: confirma que el crecimiento se explica por el aumento de pedidos.
* Categorías por ventas: destacan health_beauty, watches_gifts y bed_bath_table.
* Estados por ventas: SP, RJ y MG lideran.

### 8.4. Página 2 — Clientes y Recurrencia

**Pregunta:** ¿Repiten los clientes?

* KPIs: Clientes Totales 95mil | Clientes Recurrentes 3mil | % Recurrentes 3,05% | Valor Medio Cliente Recurrente 310,49 €.
* Ventas clientes recurrentes: 904mil € | no recurrentes: 14,94M € (valor medio no recurrente: 161,49 €).
* Los recurrentes son ~3% de la base pero generan el doble de valor medio (310 € vs. 161 €).
* Visuales: valor medio por tipo de cliente; distribución de clientes por nº de pedidos (1, 2, 3, 4+); donut de ventas por tipo de cliente (94,3% no recurrente vs. 5,7% recurrente).

### 8.5. Página 3 — Operaciones y Satisfacción

**Pregunta:** ¿Cómo afecta la logística a la satisfacción?

* KPIs: % Entregas a Tiempo 91,89% | % Pedidos Retrasados 8,11% | Tiempo Medio de Entrega 12,50 días | Satisfacción Media 4,09.
* A tiempo: satisfacción 4,29 vs. retrasadas: 2,57 — hallazgo clave del proyecto.
* Distancia geográfica: mismo estado 7,86 días vs. estado distinto 14,99 días (casi el doble).
* Distribución de valoraciones de 1 a 5 estrellas.

### 8.6. Página 4 — Captación y Conversión de Vendedores

**Pregunta:** ¿Cómo convierten los leads en vendedores?

* KPIs: Leads Totales 8.000 | Leads Convertidos 842 | % Conversión 10,53% | Tiempo Medio de Conversión 49 días.
* Embudo de conversión: 8.000 leads → 842 convertidos.
* Conversión por origen: el origen "unknown" se renombró a **"Sin trazabilidad"** para evitar interpretaciones erróneas.
* Treemap de distribución de leads por canal.

### 8.7. Interactividad

Segmentador global por Año (`Dim_Calendar[Año]`), funcional entre Resumen Ejecutivo, Clientes y Recurrencia, y Operaciones y Satisfacción. La página de Captación y Conversión se mantuvo independiente porque usa un ciclo temporal distinto (`first_contact_date` y `won_date`).

### 8.8. Resultado final

El dashboard queda estructurado en 4 páginas complementarias que recorren el negocio de forma secuencial: **Negocio → Clientes → Operaciones → Captación**, cada una respondiendo una pregunta concreta y construyendo una narrativa coherente a partir de los hallazgos del EDA.

---

## 9. Conclusiones

El proyecto permite obtener una visión integrada del marketplace Olist desde diferentes perspectivas: rendimiento comercial, comportamiento de clientes, productos, experiencia de usuario, operaciones logísticas y captación de vendedores.

El análisis muestra que el crecimiento del negocio está principalmente impulsado por el aumento del volumen de pedidos, mientras que la recurrencia de clientes representa una oportunidad relevante de mejora. Asimismo, la logística tiene una relación clara con la satisfacción del cliente, especialmente en el cumplimiento de los plazos estimados.

Por último, el análisis del embudo de marketing permite identificar diferencias relevantes entre los canales de captación y oportunidades de optimización en la conversión de leads.

Los principales resultados del análisis se trasladan al dashboard desarrollado en Power BI, que permite explorar estos indicadores de forma interactiva y facilita una lectura ejecutiva de la información.

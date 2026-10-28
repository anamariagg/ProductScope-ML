# Limpieza y Transformación de Datos — Amazon Product Sales

> **Documento técnico de referencia** para la fase de modelado. Describe, paso a paso y con cifras verificables, todas las decisiones de limpieza aplicadas al dataset `amazon_sales_data_uncleaned.csv`.
> Todas las cifras provienen de la ejecución real del pipeline contenido en **`limpieza_4_datasets.ipynb`** (el notebook guía del proyecto).

---

## 1. Contexto y variable objetivo

El dataset contiene listings de productos de Amazon con atributos comerciales (precio, descuento, badges, envíos, reseñas). El interés analítico es **entender qué atributos del listing se asocian a un mayor volumen de ventas recientes**.

**Variable objetivo:** `bought_in_last_month` — número de unidades compradas en el último mes.

**Naturaleza del objetivo:** variable numérica **ordinal con límite inferior** (los valores de Amazon son rangos truncados: `50+`, `100+`, `1K+`, `2K+`…). El número reportado es el **límite inferior del intervalo real**, por lo que el valor crudo subestima sistemáticamente el volumen en la cola alta.

**Estrategia de modelado:**
- **Regresión sobre `log1p(bought_in_last_month)`.** Se aplica `ln(1 + x)` al target para comprimir la cola derecha, estabilizar la varianza y reducir la dominancia de los pocos productos virales. `log1p` evita el dominio inválido de `log(0)`.
- **Des-transformación** al predecir: `expm1 = e^pred − 1`.
- **Categoría ordinal auxiliar** (`bought_cat`) para conteos, estratificación de partición y pruebas de asociación. No es la etiqueta del modelo.

---

## 2. Inventario inicial

| Métrica | Valor |
|---|---|
| Filas | **42.675** |
| Columnas | **16** |
| Duplicados exactos | **962** |
| Filas tras eliminar duplicados | **41.713** |

### Nulos por columna (estado original)

| Columna | Nulos | % |
|---|---|---|
| `sustainability_badges` | 39.267 | 92,0 % |
| `buy_box_availability` | 14.653 | 34,3 % |
| `Current/discounted_price` | 11.749 | 27,5 % |
| `delivery_details` | 11.720 | 27,5 % |
| `bought_in_last_month` | 3.217 | 7,5 % |
| `product_url` | 2.069 | 4,8 % |
| `Rating` | 1.024 | 2,4 % |
| `Number_of_reviews` | 1.024 | 2,4 % |
| Resto (`Title`, `Listed_price`, `is_*`, `Price_on_variant`, `image_url`, `collected_at`) | 0 | 0 % |

### Cardinalidad

Alta cardinalidad en `product_url` (40.606), `Title` (8.808), `image_url` (8.038) → **identificadores**, no features. `sustainability_badges` tiene solo 16 valores distintos pero está dominada por nulos (92 %).

---

## 3. Variables excluidas

| Variable | Justificación |
|---|---|
| `image_url` | Identificador CDN (URL de imagen). Sin valor predictivo; 8.038 valores distintos sobre 42.675 filas. |
| `product_url` | URL con tokens de sesión y tracking. Es un identificador, no una feature. Además tiene 2.069 nulos. |
| `collected_at` | Timestamp de recolección del scraping (331 valores distintos). No aporta señal sobre el producto. |
| `Price_on_variant` | ~50 % nulos y, en las filas no nulas, muchas entradas son descriptores del producto (`basic variant price: 16 Count (Pack of 1)`) en lugar de precios. No fiable. |

> **Nota:** `Price_on_variant` también se excluye aunque el docente no lo mencionara originalmente; su calidad (mezcla de precios y descriptores) hace imposible reconstruir un precio numérico consistente.

---

## 4. Transformaciones por tipo de variable

### 4.1 Variable objetivo: `bought_in_last_month`

**Parseo estricto.** Se acepta únicamente el patrón `N[.K|M]+ bought in past month` con la regex:

```
^\s*([\d.]+)\s*([KM]?)\+ bought in past month\s*$
```

- Se multiplica por 1.000 (K) o 1.000.000 (M).
- **Todo lo que no cumple el patrón se trata como etiqueta no válida → NaN → fila eliminada** (ver §5).

**Por qué estricto:** el parseo original con `str.extract(r"(\d+)")` capturaba números basura dentro de etiquetas de texto (e.g. `($766.33$766.33/in)`, `Typical:`, `List:`), generando **2.654 valores falsos** (el problema de los "766" y demás). Con el patrón estricto esos registros quedan correctamente excluidos.

**Vista previa de lo que se elimina y por qué.** Antes de borrar ninguna fila, el notebook cuantifica y clasifica los registros no válidos:

| Motivo de descarte | Filas |
|---|---|
| Vacíos originales (NaN) | 3.217 |
| `No featured offers available` | 4.571 |
| `More Buying Choices` | 654 |
| `Typical:` | 532 |
| `List:` / `List Price:` | 518 + 511 |
| `Shop products from small business...` | 262 |
| `ESRB Rating: Everyone` | 107 |
| Precios por unidad (`($766.33$766.33/in)`, `($30.00$30.00/count)`) | 43 |
| Otros (Was:, ESRB, certificados…) | 106 |

**Conteo por rango (límite inferior, sobre filas válidas):**

| Rango | Registros |
|---|---|
| 50–100 | 14.768 |
| 101–200 | 5.602 |
| 201–500 | 6.628 |
| > 500 | 4.204 |

**Distribución por valor (los más frecuentes):** 100 → 8.801 · 50 → 5.967 · 200 → 5.602 · 300 → 2.841 · 500 → 2.351 · 1.000 → 1.787 · 400 → 1.436.

**Límite inferior vs. punto medio** (lo que pidió el docente):

| Estadístico | Límite inferior (`bought_raw`) | Punto medio (`bought_mid`) |
|---|---|---|
| Media | 944,87 | **1.171,99** |
| Mediana | 200,00 | **250,00** |
| Asimetría (skew) | 13,43 | 13,82 |

**Decisión: se usa el punto medio del intervalo** como valor del objetivo. Corrige el sesgo a la baja que el límite inferior introduce en la cola alta (la media sube ~24 %). La asimetría permanece alta (esperado, es inherente a la variable), por eso se **complementa con la transformación `log1p`** en la fase de modelado.

> **Visualización (sección 2.3b del notebook):** se generan histogramas comparando ambas versiones (escala completa y zoom ≤ 3.000) para mostrar cómo el punto medio desplaza los valores al centro de cada intervalo.

> **Categoría ordinal auxiliar** (`bought_cat`, usada para estratificar y para pruebas de asociación) — distribución final:
> | 50–100 | 101–200 | 201–500 | > 500 |
> |---|---|---|---|
> | 5.967 | 8.801 | 9.879 | 6.555 |

### 4.2 `Rating`

- **Parseo:** extracción del número del texto `"4.6 out of 5 stars"` → `4.6`.
- **Nulos imputados:** 76 registros restantes → **mediana = 4.50**.
- **Criterio:** la mediana es robusta a la distribución bimodal típica de ratings de Amazon; no se usa media porque arrastraría los extremos.

### 4.3 `Number_of_reviews`

- **Parseo:** eliminación de comas → numérico.
- **Nulos imputados:** 76 → **0**.
- **Criterio:** un producto sin reseñas registradas equivale a 0 reseñas; imputar con la media inflaría artificialmente el conteo. Es una ausencia que *significa* cero.

### 4.4 Precio: `Current/discounted_price` → `current_price`

> **⚠️ Decisión clave (cambió respecto a versiones anteriores): el precio NO se imputa en el pipeline base.**

- **Parseo:** eliminar comas → numérico.
- **Nulos: 6.766 (21,68 %)** se **conservan como `NaN`**. La decisión de qué hacer con ellos se difiere a la construcción de las variantes de dataset (ver §8), porque el precio **es específico de cada producto** e imputar la mediana introduce un dato falso (un monitor de $400 no cuesta $72).
- **Criterio:** se probarán dos estrategias por comparación empírica: **eliminar** las filas sin precio vs **imputar** la mediana. No se decide a ciegas.

**Atípicos (decisión clave):**
- Cálculo del IQR **sobre los valores no nulos**: Q1 = 24,99 · Q3 = 159,95 · IQR = 134,96 · límites = [−177,45 ; 362,39].
- **Atípicos marcados con `is_price_outlier`: 2.405 (7,71 %).**
- **NO se eliminan filas.** El precio es una *feature*, no el objetivo: un producto caro es un caso legítimo, no un error. Se conserva el valor real y se aporta la bandera; el `log1p` posterior comprime la cola para el modelo.
- La bandera se recalcula dentro de cada variante, ya que el conjunto de valores cambia según se elimine o se impute.

### 4.5 `Listed_price` → `has_discount`

- **Criterio:** no se imputa un precio de lista porque la ausencia de descuento es informativa por sí misma.

### 4.6 `is_best_seller` → `is_best_seller_flag`

- La columna original **no es booleana**: contiene 8 categorías (`No Badge`, `Amazon's`, `Limited time deal`, `Best Seller`, `Save 30%`, `Ends in`, `Save 10%`, `Save 18%`…).
- **Decisión:** se codifica `is_best_seller_flag = 1` solo para `Best Seller`; las demás categorías (ruido de scraping o badges distintos) quedan en 0.
- **Limitación documentada:** `Amazon's` (686) y `Limited time deal` (360) podrían portar señal, pero no son el badge "Best Seller". Se deja como tarea futura analizarlas por separado.

### 4.7 `is_sponsored` → `is_sponsored_flag`

- `1` si `== "Sponsored"`, si no `0`.

### 4.8 `is_couponed` → `has_coupon`

- `1` si `!= "No Coupon"`, si no `0`. La estructura de la cadena (`Save 15% with coupon`) no se usa por ahora; queda como posible feature futura.

### 4.9 `buy_box_availability` → `buy_box_status` (3 niveles)

| Valor | Registros |
|---|---|
| `add_to_cart` | 22.701 |
| `unknown` (nulo) | 8.501 |
| `no_buy_box` | 0 |

- **Criterio (el que pidió el docente):** la ausencia de dato se modela como **categoría propia `unknown`**, no se colapsa con "no comprable". En este dataset no hay registros con valor distinto de `Add to cart`; la única distinción real es **presente vs. ausente**. Se recomienda convertir a dummy (`buy_box_status_unknown`) al modelar.

### 4.10 `sustainability_badges` → prueba de asociación

**⚠️ Advertencia sobre el nombre:** la columna se llama `sustainability_badges`, pero su contenido es **heterogéneo**. Mezcla insignias de sostenibilidad reales con badges de compatibilidad de electrónica:

| Insignia | Registros | ¿Sostenibilidad? |
|---|---|---|
| Small Business | 1.341 | Sí |
| Carbon impact | 769 | Sí |
| Works with Alexa | 425 | **No** (compatibilidad) |
| Energy efficiency (+ variantes) | ~476 | Sí |
| Alexa Built-in | 184 | **No** (compatibilidad) |
| Manufacturing practices | 148 | Sí |
| Forestry practices | 18 | Sí |
| Safer chemicals (+ variantes) | ~24 | Sí |
| Recycled materials (+ variantes) | ~18 | Sí |
| Made in Italy | 4 | Sí |
| 1 sustainability certification | 1 | Sí |

Además, valores como `Energy efficiency +3 more` indican que la insignia principal viene acompañada de otras **no capturadas** (información truncada en el scraping).

**Consecuencia:** el flag binario `has_sustainability_badge` en realidad captura "tiene *algún* badge de producto". Parte de la asociación detectada podría venir de `Works with Alexa` / `Alexa Built-in` (609 registros), que correlacionan con productos populares del ecosistema Amazon y no con sostenibilidad.

**Prueba de asociación realizada:**

- **Nulos: 28.460 (91,2 %)** sobre las filas válidas.
- χ² de homogeneidad (con badge vs. sin badge) contra `bought_cat`:
  - χ² = **335,53** · p ≈ **0,0000** · gl = 3 · **Cramér's V = 0,1037**.
- **Resultado:** asociación **estadísticamente significativa pese a la baja cobertura**; tamaño de efecto **pequeño pero no trivial**.

**Decisión: RETENER** como flag binario `has_sustainability_badge`, **manteniendo el nombre original** (decisión explícita del grupo — Opción C), con esta documentación de sus limitaciones:
  - La variable solo describe al 8,8 % de los productos.
  - El nombre es impreciso: el flag captura "tiene algún badge de producto", no estrictamente sostenibilidad.
  - La significación se apoya en un tamaño muestral grande; el efecto real es modesto.
  - En la fase de modelado se evaluará su aporte marginal (importancia) y se descartará si no mejora el rendimiento.

### 4.11 `Title` → `title_word_count`

- Número de palabras del título (mínimo 2, media 20,7). Es un proxy de la "riqueza" del listing.
- El texto crudo del título no se vectoriza en esta fase.

### 4.12 `delivery_details` → `is_free_delivery`, `delivery_cost`

> **⚠️ Decisión clave (cambió respecto a versiones anteriores): el costo de envío NO se imputa en el pipeline base.**

- `is_free_delivery = 1` si la cadena contiene `"FREE"` → **6.236 registros**.
- `delivery_cost`: extracción de `$X delivery` → numérico.
- **Corrección aplicada:** si `is_free_delivery == 1`, el costo se fija en **0** (envío gratis real). Así se evita confundir "envío gratis" con "envío desconocido".
- **Nulos: 24.560** (productos sin información de envío) se **conservan como `NaN`**. No se imputa la mediana aquí; la decisión de imputar con 0 o descartar la columna se difiere a las variantes (§8).
- **Por qué no imputar de entrada:** asumir que un envío desconocido cuesta 0 es una suposición fuerte. Se decide por comparación empírica.

---

## 5. Tratamiento de nulos — tabla consolidada

| Columna | Nulos iniciales | Nulos tras excluir filas no etiquetables | Estrategia | Justificación |
|---|---|---|---|---|
| `bought_in_last_month` | 3.217 | 0 (filas eliminadas) | **Eliminar fila** | Es el objetivo; no se puede inventar la etiqueta. |
| `Current/discounted_price` | 11.749 | **6.766** | **Se conservan como NaN** → decidir por variante (§8) | El precio es específico del producto; no se impone una imputación. |
| `Rating` | 1.024 | 76 | Mediana (4,50) | Robusta a extremos. |
| `Number_of_reviews` | 1.024 | 76 | 0 | Ausencia de reseñas = 0 reseñas. |
| `buy_box_availability` | 14.653 | 8.501 | Categoría `unknown` | Ausencia informativa, no imputable. |
| `delivery_details` | 11.720 | **24.560** (desconocidos) | **Se conservan como NaN** → decidir por variante (§8) | Gratis = 0 real; desconocido ≠ 0. |
| `sustainability_badges` | 39.267 | 28.460 | Flag binario + **prueba de asociación** | Se retiene por asociación significativa (V≈0,10). |
| `Price_on_variant` | ~21.000 | — | **Variable excluida** | Calidad insuficiente. |
| `image_url`, `product_url`, `collected_at` | 0 / 2.069 / 0 | — | **Variables excluidas** | Identificadores. |

**Filas eliminadas por objetivo no utilizable: 10.511 (25,20 %)** del dataset tras dedup, compuestas por:
- 3.217 nulos explícitos,
- 4.571 `No featured offers available`,
- 2.723 etiquetas de texto no numéricas (`More Buying Choices`, `Typical:`, `List:`, `ESRB Rating`, precios por unidad, etc.).

---

## 6. Tratamiento de atípicos

| Variable | Criterio | Decisión |
|---|---|---|
| `current_price` | IQR sobre no nulos (Q1=24,99; Q3=159,95; límites [−177,45 ; 362,39]) | **Bandera `is_price_outlier`, SIN eliminar.** Precio alto puede ser legítimo y es feature, no objetivo. Se recalcula en cada variante. |
| `bought_in_last_month` (objetivo) | **No se filtran outliers** | Al ser el objetivo, eliminar filas por su valor sesga la distribución. La cola se domestica con `log1p`. |

> **Diferencia con el pipeline anterior:** antes se eliminaban filas por IQR en ambos casos. Ahora: precio → bandera; objetivo → recodificación + log.

---

## 7. Transformaciones finales

| Columna nueva | Fórmula | Uso |
|---|---|---|
| `log_price` | `log1p(current_price)` | Feature |
| `log_reviews` | `log1p(Number_of_reviews)` | Feature |
| `log_bought` | `log1p(bought_in_last_month)` | **Target de modelado** |
| `log_delivery_cost` | `log1p(delivery_cost)` | Feature |
| `log_title_words` | `log1p(title_word_count)` | Feature |

**Nota:** no se duplican transformaciones. El log reemplaza al filtrado IQR como mecanismo de control de cola; aplicar ambos sería redundante y perdería datos.

---

## 8. Decisiones abiertas → diseño 2×2 de variantes

Tres decisiones de imputación ya se resolvieron (Rating→mediana, reviews→0, sustainability→flag). Quedan **dos decisiones abiertas** que no tienen una respuesta única correcta:

1. Qué hacer con los productos **sin precio** (`current_price` nulo).
2. Qué hacer con el **costo de envío desconocido** (`delivery_cost` nulo).

Para no decidirlo a ciegas, se generan **4 variantes** y en la fase de modelado se comparará su rendimiento:

| # | Precio vacío | `delivery` |
|---|---|---|
| **v1** | **Eliminar fila** | imputar costo con 0 |
| **v2** | **Eliminar fila** | **sin columna** `delivery_cost` |
| **v3** | **Imputar mediana** | imputar costo con 0 |
| **v4** | **Imputar mediana** | **sin columna** `delivery_cost` |

**Lo que compara cada contraste:**

- **v1 vs v3** → efecto de **eliminar vs imputar** el precio (con delivery).
- **v2 vs v4** → lo mismo, pero **sin** la columna delivery.
- **v1 vs v2** y **v3 vs v4** → efecto de **tener o no** `delivery_cost`.

> En todas las variantes se conserva `is_free_delivery` (la señal binaria de envío gratis), independientemente de si se conserva o no `delivery_cost`.

---

## 9. Los 4 datasets resultantes

Todas las variantes parten del **mismo pipeline** (§2–§7) y difieren solo en las dos decisiones de §8.

| Variante | Precio | Delivery | Filas | Columnas | Archivo |
|---|---|---|---|---|---|
| **v1** | eliminar | imputar 0 | **24.436** | **20** | `dataset_v1.csv` |
| **v2** | eliminar | sin columna | **24.436** | **18** | `dataset_v2.csv` |
| **v3** | imputar mediana | imputar 0 | **31.202** | **20** | `dataset_v3.csv` |
| **v4** | imputar mediana | sin columna | **31.202** | **18** | `dataset_v4.csv` |

- **Variantes con delivery (v1, v3):** 20 columnas, incluyen `delivery_cost` y `log_delivery_cost`.
- **Variantes sin delivery (v2, v4):** 18 columnas, omiten esas dos (pero conservan `is_free_delivery`).
- **Todas sin nulos** una vez aplicada su estrategia.

### Columnas comunes (todas las variantes)

| Columna | Tipo | Descripción |
|---|---|---|
| `Rating` | float | Calificación 0–5 |
| `Number_of_reviews` | float | Nº de reseñas |
| `has_discount` | int | Tiene descuento |
| `is_best_seller_flag` | int | Badge "Best Seller" |
| `is_sponsored_flag` | int | Producto patrocinado |
| `has_coupon` | int | Tiene cupón |
| `buy_box_status` | str | `add_to_cart` / `unknown` |
| `is_free_delivery` | int | Envío gratuito |
| `title_word_count` | int | Palabras del título |
| `has_sustainability_badge` | int | Flag de badge (Opción C, §4.10) |
| `bought_in_last_month` | float | **Objetivo** (punto medio) |
| `bought_cat` | category | Categoría ordinal auxiliar |
| `current_price` | float | Precio (ya resuelto según variante) |
| `is_price_outlier` | int | Bandera IQR sobre precio |
| `log_price` | float | log1p del precio |
| `log_reviews` | float | log1p de reseñas |
| `log_bought` | float | **Target de modelado** |
| `log_title_words` | float | log1p de palabras del título |

### Columnas adicionales (solo v1 y v3)

| Columna | Tipo | Descripción |
|---|---|---|
| `delivery_cost` | float | Costo de envío (0 si gratis / imputado) |
| `log_delivery_cost` | float | log1p del costo de envío |

---

## 10. Limitaciones y supuestos

1. **Sesgo de retransformación:** al predecir `log1p` y volver con `expm1`, la media de predicciones no coincide con la media en escala original. Si la métrica lo exige, se aplica corrección de Duan o se evalúa en escala log (recomendado).
2. **Punto medio aproximado:** el punto medio usa los límites de bucket observados; en el último bucket (>100.000) se usa `v * 1.5` como aproximación.
3. **`sustainability_badges`:** retenido por significación estadística, pero con 91 % de ausencia. Puede aportar poco en el modelo; se verificará con importancia de variables.
4. **`is_best_seller`:** las categorías `Amazon's` y `Limited time deal` no se explotan; posible señal desaprovechada.
5. **Eliminación del 25 % de filas** por objetivo no utilizable reduce el dataset, pero es inevitable: no se puede entrenar sin etiqueta.
6. **Scraping temporal:** los datos corresponden a una ventana de recolección concreta; los patrones de venta pueden no ser estables en el tiempo.
7. **`buy_box_status`** solo tiene 2 valores efectivos (`add_to_cart` / `unknown`); la dummy de ausencia es la que aporta.
8. **Comparación de variantes:** v1/v2 tienen menos filas que v3/v4 (no tienen los productos sin precio). Al comparar modelos hay que tenerlo en cuenta (usar un test común o interpretar el tamaño de muestra); no lo hace "mejor" solo por tener más datos.

---

## 11. Reproducibilidad

- **Notebook guía:** `limpieza_4_datasets.ipynb`
- **Datasets generados:** `dataset_v1.csv`, `dataset_v2.csv`, `dataset_v3.csv`, `dataset_v4.csv`

Para regenerar todo, ejecutar el notebook completo:

```bash
.venv/bin/python -m jupyter nbconvert --to notebook --execute limpieza_4_datasets.ipynb \
    --ExecutePreprocessor.kernel_name=amazon-venv
```

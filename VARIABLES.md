# Diccionario de Variables — Amazon Product Sales

> Explica **qué significa cada variable**: primero las **originales** (tal como llegan del
> scraping, archivo `amazon_sales_data_uncleaned.csv`) y luego las **finales** (que salen del
> pipeline `limpieza_4_datasets.ipynb` hacia `dataset_v1.csv` … `dataset_v4.csv`).
> Para el *cómo* y el *por qué* de cada transformación, ver `LIMPIEZA_DATOS.md`.

---

## Parte 1 — Variables originales (16 columnas)

El dataset crudo tiene **42.675 filas × 16 columnas**, todas almacenadas como texto (`str`),
incluso cuando representan números. Este es el significado de cada una:

### 1.1 Información de producto

| Variable | Significado | Tipo crudo | Nulos | Distintos | Ejemplo |
|---|---|---|---|---|---|
| `Title` | **Nombre comercial del producto**, el texto del título del listing en Amazon. Contiene marca, modelo y características. | texto | 0 | 8.808 | `"Apple AirPods Pro 2 Wireless Earbuds..."` |
| `image_url` | **Dirección web de la imagen** del producto (CDN de Amazon). | texto (URL) | 0 | 8.038 | `"https://m.media-amazon.com/images/I/71pA..."` |
| `product_url` | **Dirección web de la página del producto** en Amazon. Puede incluir tokens de sesión. | texto (URL) | 2.069 | 40.606 | `"/Apple-AirTag-Pack/dp/B0D54..."` |

### 1.2 Reputación / reseñas

| Variable | Significado | Tipo crudo | Nulos | Distintos | Ejemplo |
|---|---|---|---|---|---|
| `Rating` | **Calificación promedio** del producto (0 a 5 estrellas). | texto (`"4.6 out of 5 stars"`) | 1.024 | 31 | `"4.6 out of 5 stars"` |
| `Number_of_reviews` | **Cantidad de reseñas** que ha recibido el producto. | texto con comas (`"2,457"`) | 1.024 | 4.413 | `"2,457"` |

### 1.3 Precios y descuento

| Variable | Significado | Tipo crudo | Nulos | Distintos | Ejemplo |
|---|---|---|---|---|---|
| `Current/discounted_price` | **Precio actual de venta** del producto (ya con descuento si aplica). Es el precio que paga el cliente. | texto numérico | 11.749 | 2.576 | `"89.68"` |
| `Listed_price` | **Precio de lista original** (antes del descuento). Si no hay descuento contiene el texto `"No Discount"`. | texto (`"$159.00"` / `"No Discount"`) | 0 | 911 | `"$159.00"` |
| `Price_on_variant` | Precio de una **variante del producto** (ej. otro color/tamaño). En la práctica, mezcla precios con descriptores de texto, por lo que no es fiable. | texto mixto | 0 | 3.466 | `"basic variant price: $162.24"` |

### 1.4 Badges e indicadores comerciales (visibilidad)

| Variable | Significado | Tipo crudo | Nulos | Distintos | Ejemplo |
|---|---|---|---|---|---|
| `is_best_seller` | **Insignia o etiqueta promocional** que Amazon muestra sobre el producto. La más importante es `"Best Seller"` (producto más vendido de su categoría), pero la columna mezcla varias etiquetas (`Amazon's`, `Limited time deal`, ofertas `Save X%`). | texto | 0 | 12 | `"Best Seller"` |
| `is_sponsored` | Indica si el producto es **patrocinado** (anuncio pagado) u **orgánico** (resultado natural). | texto | 0 | 2 | `"Sponsored"` / `"Organic"` |
| `is_couponed` | Indica si el producto ofrece **cupón de descuento** y de qué tipo. | texto | 0 | 42 | `"Save 15%  with coupon"` / `"No Coupon"` |
| `buy_box_availability` | Indica si el producto tiene la **"Buy Box"** (el botón "Añadir al carrito"). Es un proxy de disponibilidad para compra directa. | texto | 14.653 | 1 | `"Add to cart"` |

### 1.5 Envío y sostenibilidad

| Variable | Significado | Tipo crudo | Nulos | Distintos | Ejemplo |
|---|---|---|---|---|---|
| `delivery_details` | **Información de entrega**: fechas estimadas, si el envío es gratis (`FREE`) y su costo (`$X delivery`). | texto | 11.720 | 298 | `"Delivery Mon, Sep 1"` |
| `sustainability_badges` | **Insignias de producto** relacionadas con sostenibilidad o compatibilidad (Small Business, Carbon impact, Works with Alexa…). | texto | 39.267 | 16 | `"Carbon impact"` |

### 1.6 Metadatos del scraping

| Variable | Significado | Tipo crudo | Nulos | Distintos | Ejemplo |
|---|---|---|---|---|---|
| `collected_at` | **Fecha y hora** en que el dato fue capturado por el scraper. | texto (fecha) | 0 | 331 | `"21-08-2025 11:14"` |
| `bought_in_last_month` | **Cantidad de unidades vendidas en el último mes**, expresada como un **rango truncado**: `"50+ bought in past month"`, `"1K+..."`. Es el **objetivo** del problema. | texto (`"N+ bought..."`) | 3.217 | 59 | `"300+ bought in past month"` |

> **Observación general:** casi todas las columnas traen "ruido" de scraping: la información útil está
> **incrustada en texto** (`"4.6 out of 5 stars"`, `"2,457"`, `"$159.00"`, `"Save 15% with coupon"`).
> La limpieza consiste, en buena parte, en **extraer el dato limpio** de esa cadena.

---

## Parte 2 — Variables finales (18 o 20 columnas)

Del pipeline salen **4 variantes**. Todas comparten **18 columnas**; las variantes **v1** y **v3**
añaden **2 más** (del delivery). A continuación, el significado de cada variable final.

### 2.1 Reputación

| Variable | Significado | Tipo | Origen |
|---|---|---|---|
| `Rating` | **Calificación promedio** del producto, ya como número (0–5). Nulos imputados con la mediana (4,50). | float | `Rating` |
| `Number_of_reviews` | **Cantidad de reseñas** como número entero. Nulos → 0 (sin reseñas = cero). | float | `Number_of_reviews` |

### 2.2 Precio

| Variable | Significado | Tipo | Origen |
|---|---|---|---|
| `current_price` | **Precio de venta actual** en dólares, como número. En v1/v2 solo existen filas con precio; en v3/v4 los nulos se imputan con la mediana. | float | `Current/discounted_price` |
| `is_price_outlier` | **Bandera (0/1)** que marca si el precio es atípico según el rango intercuartílico (IQR). `1` = precio fuera de lo común (muy caro o muy barato). No se elimina, solo se señala. | int | derivada de `current_price` |
| `log_price` | **Logaritmo del precio**: `log1p(current_price)`. Comprime la cola alta para que el modelo no se vea dominado por productos carísimos. | float | derivada de `current_price` |

### 2.3 Banderas comerciales

| Variable | Significado | Tipo | Origen |
|---|---|---|---|
| `has_discount` | **Bandera**: `1` si el producto tiene un precio de lista mayor al actual (tiene descuento), `0` si no. | int | `Listed_price` |
| `is_best_seller_flag` | **Bandera**: `1` solo si el producto es `"Best Seller"`; cualquier otra etiqueta pasa a `0`. | int | `is_best_seller` |
| `is_sponsored_flag` | **Bandera**: `1` si el producto es patrocinado, `0` si es orgánico. | int | `is_sponsored` |
| `has_coupon` | **Bandera**: `1` si el producto ofrece cupón, `0` si no. | int | `is_couponed` |
| `has_sustainability_badge` | **Bandera**: `1` si el producto tiene *alguna* insignia (sostenibilidad o compatibilidad). Se retiene por asociación estadística significativa (χ², ver `LIMPIEZA_DATOS.md` §4.10). | int | `sustainability_badges` |

### 2.4 Disponibilidad

| Variable | Significado | Tipo | Origen |
|---|---|---|---|
| `buy_box_status` | **Estado del botón de compra**: `add_to_cart` si el producto se puede añadir al carrito, `unknown` si el dato faltaba. La ausencia se conserva como categoría propia. | str | `buy_box_availability` |

### 2.5 Envío

| Variable | Significado | Tipo | Origen | ¿En qué variantes? |
|---|---|---|---|---|
| `is_free_delivery` | **Bandera**: `1` si el envío es gratuito (`FREE delivery`), `0` si no. | int | `delivery_details` | **Todas (v1–v4)** |
| `delivery_cost` | **Costo del envío** en dólares. `0` cuando es gratis. En v1/v3 los desconocidos se imputan con 0. | float | `delivery_details` | Solo **v1, v3** |
| `log_delivery_cost` | **Logaritmo del costo de envío**: `log1p(delivery_cost)`. | float | derivada de `delivery_cost` | Solo **v1, v3** |

> **Nota:** `is_free_delivery` se conserva **aunque se elimine** `delivery_cost` (v2, v4), porque la
> condición de "envío gratis / no" es información útil por sí misma, independiente del monto.

### 2.6 Texto (resumen)

| Variable | Significado | Tipo | Origen |
|---|---|---|---|
| `title_word_count` | **Número de palabras del título**. Es un proxy de cuán "detallado/rico" es el listing (títulos más largos suelen indicar listings más cuidados). | int | `Title` |
| `log_title_words` | **Logaritmo** del número de palabras del título: `log1p(title_word_count)`. | float | derivada de `title_word_count` |

### 2.7 Objetivo (lo que se quiere predecir)

| Variable | Significado | Tipo | Origen |
|---|---|---|---|
| `bought_in_last_month` | **Unidades vendidas en el último mes**, como número. Usa el **punto medio** del rango de Amazon (ej. `"50+"` → 75, `"100+"` → 150). Es el **objetivo numérico**. | float | `bought_in_last_month` |
| `bought_cat` | **Categoría ordinal auxiliar**: el objetivo agrupado en 4 rangos → `50-100`, `101-200`, `201-500`, `>500`. Sirve para **estratificar** particiones y hacer pruebas estadísticas. **No es feature ni target del modelo.** | category | derivada de `bought_in_last_month` |
| `log_bought` | **Logaritmo del objetivo**: `log1p(bought_in_last_month)`. Es el **target real de modelado** (la distribución cruda es muy asimétrica). | float | derivada de `bought_in_last_month` |

---

## Parte 3 — Mapa original → final

Resumen de qué columna original se convirtió en qué variable(**) final(es):

| Original | → | Final(es) |
|---|---|---|
| `Title` | → | `title_word_count`, `log_title_words` |
| `Rating` | → | `Rating` |
| `Number_of_reviews` | → | `Number_of_reviews`, `log_reviews` |
| `bought_in_last_month` | → | `bought_in_last_month`, `bought_cat`, `log_bought` |
| `Current/discounted_price` | → | `current_price`, `is_price_outlier`, `log_price` |
| `Price_on_variant` | → | ❌ **excluida** |
| `Listed_price` | → | `has_discount` |
| `is_best_seller` | → | `is_best_seller_flag` |
| `is_sponsored` | → | `is_sponsored_flag` |
| `is_couponed` | → | `has_coupon` |
| `buy_box_availability` | → | `buy_box_status` |
| `delivery_details` | → | `is_free_delivery`, `delivery_cost`*, `log_delivery_cost`* |
| `sustainability_badges` | → | `has_sustainability_badge` |
| `image_url` | → | ❌ **excluida** |
| `product_url` | → | ❌ **excluida** |
| `collected_at` | → | ❌ **excluida** |

\* `delivery_cost` y `log_delivery_cost` solo existen en las variantes **v1 y v3**.
\*\* Falta `log_reviews` en la lista de origen: se deriva de `Number_of_reviews` (ya incluido arriba).

---

## Recordatorio: el par "normal + log"

Toda variable numérica del dataset final existe **por duplicado**: en su escala original y en
escala logarítmica.

| Original | Log |
|---|---|
| `current_price` | `log_price` |
| `Number_of_reviews` | `log_reviews` |
| `bought_in_last_month` | `log_bought` |
| `title_word_count` | `log_title_words` |
| `delivery_cost` (v1/v3) | `log_delivery_cost` |

**Al modelar:** no uses ambas a la vez en modelos lineales (multicolinealidad). Los árboles toleran
cualquiera de las dos; los modelos lineales suelen preferir la versión log cuando hay cola.

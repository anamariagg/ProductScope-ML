# Amazon Product Sales — Limpieza y Preparación de Datos

Pipeline de limpieza y transformación del dataset de productos de Amazon, con el objetivo de
preparar los datos para modelar el **volumen de ventas recientes** (`bought_in_last_month`).

El resultado son **4 datasets** (`dataset_v1.csv` … `dataset_v4.csv`) que representan distintas
estrategias de imputación, para compararlas empíricamente en la fase de modelado.

---

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| `amazon_sales_data_uncleaned.csv` | **Entrada.** Dataset crudo tal como sale del scraping (42.675 filas × 16 columnas). |
| `limpieza_4_datasets.ipynb` | **Notebook guía.** Pipeline completo, paso a paso y documentado. Es el script principal del proyecto. |
| `LIMPIEZA_DATOS.md` | **Documento técnico.** Explica cada decisión de limpieza con sus cifras y justificación. |
| `VARIABLES.md` | **Documento técnico.** Explica el significado de cada variable antes y despues de la limpieza |
| `README.md` | Este archivo. |

---

## Requisitos

- **Python 3.14**
- Dependencias: `numpy`, `pandas`, `scipy`, `scikit-learn`, `matplotlib`, `jupyter`
- El entorno virtual ya existe en `.venv/`

Si necesitas recrearlo desde cero:

```bash
python -m venv .venv
.venv/bin/pip install numpy pandas scipy scikit-learn matplotlib jupyter
```

---

## Cómo interactuar con el proyecto

### Opción A — Ejecutar el notebook completo (recomendado)

Regenera los 4 datasets de una sola vez, de punta a punta:

```bash
.venv/bin/python -m jupyter nbconvert --to notebook --execute limpieza_4_datasets.ipynb \
    --ExecutePreprocessor.kernel_name=amazon-venv
```

Al terminar, se habrán sobrescrito `dataset_v1.csv` … `dataset_v4.csv`.

### Opción B — Abrir el notebook de forma interactiva

```bash
.venv/bin/jupyter notebook limpieza_4_datasets.ipynb
```

(o `jupyter lab`). Selecciona el kernel **`amazon-venv`** y ejecuta las celdas de arriba hacia
abajo con `Shift+Enter` o *Run All*.

---

## Qué hace el pipeline (resumen)

1. **Inventario** del dataset (nulos, cardinalidad).
2. **Eliminar duplicados** exactos (962 filas).
3. **Objetivo `bought_in_last_month`**: parseo estricto del patrón `N+ bought in past month`,
   vista previa de lo que se elimina, uso del **punto medio** del intervalo.
4. **Variables excluidas**: `image_url`, `product_url`, `collected_at`, `Price_on_variant`.
5. **Transformaciones**: `Rating`, reviews, precio, banderas, `buy_box_status`, `sustainability_badges`
   (con prueba χ² + Cramér's V), `Title`, `delivery_details`.
6. **Transformación logarítmica** (`log1p`) sobre numéricas y objetivo.
7. **Generación de los 4 datasets** (diseño 2×2 precio × delivery).

El detalle completo de cada decisión está en **`LIMPIEZA_DATOS.md`**.

---

## Los 4 datasets: qué representa cada uno

| Variante | Precio vacío | Delivery | Filas | Columnas |
|---|---|---|---|---|
| **v1** | Eliminar fila | Imputar costo con 0 | 24.436 | 20 |
| **v2** | Eliminar fila | Sin columna `delivery_cost` | 24.436 | 18 |
| **v3** | Imputar mediana | Imputar costo con 0 | 31.202 | 20 |
| **v4** | Imputar mediana | Sin columna `delivery_cost` | 31.202 | 18 |

- **v1 vs v3** → mide el efecto de **eliminar vs imputar** el precio.
- **v2 vs v4** → lo mismo, pero sin la columna de envío.
- **v1 vs v2** y **v3 vs v4** → mide el **aporte de `delivery_cost`**.

Las cuatro variantes comparten el mismo pipeline; solo cambian esas dos decisiones.

### Columnas de salida

**Comunes a todas (18):**
`Rating`, `Number_of_reviews`, `has_discount`, `is_best_seller_flag`, `is_sponsored_flag`,
`has_coupon`, `buy_box_status`, `is_free_delivery`, `title_word_count`, `has_sustainability_badge`,
`bought_in_last_month`, `bought_cat`, `current_price`, `is_price_outlier`, `log_price`,
`log_reviews`, `log_bought`, `log_title_words`.

**Adicionales en v1 y v3 (2):**
`delivery_cost`, `log_delivery_cost`.

> **Importante al modelar:** el objetivo es `log_bought` (= `log1p` de `bought_in_last_month`).
> Tanto `bought_in_last_month`, `bought_cat` como los demás `log_*` derivados deben **excluirse
> de las features** para evitar fuga de información.

---

## Notas para la fase de modelado

- Usa `bought_cat` solo para **estratificar** la partición, nunca como feature.
- La cola del objetivo es muy asimétrica → evalúa métricas en escala log y des-transforma con
  `expm1` si reportas en unidades.
- Las variantes v1/v2 tienen menos filas que v3/v4; al comparar modelos tenlo en cuenta
  (idealmente, evaluar sobre un conjunto de test común).
- `buy_box_status` tiene solo 2 valores efectivos (`add_to_cart` / `unknown`); conviértela a dummy.

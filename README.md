# Amazon Product Sales — ProductScope-ML
- Sebastian B. Ruiz - s.barrera@udea.edu.co
- Ana Maria G. Grisales - Anam.garzon@udea.edu.co
- Juan F. Mona Cano - juanf.mona@udea.edu.co 
---

Pipeline de limpieza y transformación del dataset de productos de Amazon, con el objetivo de
preparar los datos para modelar el **volumen de ventas recientes** (`bought_in_last_month`).

El pipeline genera **4 datasets** (`dataset_v1.csv` … `dataset_v4.csv`) que representan distintas
estrategias de imputación, para compararlas empíricamente en la fase de modelado.

---

## Estructura del repositorio

```
Amazon product sales/
├── README.md                    # Este archivo
├── Limpieza de datos/           # Pipeline de limpieza y documentación
│   ├── amazon_sales_data_uncleaned.csv   # Entrada cruda (42.675 × 16)
│   ├── limpieza_4_datasets.ipynb         # Notebook guía (pipeline completo)
│   ├── dataset_v1.csv … dataset_v4.csv   # Salidas (se regeneran)
│   ├── LIMPIEZA_DATOS.md                 # Documento técnico de decisiones
│   └── VARIABLES.md                      # Diccionario de variables
├── Modelos de ejemplo/          # Notebooks de laboratorio (referencia)
│   ├── Copy_of_lab2_parte1.ipynb
│   └── Copy_of_lab2_parte2.ipynb
└── Modelos entrenados/          # Modelos ya entrenados (artefactos)
```

| Carpeta | Contenido |
|---|---|
| **`Limpieza de datos/`** | Todo el pipeline: CSV de entrada, notebook, los 4 datasets generados y la documentación técnica (`LIMPIEZA_DATOS.md`, `VARIABLES.md`). |
| **`Modelos de ejemplo/`** | Notebooks de laboratorio usados como referencia. |
| **`Modelos entrenados/`** | Carpeta para guardar los modelos entrenados (artefactos `.pkl`, `.joblib`, reportes). |

> **Nota:** el pipeline trabaja con rutas relativas a la carpeta `Limpieza de datos/`, así que el
> notebook debe ejecutarse **desde dentro de esa carpeta** (ver más abajo).

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

El notebook se ejecuta **desde dentro de la carpeta `Limpieza de datos/`**.

### Opción A — Ejecutar el notebook completo (regenera los 4 datasets)

```bash
cd "Limpieza de datos"
../.venv/bin/python -m jupyter nbconvert --to notebook --execute limpieza_4_datasets.ipynb \
    --ExecutePreprocessor.kernel_name=amazon-venv
```

Al terminar, se habrán (re)generado `dataset_v1.csv` … `dataset_v4.csv` dentro de
`Limpieza de datos/`.

### Opción B — Abrir el notebook de forma interactiva

```bash
cd "Limpieza de datos"
../.venv/bin/jupyter notebook limpieza_4_datasets.ipynb
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

El detalle de cada decisión está en **`Limpieza de datos/LIMPIEZA_DATOS.md`** y el significado
de cada variable en **`Limpieza de datos/VARIABLES.md`**.

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
- Cada variable numérica existe como par `X` + `log_X`: no uses ambas a la vez en modelos lineales
  (multicolinealidad); los árboles toleran cualquiera.

---

## Datos versionados

Los `dataset_v*.csv` son **regenerables** ejecutando el notebook y no necesitan subirse al
repositorio. El CSV crudo (`Limpieza de datos/amazon_sales_data_uncleaned.csv`) sí se versiona
como fuente de verdad del proyecto.

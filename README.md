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
└── Modelos entrenados/          # Notebooks, modelos y resultados de KNN, SVR y otros
    ├── KNN.ipynb
    ├── SVM.ipynb                 # SVR para predecir una variable continua
    ├── RandomForestRegressor.ipynb
    ├── Red_Neuronal_MLPRegressor.ipynb
    ├── Ridge_regression.ipynb
    └── Models_Results/           # Métricas, predicciones, búsquedas y modelos
```

| Carpeta | Contenido |
|---|---|
| **`Limpieza de datos/`** | Todo el pipeline: CSV de entrada, notebook, los 4 datasets generados y la documentación técnica (`LIMPIEZA_DATOS.md`, `VARIABLES.md`). |
| **`Modelos entrenados/`** | Notebooks de KNN, SVR, Random Forest, MLP y Ridge, más modelos serializados y resultados. |
| **`Informes`** | Carpeta la cual guarda los respectivos informes E1 y E2 en sus diferentes versiones |

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

## Modelos de regresión

El proyecto contiene cinco modelos supervisados para predecir
`log_bought = log1p(bought_in_last_month)`:

| Modelo | Notebook | Dataset de entrenamiento | Familia |
|---|---|---|---|
| KNN | [`KNN.ipynb`](Modelos%20entrenados/KNN.ipynb) | v1–v4 | Vecinos más cercanos para regresión |
| SVR | [`SVM.ipynb`](Modelos%20entrenados/SVM.ipynb) | v1–v4 | Support Vector Regression |
| Random Forest | [`RandomForestRegressor.ipynb`](Modelos%20entrenados/RandomForestRegressor.ipynb) | v4 | Ensamble de árboles |
| MLP | [`Red_Neuronal_MLPRegressor.ipynb`](Modelos%20entrenados/Red_Neuronal_MLPRegressor.ipynb) | v4 | Red neuronal multicapa |
| Ridge Regression | [`Ridge_regression.ipynb`](Modelos%20entrenados/Ridge_regression.ipynb) | v4 | Regresión lineal regularizada (L2) |

El objetivo es continuo, por lo que la máquina de soporte se implementa como **SVR**, no como SVC
de clasificación. Todos los modelos usan los datasets del pipeline de limpieza y conservan las
observaciones de ventas altas; los valores extremos no se eliminan por ser extremos.

### Cómo ejecutar los notebooks

Primero genera los datasets siguiendo las instrucciones de la sección anterior. Después, desde la
raíz del repositorio:

```bash
cd "Modelos entrenados"
../.venv/bin/jupyter notebook
```

Abre el notebook del modelo y ejecuta sus celdas en orden. Random Forest genera los índices aleatorios que MLP reutiliza para comparar sobre la misma partición; Ridge conserva su propia partición aleatoria 70/15/15. KNN genera los índices agrupados que SVR reutiliza. Por ello, al regenerar todos los resultados desde cero, ejecuta primero
KNN y Random Forest, y luego SVR y MLP. Cada notebook crea o actualiza su carpeta de resultados
correspondiente. Los artefactos actuales también permiten consultar resultados sin volver a
entrenar.

### Protocolo común y variables

- El objetivo modelado es `log_bought`. `bought_in_last_month` y `bought_cat` no se usan como
  predictores; `bought_cat` solo sirve para estratificar los folds.
- Las variables predictoras son indicadores de promociones y características del producto, `Rating`,
  variables transformadas como `log_price`, `log_reviews` y `log_title_words`, e
  `buy_box_unknown`. v1/v3 incluyen además `log_delivery_cost`; v2/v4 no tienen esa variable.
- KNN y SVR excluyen `is_price_outlier` como predictor. Random Forest y MLP usan v4, incluyen esta
  bandera numérica y codifican `buy_box_status` como categoría. Las listas de features no son
  idénticas entre las cuatro familias de modelos. Excluir esa bandera no implica eliminar filas con
  ventas altas: la cola del **objetivo de ventas** se conserva.
- Las métricas primarias de selección son MAE en escala logarítmica; también se reportan RMSE y R²
  en log, además de métricas en unidades tras aplicar `expm1`.
- **Partición de KNN y SVR:** se reserva un test de 4.887 filas de las 24.436 filas comunes
  (20 %); las 19.549 restantes (80 %) quedan para búsqueda/validación y entrenamiento final. La
  partición externa usa `StratifiedGroupKFold` de 5 folds: un fold se congela como test y los otros
  cuatro se usan para desarrollo. La búsqueda de hiperparámetros usa, dentro de esas filas de
  desarrollo, validación cruzada agrupada de 4 folds; tras elegir la configuración, el ajuste final
  usa todo el 80 % disponible.
- En este protocolo, **grupo** significa filas con el mismo vector completo de features elegido
  para agrupar; no significa un producto identificado por ID. Así se evalúa el caso en que el vector
  completo de features del test no está repetido en entrenamiento. Es una decisión de evaluación,
  no un requisito general ni una regla de que productos distintos no puedan compartir algunas o
  todas sus características. Para v3/v4, además, las 6.766 filas adicionales con precio imputado se
  pueden usar en entrenamiento si su grupo no está en el test; el test común mantiene productos
  con precio observado.
- Los CSV ya contienen imputaciones y límites calculados durante la limpieza antes de la partición.
  Los pipelines de modelado ajustan sus transformaciones internas en los folds/conjuntos de
  entrenamiento, pero no pueden deshacer ese preprocesamiento global previo; las métricas deben
  interpretarse teniendo en cuenta esta limitación.
- KNN y SVR usan la partición agrupada 80/20 descrita arriba, con un test común de 4.887 filas.
  Random Forest y MLP usan una partición aleatoria estratificada 70/15/15 de v4 (21.841
  entrenamiento, 4.680 validación y 4.681 prueba); sus archivos de índices confirman que es la
  misma partición para ambos. Ridge también usa v4 y conserva una partición estratificada 70/15/15 (21.840 entrenamiento, 4.681 validación y 4.681 prueba), con índices propios que no coinciden con los de Random Forest/MLP. Para seleccionar `alpha`, combina entrenamiento y validación en un conjunto de desarrollo de 26.521 filas y aplica K-Fold de 5 folds; el test de 4.681 filas permanece aislado hasta la evaluación final.
  Las cuatro métricas de test **no son comparables como un ranking directo**: los modelos no
  comparten el mismo conjunto de prueba ni exactamente el mismo protocolo. Las evaluaciones
  agrupadas y aleatorias responden a escenarios distintos; compartir valores de features entre dos
  filas no demuestra que sean el mismo producto ni constituye por sí solo fuga de información.

### KNN de regresión

KNN es un método basado en instancias: para una observación nueva, identifica los vecinos más cercanos en el espacio de características y combina sus valores objetivo para producir la predicción. En esta implementación se prueban pesos uniformes y pesos inversos a la distancia; estos últimos dan mayor influencia a los vecinos más próximos. Su rendimiento depende de la métrica, del escalado y de que existan observaciones suficientemente similares en el conjunto de entrenamiento.

[`KNN.ipynb`](Modelos%20entrenados/KNN.ipynb) implementa KNN manualmente con distancias calculadas
mediante `cdist` y verifica el comportamiento frente a `KNeighborsRegressor`. Evalúa
`k ∈ {1, 3, 5, 7, 10, 15, 25, 40, 60, 100}`, pesos uniformes o inversamente proporcionales a la
distancia y distancias euclidiana o Manhattan (`cityblock`). El escalado se ajusta solo con el train
de cada fold. Se congela un test del 20 % mediante un split estratificado por grupos y la búsqueda
usa 4 folds agrupados sobre el 80 % restante.

La configuración seleccionada para v1 es `k=100`, `weights=distance`, `metric=cityblock`. En test
v1 obtuvo MAE log = **0,5000**, RMSE log = **0,7617**, R² log = **0,4527** y MAE = **493,3 ventas**
en unidades. Como referencia, el baseline que predice la mediana de train obtuvo MAE log = 0,8452.
En validación, las cuatro variantes seleccionaron la misma configuración:

| Variante | MAE log medio | IC 95 % del MAE | Configuración |
|---|---:|---:|---|
| v1 | 0,5212 | [0,4644; 0,5713] | k=100, distancia, Manhattan |
| v2 | 0,5260 | [0,4642; 0,5869] | k=100, distancia, Manhattan |
| v3 | 0,5252 | [0,4724; 0,5752] | k=100, distancia, Manhattan |
| v4 | 0,5300 | [0,4709; 0,5920] | k=100, distancia, Manhattan |

El notebook documenta además la distribución del objetivo, comparación con el baseline, errores por
categoría de ventas y compara protocolos de partición. Una partición aleatoria estima el desempeño
en filas de una distribución similar; una partición agrupada estima el desempeño en filas cuyo
vector completo de características no aparece en entrenamiento. Que dos filas tengan las mismas
características no demuestra que sean el mismo producto ni constituye por sí solo fuga de
información. Las métricas de los dos protocolos responden a escenarios distintos y deben
identificarse por separado.

### SVR (máquina de soporte vectorial para regresión)

SVR busca una función de regresión que mantenga las desviaciones dentro de una banda de tolerancia `epsilon`, penalizando principalmente los errores que quedan fuera de ella. El kernel RBF permite representar relaciones no lineales entre las características y el objetivo. Los parámetros `C`, `epsilon` y `gamma` controlan, respectivamente, la penalización de errores, el ancho de la banda y el alcance de la influencia de cada observación; por eso el escalado dentro del pipeline es esencial.

[`SVM.ipynb`](Modelos%20entrenados/SVM.ipynb) compara configuraciones RBF y selecciona
hiperparámetros por variante con MAE logarítmico. La búsqueda explora `C ∈ {0,01; 0,025; 0,05;
0,075; 0,1}`, `epsilon ∈ {0,025; 0,05; 0,075}` y `gamma ∈ {0,005; 0,01; 0,015}`. Por costo
computacional, cada búsqueda usa una muestra agrupada de aproximadamente 2.000 filas y
`StratifiedGroupKFold` de 4 folds. El pipeline coloca imputación mediana, `StandardScaler` y SVR
dentro de cada fold; el modelo final se ajusta con todo el conjunto de entrenamiento.

| Variante | Mejor C | Epsilon | Gamma | MAE log test | RMSE log test | R² log test | MAE en ventas |
|---|---:|---:|---:|---:|---:|---:|---:|
| v1 | 0,05 | 0,025 | 0,015 | 0,4834 | 0,7482 | 0,4720 | 527,8 |
| v2 | 0,075 | 0,025 | 0,015 | 0,4836 | 0,7446 | 0,4770 | 507,7 |
| v3 | 0,01 | 0,025 | 0,015 | 0,4844 | 0,7706 | 0,4398 | 545,5 |
| v4 | 0,05 | 0,05 | 0,005 | 0,5041 | 0,7769 | 0,4306 | 530,4 |

#### Cola pesada de ventas e interpretación

El volumen de ventas tiene una cola derecha pesada: el negocio puede tener productos con pocas
ventas y otros con muchísimas. Esos valores extremos son observaciones posibles y relevantes, no
errores que deban eliminarse solo por ser extremos. La transformación logarítmica ayuda a modelar
la escala, pero no hace que SVR represente perfectamente toda la heterogeneidad de la demanda.

En el test, el grupo `>500` tiene MAE log entre **0,94 y 0,99** y MAE en unidades entre **1.903 y
2.099 ventas** según la variante; en los demás grupos, el MAE log está aproximadamente entre 0,31
y 0,39. Esto muestra una limitación predictiva de SVR en la cola alta con las características
disponibles, no un fallo de ejecución. Los resultados se reportan conservando los casos de ventas
altas.

KNN v1 y SVR v1 evalúan el mismo test común: sus MAE log son 0,5000 y 0,4834, respectivamente. La
diferencia es pequeña y no debe interpretarse como una mejora estadísticamente significativa sin
una comparación inferencial adicional. Las filas de test de v1–v4 representan los mismos productos,
por lo que las métricas permiten observar el efecto de las variantes del dataset; las métricas de
SVR por variante también reflejan cambios en los datos de entrenamiento.

### Random Forest Regressor

Random Forest combina múltiples árboles de decisión entrenados sobre muestras y subconjuntos aleatorios de características. En regresión, agrega las predicciones de los árboles, lo que permite capturar relaciones no lineales e interacciones sin exigir escalado de las variables. La aleatoriedad del ensamble ayuda a reducir la varianza de un árbol individual, aunque la importancia de variables debe interpretarse como una medida del uso predictivo dentro del modelo y no como evidencia causal.

[`RandomForestRegressor.ipynb`](Modelos%20entrenados/RandomForestRegressor.ipynb) entrena sobre
v4, que contiene 31.202 filas. Su pipeline imputa numéricas por mediana y `buy_box_status` por la
categoría más frecuente, y aplica One-Hot Encoding a esta última. No escala las variables, algo que
no es necesario para los árboles. Usa 12 configuraciones muestreadas de hiperparámetros y selecciona
por RMSE en la partición de validación; luego ajusta el modelo final con entrenamiento más
validación. La configuración seleccionada tiene 350 árboles, profundidad máxima 20,
`min_samples_split=5`, `min_samples_leaf=2` y `max_features=0.7`.

En su test aleatorio de 4.681 filas obtuvo MAE log = **0,1772**, RMSE log = **0,4463**, R² log =
**0,8596**, MAE original = **280,2 ventas** y R² original = **0,8651**. El notebook también exporta
importancia de variables, un árbol representativo y análisis de residuos.

### Ridge Regression

[`Ridge_regression.ipynb`](Modelos%20entrenados/Ridge_regression.ipynb) implementa una regresión lineal regularizada mediante penalización L2. Predice `log_bought` usando el dataset v4 (31.202 filas), con 11 variables numéricas y `buy_box_status` como variable categórica. Excluye `bought_in_last_month`, `log_bought` y `bought_cat` de los predictores; `bought_cat` se utiliza únicamente para estratificar la partición.

El preprocesamiento está integrado en un `Pipeline`: imputación por mediana y estandarización para las variables numéricas, e imputación por moda y One-Hot Encoding para `buy_box_status`. La estandarización es especialmente importante en Ridge porque la penalización L2 actúa sobre la magnitud de los coeficientes; sin escalado, las variables con unidades o rangos mayores podrían influir de manera desproporcionada en la regularización.

#### Selección de hiperparámetros y evaluación

Se conserva la partición estratificada 70/15/15: 21.840 filas de entrenamiento, 4.681 de validación y 4.681 de prueba. Para la búsqueda, entrenamiento y validación se combinan en un conjunto de desarrollo de 26.521 filas. Sobre este conjunto se ejecuta `GridSearchCV` con K-Fold de 5 folds, barajado y semilla fija (`random_state=0`). Se evalúan 49 valores de `alpha` entre `10^-8` y `10^4`, seleccionando el menor RMSE en escala logarítmica. El preprocesamiento permanece dentro del pipeline, de modo que imputación, escalado y codificación se ajustan de nuevo en cada fold. El conjunto de prueba no interviene en esta selección.

La búsqueda seleccionó `alpha=31,6228`, con RMSE medio de validación cruzada de **0,8550** en escala logarítmica. La desviación estándar entre folds fue aproximadamente **0,0129**, lo que indica resultados similares entre las particiones de validación. El mejor valor no está en un extremo de la rejilla y los valores cercanos de `alpha` obtuvieron resultados prácticamente equivalentes.

Una vez seleccionado el hiperparámetro, `GridSearchCV` reajusta el mejor pipeline con las 26.521 observaciones del conjunto de desarrollo (`refit=True`). El conjunto de prueba se utiliza después para una única evaluación final.

#### Resultados y lectura del modelo

En el test de 4.681 filas, Ridge obtuvo:

| Métrica | Resultado |
|---|---:|
| MAE log | **0,6081** |
| RMSE log | **0,8776** |
| R² log | **0,4800** |
| MAE original | **990,9 ventas** |
| RMSE original | **5.662,6 ventas** |
| R² original | **0,0725** |

El resultado muestra una diferencia importante entre escalas: el modelo explica cerca del 48 % de la variabilidad del objetivo transformado, pero solo alrededor del 7,2 % en unidades originales. El MAE y, especialmente, el RMSE en unidades reflejan errores relevantes en productos con volúmenes altos de compras.

El análisis de las predicciones también evidencia una tendencia a subestimar la cola alta del objetivo. En el test, el error absoluto medio aumenta notablemente en los segmentos de mayores ventas; para productos con más de 9.500 compras, la subestimación es especialmente marcada. Por ello, Ridge puede describir parte de la tendencia general en escala logarítmica, pero sus predicciones en unidades originales deben interpretarse con cautela cuando se trata de productos de alta demanda.

#### Comparación con la versión anterior y limitaciones

La versión anterior seleccionaba `alpha` con un único conjunto de validación y había elegido el límite inferior de su rejilla (`0,0001`). La nueva búsqueda con cinco folds seleccionó un valor mayor (`31,6228`), pero las métricas de test permanecieron prácticamente iguales: no hay evidencia de una mejora predictiva material. La mejora de esta versión es principalmente metodológica: selección más robusta de `alpha`, preprocesamiento ajustado dentro de cada fold y separación del test.

Aunque Ridge y Random Forest/MLP usan particiones 70/15/15 de v4 con el mismo tamaño de test, los índices de Ridge son diferentes. Por tanto, sus métricas no son una comparación pareada ni permiten establecer por sí solas un ranking definitivo. Además, el desempeño bajo en escala original y la subestimación de ventas altas son limitaciones que deben quedar explícitas al presentar el modelo.

El notebook exporta el pipeline final (`ridge_final.joblib`), `metrics.json`, `best_hyperparameters.json`, `cross_validation_results.csv`, `hyperparameter_search.csv`, `test_predictions.csv` y `split_indices.json`. Estos archivos permiten consultar tanto la selección de hiperparámetros como la evaluación final sin volver a entrenar el modelo.

### Red neuronal MLP Regressor

El MLP es una red neuronal de capas densas que aprende transformaciones no lineales de las características mediante pesos, sesgos y funciones de activación. La arquitectura `(128, 64, 32)` representa el número de neuronas de sus capas ocultas; la estandarización facilita la optimización y el *early stopping* detiene el entrenamiento cuando la validación deja de mejorar, ayudando a limitar el sobreajuste. Su flexibilidad también implica mayor sensibilidad a la arquitectura, la inicialización y los hiperparámetros que en modelos más simples.

[`Red_Neuronal_MLPRegressor.ipynb`](Modelos%20entrenados/Red_Neuronal_MLPRegressor.ipynb) también
usa v4 y la misma partición aleatoria 70/15/15 de Random Forest. Su preprocesamiento escala
variables numéricas y aplica One-Hot Encoding a `buy_box_status`. La búsqueda prueba 12
configuraciones mediante `RandomizedSearchCV` con 3 folds en el conjunto de entrenamiento; selecciona
por RMSE logarítmico. El MLP final usa capas `(128, 64, 32)`, activación `tanh`, `alpha=0.0001`,
tasa inicial de aprendizaje `0.001` y early stopping.

En el test aleatorio compartido obtuvo MAE log = **0,2481**, RMSE log = **0,5394**, R² log =
**0,7950**, MAE original = **351,4 ventas** y R² original = **0,7728**. Se guardan además la curva
de pérdida, la búsqueda de hiperparámetros y las predicciones.

### Resumen de resultados guardados

| Modelo y variante | Test usado | MAE log | RMSE log | R² log | MAE en unidades |
|---|---|---:|---:|---:|---:|
| KNN v1 | Agrupado, 4.887 filas | 0,5000 | 0,7617 | 0,4527 | 493,3 |
| SVR v1 | Agrupado, 4.887 filas | 0,4834 | 0,7482 | 0,4720 | 527,8 |
| Random Forest v4 | Aleatorio, 4.681 filas | 0,1772 | 0,4463 | 0,8596 | 280,2 |
| MLP v4 | Aleatorio, 4.681 filas | 0,2481 | 0,5394 | 0,7950 | 351,4 |
| Ridge v4 | Aleatorio, 4.681 filas (split propio) | 0,6081 | 0,8776 | 0,4800 | 990,9 |

**Importante:** los resultados de Random Forest y MLP son muy buenos en su test aleatorio, mientras
KNN y SVR se reportan sobre el test agrupado. No se debe concluir solo con esta tabla que los
primeros generalizan mejor: el cambio de partición, tamaño del test y posibilidad de encontrar
vectores de features observados previamente cambian la tarea medida. Dentro del par RF/MLP la
comparación sí usa exactamente los mismos índices; dentro del par KNN/SVR también.

### Artefactos generados

| Carpeta | Contenido principal |
|---|---|
| `Modelos entrenados/Models_Results/knn/` | `metrics.json`, `results_summary.csv`, `hyperparameter_search.csv`, `grid_agg.csv`, `test_predictions.csv`, `split_indices.json`, `knn_final.joblib` y gráficas. |
| `Modelos entrenados/Models_Results/svm/` | `metrics.json`, `validation_summary.csv`, `hyperparameter_search.csv`, `test_predictions.csv`, `error_by_sales_category.csv`, `svr_v1_final.joblib`–`svr_v4_final.joblib` y gráficas. |
| `Modelos entrenados/Models_Results/results_random_forest/` | `metrics.json`, `hyperparameter_search.csv`, `test_predictions.csv`, `split_indices.json`, importancia de variables, árbol, residuos y modelo serializado. |
| `Modelos entrenados/Models_Results/mlp/` | `metrics.json`, `best_hyperparameters.json`, `hyperparameter_search.csv`, `test_predictions.csv`, `split_indices.json`, curva de pérdida y modelo serializado. |
| `Modelos entrenados/Models_Results/results_ridge/` | `metrics.json`, `best_hyperparameters.json`, `cross_validation_results.csv`, `hyperparameter_search.csv`, `test_predictions.csv`, `split_indices.json` y pipeline Ridge serializado. |

Los detalles de configuración, validación, limitaciones y resultados completos están en los notebooks
y en los archivos `metrics.json` de cada carpeta.

---

## Datos versionados

Los `dataset_v*.csv` son **regenerables** ejecutando el notebook y no necesitan subirse al
repositorio. El CSV crudo (`Limpieza de datos/amazon_sales_data_uncleaned.csv`) sí se versiona
como fuente de verdad del proyecto.

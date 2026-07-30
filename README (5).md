# Resultados

Salidas de la ejecución que sustenta las tablas y figuras de la memoria. Se versionan
deliberadamente: permiten consultar los resultados sin reejecutar el cuaderno, que requiere
descargar el conjunto original y varios minutos de cálculo.

Todas se regeneran ejecutando `Script_4_modelos.ipynb` de arriba abajo.

## Contenido

| Archivo | Descripción | Correspondencia en la memoria |
|---|---|---|
| `resultados_comparativa.csv` | ROC-AUC, PR-AUC, F1, recall y precisión de los cuatro modelos | Tablas 2, 3 y 4 |
| `comparativa_4_modelos.png` | Gráfico de barras de las tres métricas por modelo | Figura 5 |
| `curvas_validacion_pna.png` | PR-AUC de validación época a época de ambas redes, con la mejor época señalada | — |
| `curvas_validacion_pna.csv` | Historial numérico de esas curvas (época, BCE, PR-AUC) | — |
| `shap_beeswarm.png` | Distribución de valores SHAP por variable en el modelo híbrido | Figura 6 |
| `shap_importancia_barras.png` | Importancia media \|SHAP\| ordenada | — |
| `resultados_shap.csv` | Importancia media por variable y peso relativo agregado de los embeddings | — |

## Notas de lectura

**`resultados_comparativa.csv`** incluye el ROC-AUC aunque no sea la métrica principal del
estudio. Su función es ilustrativa: los cuatro modelos se sitúan en una banda estrecha
mientras el PR-AUC casi se duplica del peor al mejor, lo que evidencia cómo un indicador de
uso generalizado puede enmascarar las diferencias reales bajo un desbalanceo del 0,42 %.

**`curvas_validacion_pna.*`** documentan la parada temprana. El número de épocas de cada red
no es un parámetro fijado de antemano, sino el resultado de vigilar el PR-AUC sobre la
partición de validación; estas curvas son la evidencia de que la parada actuó donde debía.

**`resultados_shap.csv`** distingue las variables explícitas de las dimensiones latentes
(`emb_0` a `emb_15`) y agrega el peso total de estas últimas, lo que permite cuantificar
cuánto de la decisión final procede de la representación aprendida por la red.

## Reproducibilidad

La semilla está fijada (`SEED = 42`), de modo que los tres modelos basados en árboles son
deterministas. En los dos que incorporan la red neuronal pueden aparecer variaciones menores
según la versión de PyTorch y el dispositivo de cálculo, ya que algunas operaciones de
agregación de PyTorch Geometric no son deterministas en GPU.

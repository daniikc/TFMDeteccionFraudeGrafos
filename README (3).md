# Detección de fraude financiero utilizando grafos

Código correspondiente al Trabajo Fin de Máster **«Detección de Fraude Financiero utilizando
Grafos»** (Máster Universitario en Inteligencia Artificial, Universidad Tecnológica
Atlántico-Mediterráneo, 2026).

El trabajo aborda la detección de blanqueo de capitales reformulando el problema como una
**clasificación de nodos** sobre un grafo transaccional, y compara cuatro arquitecturas de
complejidad creciente para cuantificar qué aporta exactamente la información estructural.

| # | Modelo | Información que utiliza |
|---|--------|-------------------------|
| 1 | XGBoost sin grafo | 16 variables tabulares |
| 2 | XGBoost con grafo | las 16 tabulares + 17 estructurales explícitas (33) |
| 3 | PNA en solitario | las 33 variables + estructura de la red vía paso de mensajes |
| 4 | Híbrido PNA + XGBoost | las 33 del modelo 2 + 16 embeddings de la PNA |

Los cuatro modelos operan sobre el mismo conjunto de 33 variables y bajo idéntico protocolo de
evaluación, de modo que el salto **1→2** aísla la aportación de las métricas estructurales
calculadas de antemano y el salto **2→4** la de los embeddings aprendidos por la red.

---

## Estructura del repositorio

```
DeteccionFraudeGrafosHibrido/
├── README.md
├── requirements.txt
├── .gitignore
├── generar_dataset_solo_patrones.py    Paso 1 · reetiquetado del conjunto
├── Script_4_modelos.ipynb              Paso 2 · estudio comparativo completo
├── datos/
│   ├── README.md                       Cómo obtener el conjunto original
│   └── patterns.txt                    Patrones de blanqueo de AMLworld
└── resultados/
    ├── README.md                       Descripción de cada salida
    └── ...                             Métricas, curvas y figuras
```

Los dos archivos ejecutables quedan en la raíz y se lanzan en ese orden. Las carpetas separan
lo que no es código:

- **`datos/`** contiene el insumo del reetiquetado. Los CSV de transacciones **no se
  versionan**: ocupan cientos de MB y se regeneran con el script.
- **`resultados/`** contiene las salidas **sí versionadas**, como evidencia de la ejecución
  que sustenta las tablas y figuras de la memoria. Permiten consultar los resultados sin
  descargar el conjunto original ni reejecutar el cuaderno.

---

## Instalación

Entorno de referencia: **Python 3.10**.

```bash
git clone https://github.com/daniikc/DeteccionFraudeGrafosHibrido.git
cd DeteccionFraudeGrafosHibrido

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

Si la instalación conjunta de PyTorch Geometric falla, instálese en dos pasos:

```bash
pip install torch==2.5.1
pip install torch-geometric==2.6.1
```

> ### Aviso sobre la versión de XGBoost
>
> `requirements.txt` fija **`xgboost==2.1.4`** y no debe actualizarse a la rama 3.x. Desde la
> versión 3.0, XGBoost serializa el parámetro `base_score` como cadena de lista (`'[5E-1]'`)
> en lugar de como número real; el cargador de modelos de SHAP intenta convertirlo con
> `float()` y aborta, lo que impide construir `TreeExplainer` y deja sin efecto el análisis de
> interpretabilidad. La corrección existe en SHAP, pero solo en versiones que exigen
> Python ≥ 3.11.
>
> El pin no altera los resultados: algoritmo, hiperparámetros y predicciones son equivalentes.
> La primera celda del cuaderno comprueba la versión y detiene la ejecución si detecta una
> incompatible, para que el problema no aparezca al final del proceso.

---

## Pasos seguidos

### Paso 0 · Obtener el conjunto de datos

El conjunto original no se incluye por su tamaño. Procede del conjunto sintético
**IBM AMLworld**, variante *HI*, publicado por Altman et al. (2023). Las instrucciones están
en [`datos/README.md`](datos/README.md).

### Paso 1 · Reetiquetado: conservar solo los patrones estructurales

```bash
python generar_dataset_solo_patrones.py \
       datos/patterns.txt \
       datos/Transacciones_mod_hi.csv \
       datos/Transacciones_solo_patrones.csv
```

El conjunto original marca como blanqueo tanto las operaciones que forman parte de un patrón
estructural —recogidas en `patterns.txt`— como las de la fase de **integración**, que son
movimientos aislados hacia la economía formal. Estas últimas no dejan huella topológica
alguna: desde la red resultan indistinguibles de cualquier operación legítima, y ninguna
métrica de grafo puede detectarlas.

El script conserva por tanto la etiqueta positiva en las transacciones pertenecientes a
cualquiera de los ocho tipos de patrón y **reetiqueta a 0 las de integración**. El cruce se
realiza por marca de tiempo, cuenta de origen, cuenta de destino e importes.

**Ninguna fila se elimina.** Las transacciones reetiquetadas permanecen en el conjunto y
siguen contribuyendo a la topología del grafo: solo cambia la variable objetivo.

Distribución de los 3.209 movimientos que conservan la etiqueta positiva:

| Patrón | Bloques | Transacciones |
|---|---:|---:|
| Gather-scatter | 51 | 716 |
| Scatter-gather | 44 | 626 |
| Stack | 43 | 466 |
| Fan-out | 48 | 342 |
| Fan-in | 40 | 318 |
| Cycle | 54 | 287 |
| Bipartite | 49 | 263 |
| Random | 41 | 191 |
| **Total** | **370** | **3.209** |

### Paso 2 · Ejecutar el estudio comparativo

```bash
jupyter notebook Script_4_modelos.ipynb
```

Ejecútense las celdas en orden, **desde la raíz del repositorio**: el cuaderno lee de `datos/`
y escribe en `resultados/` mediante rutas relativas.

Fases del cuaderno:

1. **Carga y filtrado.** Se descartan las transacciones de una cuenta consigo misma, que
   inflarían simultáneamente los grados de entrada y de salida sin que exista relación con
   ninguna otra cuenta.
2. **Detección de ciclos temporales.** Caminos cerrados de longitud 3 a 8 con marcas de
   tiempo no decrecientes en los que el importe se conserva aproximadamente (cociente entre
   el final y el inicial dentro del intervalo 0,75–1,25).
3. **Construcción de la tabla de nodos.** Una función unificada calcula las 33 variables de
   cada cuenta, de modo que los cuatro modelos parten de la misma tabla.
4. **Partición temporal.** Días 1–3 para entrenamiento, días 4–6 para prueba.
5. **Entrenamiento y evaluación** bajo protocolo común: idéntico espacio de hiperparámetros
   para los tres XGBoost e idéntica arquitectura y criterio de parada para las dos redes.
6. **Análisis SHAP** del modelo híbrido.

> La celda de construcción de nodos es la más costosa: la detección de ciclos recorre el
> grafo completo de cada partición y puede tardar varios minutos.

---

## Decisiones metodológicas relevantes

**Propagación de las etiquetas a los nodos.** El conjunto está etiquetado a nivel de
transacción, pero la unidad de análisis es la cuenta. Se adopta un criterio inclusivo: una
cuenta se etiqueta como fraudulenta si participa, **como emisora o como receptora**, en al
menos una transacción marcada dentro de la ventana temporal. En un ciclo o en una estructura
de tipo *gather-scatter* ninguna cuenta implicada ocupa una posición accesoria, y excluir a
las intermediarias supondría descartar aquellas cuya posición resulta más reveladora. La
contrapartida es que la clase positiva no representa el conjunto de cuentas controladas por
una organización, sino el de cuentas implicadas estructuralmente en un patrón.

**Aristas dirigidas en el paso de mensajes.** El tensor de aristas conserva el sentido
económico de la operación: cada cuenta agrega información de aquellas que le envían fondos.
Se ensayó una variante con aristas en ambos sentidos, bajo la hipótesis de que permitiría a
las cuentas dispersoras recibir información de sus destinatarios, pero el rendimiento resultó
inferior y se descartó. La interpretación más plausible es que la direccionalidad constituye
en sí misma una señal discriminante: introducir la arista inversa la diluye, igualando el
vecindario de una cuenta receptora y el de una emisora.

**Pesos de arista agregados.** Antes de construir el grafo se suman los importes de todas las
transacciones entre cada par de cuentas. Pasar el DataFrame de transacciones directamente a
`from_pandas_edgelist` haría que cada arista repetida sobrescribiese el atributo de la
anterior, con lo que el peso sería el de la última operación y no el total transferido, en
contra de lo que exige la formulación del PageRank ponderado.

**Tratamiento del desbalanceo.** No se recurre a remuestreo (SMOTE, submuestreo aleatorio),
sino a los mecanismos de ponderación de clases propios de cada algoritmo —`scale_pos_weight`
en XGBoost, `pos_weight` en la función de pérdida de la red— complementados con un ajuste
posterior del umbral de decisión.

**Parada temprana sobre el PR-AUC de validación.** El número de épocas de cada red no se fija
de antemano: se vigila el PR-AUC sobre una partición de validación del 25 % y se restaura el
estado de la mejor época. El criterio es la métrica principal del estudio y no la pérdida,
porque con una ponderación positiva de este orden la entropía cruzada puede seguir bajando
mientras el ordenamiento de la clase minoritaria ya se degrada.

**Protocolo del umbral.** El umbral **no** se elige sobre el conjunto de prueba, sino sobre esa
misma partición de validación, maximizando el F1 de la clase minoritaria. El PR-AUC, que es la
métrica principal, es independiente del umbral y por tanto ajeno a esta decisión: eso lo
convierte en el criterio limpio para comparar las cuatro arquitecturas.

---

## Resultados

| Modelo | PR-AUC | F1 minoritaria | Recall | Precisión |
|---|---:|---:|---:|---:|
| 1 · XGBoost sin grafo | 0,2690 | 0,1858 | 0,11 | 0,78 |
| 2 · XGBoost con grafo | 0,3061 | 0,3104 | 0,20 | 0,67 |
| 3 · PNA en solitario | 0,2187 | 0,2642 | **0,25** | 0,28 |
| 4 · Híbrido PNA + XGBoost | **0,4241** | **0,3681** | 0,23 | **0,90** |

El híbrido mejora el PR-AUC en 11,8 puntos sobre el mejor de los modelos anteriores —un
38,5 % en términos relativos— y en 15,5 puntos sobre la línea de referencia tabular (57,7 %).

El resultado más instructivo es el del modelo 3: la red en solitario obtiene el PR-AUC más
bajo del estudio pese a ser la que más cuentas fraudulentas recupera. Su cabeza clasificadora
es una única capa lineal sobre dieciséis dimensiones, insuficiente para convertir la señal
estructural en un buen ordenamiento. Al delegar la decisión en un clasificador de árboles, el
PR-AUC prácticamente se duplica (+93,9 %). La aportación del enfoque relacional no reside, por
tanto, en la capacidad discriminante de la red por sí sola, sino en la representación que
produce cuando se combina con un clasificador capaz de explotarla.

---

## Limitaciones conocidas

- Los resultados proceden de **una única ejecución** de cada modelo, sin repeticiones con
  distintas semillas, por lo que no van acompañados de una estimación de su variabilidad.
- En el modelo híbrido, el clasificador final se entrena sobre la totalidad del conjunto
  mientras los embeddings proceden de una red ajustada con el 75 % de las etiquetas. La
  separación temporal garantiza que ninguna información del periodo de prueba interviene en
  ninguna etapa, pero no puede descartarse cierta circularidad residual. Un protocolo de
  validación cruzada anidada para generar los embeddings lo resolvería.
- El gradiente del clasificador final **no retropropaga a la red**: los embeddings son
  características congeladas, no representaciones optimizadas para él. La arquitectura es
  secuencial y no conjunta, lo que es inevitable con un ensemble de árboles no diferenciable.
- La detección de ciclos es completa en cuanto a la coherencia temporal, pero no respecto al
  criterio de importe: se comprueba sobre una única secuencia válida por rotación, de modo que
  `En_Ciclo` puede infraestimar.
- Las variables temporales que dependen de la dispersión de los intervalos exigen al menos
  tres transacciones; las cuentas con dos o menos reciben un valor imputado. La celda de
  diagnóstico del cuaderno cuantifica la proporción afectada.
- Al recorrer el paso de mensajes solo las aristas entrantes, las cuentas sin operaciones de
  entrada no reciben mensaje alguno. Las métricas explícitas del vecindario cubren
  parcialmente esa carencia.
- La intermediación y la cercanía se definen en la memoria pero no se calculan, por su coste
  computacional sobre un grafo de esta magnitud.

---

## Cita

```
Kock Cabrera, D. H. (2026). Detección de fraude financiero utilizando grafos
[Trabajo Fin de Máster]. Universidad Tecnológica Atlántico-Mediterráneo.
```

## Conjunto de datos

```
Altman, E., Blanuša, J., von Niederhäusern, L., Egressy, B., Anghel, A., y Atasu, K. (2023).
Realistic synthetic financial transactions for anti-money laundering models.
Advances in Neural Information Processing Systems, 36. https://arxiv.org/abs/2306.16424
```

## Licencia

Código publicado con fines académicos. El conjunto de datos original se rige por la licencia
de su distribución de origen.

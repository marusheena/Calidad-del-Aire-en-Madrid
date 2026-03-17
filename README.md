# 🌫️ Calidad del Aire en Madrid — AQI Personalizado con Machine Learning

> **Proyecto ganador del Premio al Mejor Proyecto de Data Science** de la promoción · The Bridge Digital Talent Accelerator

---

## ¿De qué trata este proyecto?

Madrid, como muchas grandes ciudades europeas, mide la calidad del aire usando el **AQI** (Air Quality Index), un índice estandarizado que solo tiene en cuenta la concentración de contaminantes atmosféricos (NO₂, PM2.5, PM10, O₃, CO…). Es como si para medir tu salud solo miraras la temperatura corporal e ignoraras todo lo demás.

Este proyecto va más allá: **construye un AQI personalizado que integra, por primera vez de forma conjunta, datos de contaminantes, condiciones meteorológicas, tráfico rodado y eventos especiales**, para predecir con mayor precisión los niveles de contaminación en diferentes zonas de la ciudad.

El resultado es un modelo de Machine Learning (XGBoost) que alcanza un **R² = 0.89** a nivel global de Madrid y un **RMSE de 2.76** en predicciones por zona, lo que lo convierte en una herramienta robusta para anticipar episodios de contaminación.

---

## ¿Por qué es innovador en el contexto europeo?

En Europa existen sistemas de monitoreo de calidad del aire bien establecidos (EEA, ACEA, redes nacionales), pero la mayoría de los índices públicos siguen el modelo EPA tradicional: **solo miden contaminantes, no predicen ni contextualizan**.

Este proyecto innova en tres frentes:

1. **AQI dinámico y contextualizado**: El índice se ajusta en función de variables meteorológicas (temperatura, viento, humedad, presión) y de tráfico. Un día con mucho NO₂ pero con viento fuerte tiene un impacto real diferente al mismo nivel en calma atmosférica.

2. **Análisis del impacto de las Zonas de Bajas Emisiones (ZBE)**: Madrid fue una de las primeras ciudades españolas en implementar restricciones de acceso vehicular por zonas (Madrid Central, Plaza Elíptica, Madrid ZBE). Este proyecto **cuantifica el impacto real de estas políticas** en la calidad del aire usando una variable propia: `diferencia_trafico_en_zbe_centro`.

3. **Granularidad geográfica real**: En lugar de un único valor para toda la ciudad, el modelo genera predicciones **por zona geográfica**, reconociendo que el interior de la M-30 se comporta de forma radicalmente distinta al extrarradio sur.

---

## El equipo

Proyecto desarrollado por **Alejandra, Maruxa y Raquel**, como Capstone Project del Bootcamp de Data Science en The Bridge Digital Talent Accelerator (2025).

---

## Estructura del proyecto y flujo de trabajo

El proyecto sigue un pipeline de datos end-to-end, desde los datos brutos hasta los modelos predictivos finales. A continuación se describe cada etapa con el notebook correspondiente.

---

### Etapa 1 — Entender el territorio: Clasificación de estaciones y Zonas de Bajas Emisiones

**📓 [`VF_identificacion_ZBE.ipynb`](VF_identificacion_ZBE.ipynb)**

Antes de modelar, hay que saber dónde están los sensores y qué miden. Este notebook carga y analiza las **24 estaciones de medición de calidad del aire de Madrid** (datos 2021–2025) e identifica qué contaminantes registra cada una.

Lo más importante: usando coordenadas geográficas y polígonos construidos con la librería `Shapely`, **cada estación se clasifica como "dentro" o "fuera" de cada ZBE** (Zona de Bajas Emisiones). Esto es la base para todo el análisis posterior del impacto de las políticas de movilidad.

> **Para no técnicos**: Es como dibujar los límites de Madrid Central en un mapa y marcar qué contadores de contaminación están dentro y cuáles fuera.

---

### Etapa 2 — Procesar el tráfico: De millones de registros a una variable útil

**📓 [`dataset_tráfico_por_zonas.ipynb`](dataset_tráfico_por_zonas.ipynb)**

Madrid cuenta con **4.725 sensores de tráfico** distribuidos por la ciudad. Este notebook procesa **36 archivos CSV mensuales** (2021–2025) con millones de registros de intensidad, ocupación y velocidad media, y los **agrega diariamente por zona geográfica**.

Se definen 5 zonas de análisis (Interior M-30, Sureste, Noreste, Noroeste, Suroeste) asignando cada sensor a la zona más cercana mediante cálculo de distancia a centroides. El resultado es un dataset manejable y geográficamente estructurado, listo para integrarse con los datos de calidad del aire.

---

### Etapa 3 — La variable clave: Impacto diferencial de las ZBE en el tráfico

**📓 [`VF_Analisis_diferencia_trafico_dentro_fuera_ZBE.ipynb`](VF_Analisis_diferencia_trafico_dentro_fuera_ZBE.ipynb)**

Este es uno de los análisis más originales del proyecto. Para cada día del periodo estudiado, se calcula la diferencia entre el tráfico medio **dentro** de la ZBE y el tráfico medio **fuera** de la ZBE:

```
diferencia_trafico_en_zbe_centro = tráfico_medio_dentro − tráfico_medio_fuera
```

Los resultados son reveladores:
- La distribución tiende a ser **negativa** (menos tráfico dentro que fuera), coherente con las restricciones de acceso.
- Durante el **COVID-19 (2020-2021)** la diferencia colapsó hacia cero, ya que el tráfico se redujo homogéneamente en toda la ciudad.
- En **2022**, con la entrada en vigor de nuevas ZBE, se detecta un posible **"efecto frontera"**: el tráfico se desplaza hacia los límites exteriores.

Esta variable, junto con medias móviles de 30 días para suavizar el ruido diario, se incorpora como predictor en los modelos de Machine Learning.

---

### Etapa 4 — Enriquecer los datos: Eventos especiales en Madrid

**📓 [`Variable_eventos especiales.ipynb`](Variable_eventos especiales.ipynb)**

Los días de grandes conciertos, partidos de fútbol o manifestaciones tienen un comportamiento atípico en el tráfico y, por ende, en la calidad del aire. Este notebook crea una variable binaria (`evento_especial`: 1/0) para cada día del periodo, identificando manualmente los eventos relevantes a partir de fuentes oficiales y portales de eventos de Madrid.

Aproximadamente el **1.9% de los días** quedan marcados como eventos especiales, un dato pequeño pero con alto poder explicativo en los outliers del modelo.

---

### Etapa 5 — El modelo principal: AQI personalizado para toda Madrid

**📓 [`AQI_personalizada_general.ipynb`](AQI_personalizada_general.ipynb)**

Con todas las variables preparadas, este notebook construye el modelo central del proyecto sobre el dataset `Final_data.csv` (1.549 días × 33 variables).

**¿Cómo se construye el AQI personalizado?**

1. Se calcula el **AQI base** siguiendo los breakpoints oficiales de la EPA para cada contaminante (CO, NO₂, SO₂, O₃, PM10, PM2.5) y se toma el valor máximo.
2. Se aplican **factores multiplicativos de ajuste** según condiciones meteorológicas y de tráfico, reflejando cómo el contexto real modula el impacto de los contaminantes en la salud.
3. El AQI resultante se usa como variable objetivo para entrenar y comparar múltiples modelos de Machine Learning.

**Modelos evaluados:**

| Modelo | R² (Test) | RMSE (Test) |
|---|---|---|
| Linear Regression | ~0.72 | ~9.1 |
| K-Nearest Neighbors | ~0.81 | ~7.5 |
| Random Forest | ~0.86 | ~6.2 |
| HistGradientBoosting | ~0.88 | ~5.4 |
| **XGBoost** ✅ | **~0.89** | **~5.07** |

El modelo ganador es **XGBoost**, optimizado con `GridSearchCV`. Las visualizaciones incluyen comparativas de R² entre modelos, scatter plots de valores reales vs. predichos y análisis de importancia de variables.

---

### Etapa 6 — Predicción por zonas: granularidad geográfica

**📓 [`AQI_personalizado_por_zonas_trafico.ipynb`](AQI_personalizado_por_zonas_trafico.ipynb)**

El mismo enfoque del notebook anterior pero aplicado **por zona geográfica**, usando el dataset `df_zonas_final.csv` (7.445 filas = 5 zonas × ~1.490 días).

Los modelos por zona capturan patrones locales que el modelo global no puede ver: el interior de la M-30 responde de forma diferente a las restricciones de ZBE que las zonas periféricas. El mejor modelo por zona (**XGBoost**) alcanza un **RMSE de 2.76**, más preciso que el modelo global gracias a esta granularidad.

Las visualizaciones incluyen mapas interactivos con `folium` mostrando la distribución de estaciones por zona y la evolución del AQI en el tiempo por área geográfica.

---

### Etapa 7 — Aprendizaje no supervisado: ¿Qué tipos de días hay en Madrid?

**📓 [`Analisis_no_supervisado_Capstone.ipynb`](Analisis_no_supervisado_Capstone.ipynb)**

Sin usar ninguna etiqueta previa, el algoritmo **K-Means** agrupa los 1.549 días del periodo en **3 clusters** con perfiles ambientales diferenciados. El número óptimo de clusters se determina combinando el **Elbow Method** y el **Silhouette Score**.

Los 3 arquetipos de días que emergen son:

- **Cluster 1 — Días de alta contaminación**: NO₂ elevado, viento escaso, temperaturas bajas. Típicamente días de invierno en el interior urbano.
- **Cluster 2 — Días de calidad moderada**: Condiciones intermedias, patrón más común durante otoño y primavera.
- **Cluster 3 — Días de buena calidad del aire**: Temperaturas altas, viento fuerte. El viento dispersa los contaminantes; suelen ser días de verano o días con precipitaciones intensas.

Estos clusters se incorporan como variable categórica adicional en el modelo supervisado, mejorando su capacidad predictiva.

---

## Stack tecnológico

| Categoría | Herramientas |
|---|---|
| Lenguaje | Python 3.10+ |
| Manipulación de datos | `pandas`, `numpy` |
| Visualización | `matplotlib`, `seaborn`, `folium` |
| Machine Learning | `scikit-learn`, `xgboost` |
| Geometría / Geo | `shapely` |
| Optimización de modelos | `GridSearchCV`, `RandomizedSearchCV` |
| Entorno | Jupyter Notebooks |

---

## Datos utilizados

| Fuente | Descripción | Periodo |
|---|---|---|
| Ayuntamiento de Madrid (Portal de Datos Abiertos) | 24 estaciones de calidad del aire (NO₂, PM10, PM2.5, O₃, CO, SO₂) | 2021–2025 |
| Ayuntamiento de Madrid | 4.725 sensores de tráfico (intensidad, velocidad, ocupación) | 2021–2025 |
| AEMET / Meteomática | Variables meteorológicas diarias (temperatura, humedad, viento, presión) | 2021–2025 |
| Fuentes propias | Calendario de eventos especiales en Madrid | 2021–2025 |

---

## Resultados principales

- **AQI personalizado** más preciso y contextualizado que el índice EPA estándar.
- Modelo XGBoost global: **R² = 0.89**, RMSE = 5.07
- Modelo XGBoost por zona: **RMSE = 2.76** (mejora significativa por granularidad geográfica)
- Identificación de **3 perfiles de días** con características ambientales diferenciadas.
- Evidencia cuantitativa del **impacto de las ZBE** en el tráfico y, transitivamente, en la calidad del aire.
- Detección del efecto COVID-19 y del efecto frontera en las restricciones de movilidad.

---

## Reconocimientos

Este proyecto fue reconocido como el **Mejor Proyecto de Data Science de la promoción** en The Bridge Digital Talent Accelerator (2025).

---

*Datos de uso público. Proyecto con fines académicos y de investigación.*

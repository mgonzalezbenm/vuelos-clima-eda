# ✈️ Impacto del clima en la puntualidad de vuelos (Q1 2025)

### Análisis de retrasos y cancelaciones en vuelos domésticos de EE. UU. y su relación con el clima en origen

[![Python](https://img.shields.io/badge/Python-3.9-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![pandas](https://img.shields.io/badge/pandas-data%20wrangling-150458?logo=pandas&logoColor=white)](https://pandas.pydata.org/)
[![scipy](https://img.shields.io/badge/scipy-estad%C3%ADstica-8CAAE6?logo=scipy&logoColor=white)](https://scipy.org/)
[![Power BI](https://img.shields.io/badge/Power%20BI-Dashboard-F2C811?logo=powerbi&logoColor=black)](https://powerbi.microsoft.com/)
[![Status](https://img.shields.io/badge/status-completado-brightgreen)]()

---

## Descripción del proyecto

¿Cuánto pesa realmente el clima a la hora de que un vuelo llegue tarde o se cancele, comparado con otros factores como la aerolínea o la hora del día? Este proyecto intenta responder a esa pregunta con datos reales: un trimestre completo (enero-marzo 2025) de vuelos domésticos en Estados Unidos, cruzado con el clima diario en cada aeropuerto de origen.

La fuente de vuelos es el histórico que publica el *Bureau of Transportation Statistics* (BTS); el clima viene de la librería `meteostat`. Todo el proceso, desde la primera auditoría de calidad hasta los tests estadísticos, está en cuatro notebooks documentados, y los resultados se pueden explorar de forma interactiva en un dashboard de Power BI.

Spoiler del resultado principal: el clima importa, y la nieve en particular, pero no explica ni de lejos la mayor parte del problema. Los detalles están más abajo.

---

## Datos de partida

| Dataset | Fuente | Volumen | Contenido |
|---|---|---|---|
| Reporting Carrier On-Time Performance (3 CSV, ene–mar 2025) | [BTS TranStats](https://www.transtats.bts.gov/) | ~500–600 mil filas/mes, 110 columnas originales | Horarios, retrasos, causas de retraso, cancelaciones y desvíos, vuelo a vuelo |
| Clima diario por aeropuerto | `meteostat` 1.7.6 (API clásica: `Point`, `Daily`, `.fetch()`) | Diario, 35 aeropuertos, 11 variables | Temperatura, precipitación, viento, presión, nieve |
| Coordenadas de aeropuertos (apoyo) | [OurAirports](https://ourairports.com/) | 333 códigos IATA | Latitud/longitud para geolocalizar cada aeropuerto |

Unión por código IATA del aeropuerto de origen + fecha del vuelo.

Los tres CSV del BTS pesan más de 100 MB cada uno y no van en el repositorio; en la sección de reproducibilidad explico dónde descargarlos. El clima y las coordenadas sí están en `data/raw/`.

---

## Estructura del repositorio

```
vuelos-clima-eda/
│
├── data/
│   ├── raw/
│   │   ├── bts/                          # CSV de vuelos BTS (no versionado, >100MB c/u)
│   │   ├── clima/
│   │   │   └── clima_diario_2025q1.csv
│   │   └── apoyo/
│   │       └── airports.csv
│   └── processed/
│       └── vuelos_clima_2025q1.parquet   # dataset final, 1.138.858 filas × 49 columnas (25MB)
│
├── notebooks/
│   ├── 01_auditoria.ipynb
│   ├── 02_limpieza.ipynb
│   ├── 03_eda.ipynb
│   └── 04_estadistica.ipynb
│
├── dashboard/
│   ├── vuelos_clima_dashboard.pbix
│   └── dashboard.pdf
│
├── reports/
│   ├──Informe_final_vuelos_clima.pdf
│   └── figures/
│
├── src/
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Tecnologías

Python 3.9.6, pandas 2.3.3, numpy 2.0.2, matplotlib 3.9.4, seaborn 0.13.2, scipy 1.13.1, statsmodels 0.14.6, meteostat 1.7.6, pyarrow 21.0.0 y Power BI Desktop para el dashboard final.

---

## Cómo se ha trabajado

El flujo es el habitual en un proyecto de este tipo, pero con una regla que he intentado seguir a rajatabla: ninguna decisión de limpieza o umbral estadístico se toma por defecto, todas están justificadas en el propio notebook a partir de lo que muestran los datos.

```
Auditoría → Limpieza y transformación → Unión vuelos + clima
    → Análisis descriptivo → Estadística inferencial → Dashboard
```

Algunas de esas decisiones, resumidas:

- `TaxiOut`, `TaxiIn`, `AirTime` y `ArrTime` vienen nulos solo en vuelos cancelados o desviados. No es un dato que falte, es un dato que no existe para ese vuelo, así que no se imputa, y el análisis de retrasos se limita a vuelos en estado normal.
- El BTS solo rellena las causas de retraso (`CarrierDelay`, etc.) cuando `ArrDelay >= 15` min; lo comprobé empíricamente antes de fijar ese umbral como el de "retraso" en todo el análisis.
- 111 vuelos con `DepTime` pero sin `DepDelay` resultaron ser, todos, vuelos cancelados tras el despegue registrado.
- `Tail_Number` falta en 4.102 registros, siempre en cancelados; se deja como está.
- Los horarios en formato HHMM se convierten a minutos desde medianoche, con `2400 → 0`. `ArrMin` se anula en desvíos, porque el avión aterriza en un aeropuerto distinto al de destino previsto.
- Los retrasos extremos (hasta ~3.400 minutos) son reales, no errores de carga, así que se quedan. Para no dejar que distorsionen las medias se trabaja con mediana y percentiles, tests no paramétricos, y un indicador aparte (`retraso_extremo`, >300 min) para el análisis de sensibilidad.
- `snow_orig` tiene un patrón de nulos nada aleatorio: faltan sobre todo en aeropuertos donde casi nunca nieva (HNL, LAX, SFO, SMF). Rellenar eso con 0 sería inventar un dato, así que en su lugar se construye una proxy, `nieve_aprox`, a partir de precipitación y temperatura.
- El cruce vuelos-clima se valida como `merge` `m:1` sin pérdida ni duplicación de filas, comprobado con la columna `_merge`.

Resultado: **1.138.858 filas y 49 columnas**, cubriendo el 69,2% del tráfico total del trimestre en los 35 aeropuertos de origen seleccionados.

---

## Qué dicen los datos

A nivel descriptivo, la puntualidad global (vuelos normales, retraso menor a 15 min) es del **79,8%**, con una mediana de retraso de −7 minutos (es decir, los vuelos suelen llegar un poco antes de lo previsto). Cancelados: 1,74%. Desviados: 0,25%.

Por aerolínea hay diferencias notables (69,8% en OH frente a 83,7% en HA), pero el factor con el efecto más visible a simple vista es la franja horaria: 91,2% de puntualidad en vuelos de madrugada frente a 74,8% en los de noche. El día de la semana también influye, aunque menos (martes y miércoles algo mejores, domingo el peor).

Y el clima: con clima adverso la puntualidad baja de 81,8% a 67,9%. Con nieve, la caída es mucho más marcada: de 80,8% a 56,4%, el efecto más fuerte de todo el análisis descriptivo. Las cancelaciones se multiplican por cuatro tanto con clima adverso como con nieve.

Ahora bien, cuando se pasa a estadística inferencial (Mann-Whitney, Spearman, Kruskal-Wallis, chi-cuadrado, todos significativos tras corrección FDR por el tamaño muestral) el panorama se matiza: los tamaños de efecto son **pequeños en todos los casos**, incluida la nieve (r = 0,105). El que más pesa de todos es la franja horaria (ε² = 0,0161), justo por delante de la nieve y muy por delante de la aerolínea (ε² = 0,0065, a pesar de tener 14 categorías).

La lectura conjunta: el clima, y sobre todo la nieve, es real y perceptible, pero ningún factor por separado explica gran parte de por qué un vuelo llega tarde. Es lo esperable en un sistema tan interconectado como el tráfico aéreo, donde un retraso en un aeropuerto se propaga a otros vuelos de la misma aeronave o tripulación.

> Nota técnica para quien reutilice el pipeline: con más de un millón de filas, `scipy.stats.mannwhitneyu` puede devolver un p-valor de 0.0 por desbordamiento de precisión. El tamaño del efecto r hay que calcularlo desde el estadístico U directamente (z = (U − n₁n₂/2) / √(n₁n₂(n+1)/12)), no a partir del p-valor.

---

## Dashboard (Power BI)

`dashboard/vuelos_clima_dashboard.pbix` (exportado también a `dashboard/dashboard.pdf`) recoge lo anterior en un formato explorable:

- KPIs generales: % puntualidad, % cancelados, % desviados, total de vuelos, retraso medio.
- Mapa de los 35 aeropuertos de origen, con el volumen de tráfico por tamaño de burbuja.
- Evolución de vuelos y puntualidad a lo largo del trimestre, con jerarquía de fecha y *drill-down* de mes a día. A nivel mensual se ve con claridad cómo volumen y puntualidad caen juntos en febrero (el mes con más nieve) y se recuperan en marzo; bajando a día se identifican episodios concretos de mal tiempo.
- Puntualidad por franja horaria, que traslada al dashboard el factor con mayor peso relativo encontrado en el análisis inferencial.
- Segmentadores por aerolínea y por clima adverso, que filtran todos los visuales a la vez.

---

## Conclusiones y qué haría con esto una aerolínea o un aeropuerto

- **La nieve es el factor climático que de verdad importa; la lluvia o el viento, mucho menos.** Si hay que priorizar recursos de contingencia (deshielo, personal extra, reprogramación preventiva), tiene más sentido concentrarlos en los aeropuertos y meses con nieve que en un plan genérico "clima adverso".
- **La franja horaria pesa más que el clima.** Cualquier estrategia para mejorar la puntualidad que no toque la planificación de horarios (por ejemplo, evitar acumular vuelos en la franja de noche) va a dejar sobre la mesa una palanca más grande que la meteorológica.
- **Ningún factor aislado explica el problema**, así que un modelo de predicción de retrasos que solo mire el clima, o solo la aerolínea, se va a quedar corto. Tendría sentido combinar clima + franja horaria + aerolínea en un mismo modelo antes de sacar conclusiones operativas definitivas.
- **La cancelación es mucho más sensible al clima que el retraso** (se multiplica por cuatro), lo que sugiere que las decisiones de cancelar se toman con un criterio más conservador ante mal tiempo del que refleja el simple retraso medio.

---

## Limitaciones

`snow_orig` no es fiable tal cual (nulos no aleatorios), así que todo lo relativo a nieve se apoya en la proxy `nieve_aprox`, no en el dato directo de meteostat. El análisis cubre 35 aeropuertos que representan el 69,2% del tráfico del trimestre; lo que pasa en el 30,8% restante no está necesariamente reflejado aquí. Y, lo más importante: esto es un análisis de asociación, no un modelo causal; que el clima se relacione con el retraso no implica que el resto de factores (congestión de red, tripulaciones, mantenimiento) estén controlados. Un solo trimestre tampoco permite hablar de estacionalidad interanual.

---

## ¿Serviría esto para Eurocontrol o Aena?

En líneas generales, sí, sin cambiar apenas el enfoque. El pipeline (auditoría, limpieza documentada, cruce por aeropuerto y fecha, estadística descriptiva e inferencial con tamaño de efecto, dashboard) no depende de ninguna particularidad del BTS estadounidense. Habría que cambiar la fuente de vuelos y adaptar el código de aeropuerto (IATA/ICAO), pero `meteostat` también cubre estaciones europeas, así que la parte de clima se reutilizaría casi tal cual.

---

## Cómo reproducirlo

```bash
git clone https://github.com/mgonzalezbenm/vuelos-clima-eda.git
cd vuelos-clima-eda

python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Los CSV del BTS no están en el repo por tamaño. Se descargan desde [TranStats](https://www.transtats.bts.gov/) (tabla *Reporting Carrier On-Time Performance*, enero-marzo 2025) y se colocan en `data/raw/bts/`. El resto de datos, incluido el dataset final ya procesado, sí está en el repositorio, así que se puede ir directo a `03_eda.ipynb` sin repetir la limpieza.

Los notebooks se ejecutan en orden desde `notebooks/` (usan rutas relativas con `../`):

```
01_auditoria.ipynb → 02_limpieza.ipynb → 03_eda.ipynb → 04_estadistica.ipynb
```

El dashboard se abre con Power BI Desktop desde `dashboard/vuelos_clima_dashboard.pbix`.

---

## Próximos pasos

- Script `src/descargar_datos.py` para no tener que descargar el BTS a mano.
- Sacar a `src/` las funciones de limpieza que ahora mismo viven dentro de los notebooks.
- Post-hoc de Dunn con Bonferroni para la aerolínea en el Kruskal-Wallis (no urgente: el tamaño de efecto ya dice que es pequeño).
- Un modelo de predicción de retrasos que combine clima, franja horaria y aerolínea, en vez de mirarlos por separado.

---

## Contribuciones

Este es un proyecto académico individual, pero está abierto a sugerencias. Si detectas algo que se pueda mejorar (una visualización, un enfoque de limpieza distinto, un hallazgo adicional), siéntete libre de abrir un *issue* o una *pull request*.

---

## Autora

**María González**, [github.com/mgonzalezbenm](https://github.com/mgonzalezbenm)

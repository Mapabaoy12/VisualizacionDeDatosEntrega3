# StreamView Analytics: dashboards de contenido

Solución de visualización en Power BI para **StreamView Analytics**, plataforma de streaming que necesita decidir **en qué contenido invertir** y **qué contenido promocionar**. El proyecto consolida películas y series en un solo modelo de datos y lo presenta en dos dashboards, uno por audiencia.

Asignatura: Visualización de Datos (ADY1104-001D), Duoc UC San Andrés de Concepción.
Equipo: Gustavo Alvial, Benjamín Neira y Matías Turra. Profesor: Felipe Cabrera Orella.

| Dashboard | Audiencia | Pregunta de negocio |
|---|---|---|
| Ejecutivo | Gerencia general y directorio | ¿En qué contenido conviene invertir? |
| Marketing | Marketing y Operaciones | ¿Qué contenidos se están desaprovechando y qué acciones realizar? |

---

## 1. Estructura del repositorio

```
.
├── data/          Fuentes de datos (CSV)
├── dashboard/     Archivo de Power BI (.pbix) y modelo exportado (.vpax)
├── src/           Consultas de Power Query (M) y medidas DAX documentadas
├── docs/          Informe ejecutivo (PDF) y presentación (PPTX)
├── images/        Capturas antes/después de cada dashboard
└── README.md
```

Si algún nombre de archivo cambia, actualiza esta sección.

---

## 2. Fuentes de datos

Bases corporativas enriquecidas con metadatos internacionales de **TMDB** (The Movie Database).

| Archivo | Contenido | Registros |
|---|---|---|
| `data/netflix_movies_detailed_up_to_2025.csv` | Películas: título, director, reparto, país, fecha de incorporación, año de estreno, géneros, idioma, popularidad, votos, rating, **budget** y **revenue** | 16.000 |
| `data/netflix_tv_shows_detailed_up_to_2025.csv` | Series: los mismos campos descriptivos, **sin datos financieros** | 15.594 tras eliminar duplicados |

**Campos principales**

| Campo | Descripción |
|---|---|
| `show_id` | Identificador único del título |
| `type` | `Movie` o `TV Show` |
| `release_year` | Año de estreno (2010–2025) |
| `genres` | Géneros separados por coma (por ejemplo, "Comedy, Drama") |
| `country` | País o países de producción |
| `popularity` | Índice de popularidad de TMDB |
| `vote_count` / `vote_average` | Cantidad de votos y nota promedio (0–10) |
| `budget` / `revenue` | Presupuesto e ingresos en USD (solo películas) |

**Calidad de los datos que hay que conocer**

- Solo el **22% de las películas (2010–2024)** tiene budget y revenue mayores a 0. Las series no tienen datos financieros.
- **2025 está incompleto**: el 66% de sus títulos entra en el primer trimestre y el 68% aún no tiene votos. Por eso se excluye del análisis.
- El campo `duration` de las series vale "1 Seasons" en todos los registros, por lo que no sirve para medir renovación.
- El 17% de los títulos tiene varios países en un mismo texto.

---

## 3. Dependencias

| Requisito | Detalle |
|---|---|
| Sistema operativo | Windows 10 u 11 (Power BI Desktop no tiene versión para macOS) |
| Software | [Power BI Desktop](https://powerbi.microsoft.com/desktop/), versión reciente (2025 o posterior), gratuita |
| Configuración regional | Español (Chile) recomendada, para que los decimales se muestren con coma |
| Visuales | Solo visuales nativos de Power BI; no requiere visuales personalizados |
| Opcional | [DAX Studio](https://daxstudio.org/) para abrir el archivo `.vpax` y revisar el modelo |

No se necesita conexión a internet ni cuentas de servicio: todas las fuentes son archivos CSV locales.

---

## 4. Ejecución

1. Clona o descarga el repositorio.
2. Abre `dashboard/DashboardsEntregable2.pbix` con Power BI Desktop.
3. **Actualiza las rutas de los CSV.** Las consultas apuntan a la carpeta local de quien construyó el archivo, así que debes redirigirlas:
   1. Cinta **Inicio → Transformar datos → Configuración de origen de datos**.
   2. Selecciona cada origen, haz clic en **Cambiar origen** y elige el CSV correspondiente dentro de `data/`.
   3. Cierra la ventana y haz clic en **Aplicar cambios**.
4. Haz clic en **Inicio → Actualizar** y espera a que termine la carga.
5. Verifica que los KPIs coincidan con la [tabla de verificación](#7-verificación).

---

## 5. Modelo de datos

### Tablas

| Tabla | Origen | Uso |
|---|---|---|
| `PeliculasNetflix` | CSV de películas | Tabla base de películas |
| `SeriesNetflix` | CSV de series | Tabla base de series |
| `CatalogoGlobal` | Unión de películas y series, sin duplicados por `show_id` (31.594 títulos) | Tabla principal de ambos dashboards |
| `GenerosCG` | `CatalogoGlobal` con géneros separados (un registro por título y género) | Filtro y análisis por género |
| `dimCG` | `show_id` y país | Análisis por país |
| `DimNivel` | Tabla manual con los 6 niveles de potencial y su orden | Ordenar los niveles en los gráficos |
| `Seleccion de Metrica` | Parámetro de campo (ROI / Popularidad) | Cambiar la métrica de un visual |

### Relaciones

- `GenerosCG[show_id]` → `CatalogoGlobal[show_id]`: varios a uno, filtro en ambas direcciones (necesario para filtrar por género).
- `dimCG[show_id]` → `CatalogoGlobal[show_id]`: varios a uno.
- `DimNivel[Nivel]` → `CatalogoGlobal[NivelPotencial]`: uno a varios.

### Transformaciones clave (Power Query)

- **Nota nula para títulos sin votos:** si `vote_count = 0`, `vote_average` queda nulo (y no 0), para no sesgar el rating ponderado.
- **Limpieza de géneros:** se separa por "," y por "&", se recortan espacios, se unifica "Science Fiction" con "Sci-Fi" y se eliminan duplicados título-género.

### Concepto central: nivel de potencial

Cada título se clasifica cruzando dos criterios:

- **Rating ponderado** (promedio bayesiano con m = 50 votos): corrige las notas basadas en pocos votos.
- **Popularidad comparada con el percentil 75 de su propio tipo**: 17,3 en películas y 62,2 en series. Así cada formato se compara consigo mismo.

| Nivel | Regla |
|---|---|
| Priorizar | Popularidad sobre P75 y rating ponderado ≥ 7,5 |
| Descubrir | Popularidad igual o bajo P75 y rating ≥ 7,5 |
| Revisar | Popularidad sobre P75 y rating < 7,5 |
| Seguimiento | Popularidad entre P50 y P75 y rating < 7,5 |
| Explorar | Popularidad igual o bajo P50 y rating < 7,5 |
| No evaluable | Sin votos o sin popularidad |

### Medidas principales (DAX)

| Medida | Definición |
|---|---|
| `ROIcg` | Suma de revenue / suma de budget, solo películas con ambos valores mayores a 0 |
| `Meta ROI` | 3,5 |
| `Brecha ROI %` | (ROI − meta) / meta |
| `Revenue base ROI` | Suma de revenue sobre la misma base del ROI |
| `Cobertura Datos Financieros` | Películas con budget y revenue / total de películas |
| `Priorizar` | Títulos Priorizar / títulos evaluables |
| `% Descubrir` | Títulos Descubrir / títulos evaluables |
| `Contenidos Evaluables Marketing` | Títulos con nivel distinto de "No evaluable" |
| `Color ROI` | Verde si cumple la meta, ámbar si está a menos de 10%, rojo si no |

El código completo de cada medida y columna calculada está en `src/`.

---

## 6. Funcionamiento de los dashboards

Ambas páginas tienen un **filtro de página 2010–2024** (2025 excluido) y la misma franja de segmentadores: **año de estreno**, **tipo** (película o serie) y **género**. Cada segmentador actualiza todos los KPIs, gráficos y tablas de su página.

### Dashboard Ejecutivo

| Elemento | Qué muestra |
|---|---|
| Tarjeta ROI películas | ROI actual, meta, brecha y base. Se pone roja cuando está bajo la meta |
| Tarjeta Revenue | Ingresos sobre la misma base del ROI |
| % catálogo a priorizar / a descubrir | Proporción de títulos evaluables en cada nivel |
| % Priorizar por género | Qué géneros generan más contenido de alto potencial, con línea de promedio del catálogo (solo géneros con 400 o más títulos) |
| ROI por nivel de potencial | ROI de cada nivel contra la línea de meta de 3,5x |
| Tendencia de Priorizar por año | Evolución del porcentaje de títulos Priorizar; 2024 punteado porque sus títulos aún acumulan votos |

> Al filtrar **Tipo = TV Show**, el ROI y el revenue quedan en blanco. Es el comportamiento esperado: las series no tienen datos financieros.

### Dashboard de Marketing

| Elemento | Qué muestra |
|---|---|
| KPIs | Contenidos evaluables, % a descubrir, % a priorizar y contenidos con oportunidad |
| Dispersión popularidad vs rating | Cada título según su percentil de popularidad y su rating, con líneas en P75 y 7,5 que forman los niveles |
| Top 5 países y top 5 géneros | % de títulos Descubrir (amarillo) y Priorizar (azul oscuro) sobre el catálogo propio de cada país o género |
| Tabla de títulos | Títulos Descubrir y Priorizar con al menos 1.000 votos |
| Tarjetas PRIORIZAR y DESCUBRIR | Acciones recomendadas y objetivo para cada grupo |

### Escenarios de uso

1. **¿Dónde licenciar la próxima ronda?** En el Ejecutivo, revisa la brecha del ROI, identifica los géneros con mayor tasa de Priorizar y confirma en "ROI por nivel" que ese perfil supera la meta.
2. **¿Qué títulos promocionar este mes?** En Marketing, filtra un género (por ejemplo, Animation), revisa los países líderes y elige títulos Descubrir de la tabla para una campaña segmentada.

---

## 7. Verificación

Con el filtro 2010–2024 y los segmentadores en "Todo", los KPIs deben mostrar:

| KPI | Valor esperado |
|---|---|
| ROI películas | 2,91x (−16,8% vs. meta 3,5x) |
| Revenue (base ROI) | $364 mil M |
| Cobertura financiera | 22% |
| % catálogo a priorizar | 3,8% |
| % catálogo a descubrir | 3,4% |
| Contenidos evaluables | 26.465 |
| Contenidos con oportunidad | 1.899 |

Si algún valor no coincide, revisa que las rutas de los CSV estén bien configuradas y que la actualización haya terminado sin errores.

---

## 8. Limitaciones

- El ROI solo representa al 22% de las películas, y las que reportan datos suelen ser grandes estrenos, por lo que puede sobrestimar el retorno del resto.
- Las relaciones entre género, nivel y ROI son asociaciones, no causalidad.
- Los umbrales (P75, rating 7,5 y meta 3,5) son criterios definidos por el equipo.
- Los datos describen el catálogo, no a los usuarios: la retención no se mide directamente.


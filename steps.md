# 📊 Proyecto NoSQL + ETL + Analytics

## Dataset: Pokémon

---

# ✅ FASE 1 — Diseño de la Base de Datos

## 📁 Colecciones requeridas

Se crearán **tres colecciones** dentro de MongoDB:

```
raw_pokemon
curated_pokemon
analytics_pokemon
```

---

## 🧱 Esquema esperado

### 🔴 RAW (datos casi originales)

Contiene los datos tras una limpieza mínima estructural.

```json
{
  "_id": ObjectId,
  "pokemon": "Bulbasaur",
  "type": ["Grass", "Poison"],
  "species": "Seed Pokémon",
  "height_m": 0.7,
  "weight_kg": 6.9,
  "abilities": ["Overgrow"],
  "catch_rate": 45,
  "base_exp": 64,
  "growth_rate": "Medium Slow",
  "egg_groups": ["Monster", "Grass"],
  "stats": {
      "hp": { "base": 45, "min": 231, "max": 294 },
      "attack": { "base": 49, "min": 92, "max": 216 },
      "defense": { "base": 49, "min": 92, "max": 216 },
      "speed": { "base": 45, "min": 85, "max": 207 }
  }
}
```

✔ Datos tipados
✔ Conversión de strings numéricos
✔ Estructuración lógica mínima

---

### 🟢 CURATED (datos analíticos limpios)

Pensada para análisis y consultas.

```json
{
  "pokemon": "Bulbasaur",
  "type_primary": "Grass",
  "type_secondary": "Poison",
  "height_m": 0.7,
  "weight_kg": 6.9,
  "bmi": 14.08,
  "total_base_stats": 318,
  "is_dual_type": true,
  "growth_rate": "Medium Slow"
}
```

✔ Columnas derivadas
✔ Sin ruido
✔ Lista para agregaciones

---

## 📦 ¿Datos embebidos o referenciados?

### ✔ Decisión: **Datos embebidos**

Ejemplo:

```
stats.hp.base
stats.attack.base
```

### ✔ Justificación

| Criterio                        | Decisión                    |
| ------------------------------- | --------------------------- |
| Dataset pequeño                 | No requiere normalización   |
| Consultas analíticas            | Más rápidas sin joins       |
| MongoDB favorece                | Lecturas atómicas           |
| No hay relaciones reutilizables | No se necesitan referencias |

➡ **Modelo documental optimizado para analítica.**

---

# ✅ FASE 2 — Carga de datos en RAW

## 1️⃣ Cargar dataset

```python
df = pd.read_csv("pokemon.csv")
```

## 2️⃣ ETL ligero obligatorio

### ✔ Normalización de nombres

```
"HP Base" → "hp_base"
"Catch Rate" → "catch_rate"
```

```python
df.columns = df.columns.str.lower().str.replace(" ", "_")
```

---

### ✔ Conversión de tipos

Campos originalmente `object`:

* height
* weight
* catch_rate
* base_exp

```python
df["height_m"] = df["height"].str.replace(" m","").astype(float)
df["weight_kg"] = df["weight"].str.replace(" kg","").astype(float)
df["catch_rate"] = df["catch_rate"].astype(int)
```

---

### ✔ Gestión mínima de nulos

```python
df = df.dropna(subset=["pokemon","type"])
```

---

### ✔ Eliminación de duplicados

```python
df = df.drop_duplicates(subset="pokemon")
```

---

## 3️⃣ Insertar en MongoDB

⚠ Debe poder re-ejecutarse:

```python
collection.delete_many({})
collection.insert_many(df.to_dict("records"))
```

✔ Uso obligatorio de `insert_many()`
✔ Limpieza previa de colección

---

# ✅ FASE 3 — CRUD sobre RAW

## 🟢 CREATE

```python
collection.insert_one({...})
```

---

## 🔵 READ

Filtro categórico:

```python
collection.find({"type": "Fire"})
```

Comparación numérica:

```python
collection.find({"hp_base": {"$gt": 100}})
```

Proyección:

```python
collection.find({}, {"pokemon":1,"hp_base":1})
```

Ordenación + límite:

```python
collection.find().sort("hp_base",-1).limit(10)
```

---

## 🟡 UPDATE

```python
collection.update_one({"pokemon":"Pikachu"},
                      {"$set":{"catch_rate":190}})
```

Normalización:

```python
collection.update_many({},
                       {"$set":{"generation":1}})
```

---

## 🔴 DELETE

```python
collection.delete_one({"pokemon":"MissingNo"})
collection.delete_many({"catch_rate":{"$lt":10}})
```

---

# ✅ FASE 4 — CURATED (Transformación)

Crear:

```python
df_cur = df.copy()
```

---

## ✔ Gestión de nulos (mínimo 2 columnas)

```python
df_cur["weight_kg"] = df_cur["weight_kg"].fillna(df_cur["weight_kg"].median())
df_cur["height_m"] = df_cur["height_m"].fillna(df_cur["height_m"].median())
```

---

## ✔ Corrección de tipos

```python
df_cur["base_exp"] = df_cur["base_exp"].astype(int)
```

---

## ✔ Columnas derivadas (2–3 obligatorias)

```python
df_cur["total_base_stats"] = df_cur[["hp_base","attack_base",
                                    "defense_base","speed_base"]].sum(axis=1)

df_cur["bmi"] = df_cur["weight_kg"] / (df_cur["height_m"]**2)

df_cur["is_dual_type"] = df_cur["type"].str.contains("/")
```

---

## ✔ Validaciones de calidad

* Sin duplicados
* Rangos válidos
* Sin nulos críticos

```python
assert df_cur["total_base_stats"].min() > 0
```

---

## Guardar en MongoDB

```python
curated.delete_many({})
curated.insert_many(df_cur.to_dict("records"))
```

---

# ✅ FASE 5 — ANALYTICS (Agregaciones)

## Pipeline 1 — KPIs por tipo

```python
[
 {"$group":{
   "_id":"$type_primary",
   "avg_stats":{"$avg":"$total_base_stats"},
   "count":{"$sum":1}
 }}
]
```

---

## Pipeline 2 — Top Pokémon

```python
[
 {"$match":{"total_base_stats":{"$gt":400}}},
 {"$sort":{"total_base_stats":-1}},
 {"$limit":10}
]
```

---

## Pipeline 3 — Campo calculado

```python
[
 {"$project":{
   "pokemon":1,
   "power_index":{"$multiply":["$attack_base","$speed_base"]}
 }},
 {"$group":{
   "_id":None,
   "avg_power":{"$avg":"$power_index"}
 }}
]
```

---

## Convertir resultados a DataFrame

```python
df_analytics = pd.DataFrame(list(collection.aggregate(pipeline)))
```

Guardar en:

```
analytics_pokemon
```

---

# ✅ FASE 6 — Rendimiento

## Índices creados

```python
curated.create_index("type_primary")
curated.create_index("total_base_stats")
```

---

## ✔ Justificación

| Índice           | Mejora                            |
| ---------------- | --------------------------------- |
| type_primary     | Agrupaciones por tipo más rápidas |
| total_base_stats | Top N optimizado                  |

➡ Reduce collection scan en pipelines.

---

# ✅ FASE 7 — Visualización

⚠ Datos deben venir de `analytics_pokemon`.

---

## 📊 Gráfico 1 — Barras

Comparación de fuerza media por tipo.

```python
df.plot(kind="bar", x="type_primary", y="avg_stats")
```

---

## 📈 Gráfico 2 — Líneas

Evolución de estadísticas medias ordenadas.

```python
df.sort_values("avg_stats").plot(kind="line")
```

---

## 📦 Gráfico 3 — Histograma

Distribución de `total_base_stats`.

```python
df_cur["total_base_stats"].plot(kind="hist")
```

---

## ✔ Todas las gráficas deben tener:

* Título
* Etiquetas de ejes
* Leyenda (si aplica)

---

## 🧠 Interpretación (obligatoria en notebook)

Debajo de cada gráfica explicar:

1️⃣ Qué se observa
2️⃣ Qué significa analíticamente
3️⃣ Qué decisión permitiría tomar

Ejemplo:

> Los Pokémon de tipo Dragón presentan mayor media de estadísticas, lo que sugiere balanceo desigual en diseño de juego.

---

# 🎯 RESULTADO FINAL

Se construye un flujo profesional tipo **Data Engineering Pipeline**:

```
CSV → RAW → CURATED → ANALYTICS → Visualización
```

Simula arquitectura real usada en:

* Data Lakes
* Machine Learning pipelines
* Business Intelligence
* Sistemas ETL modernos

---

# ✔ Checklist de entrega

* [ ] 3 colecciones creadas
* [ ] Esquema documentado
* [ ] ETL reproducible
* [ ] CRUD completo
* [ ] Transformaciones justificadas
* [ ] 3 pipelines aggregate()
* [ ] Índices creados
* [ ] 3 visualizaciones desde ANALYTICS
* [ ] Interpretación analítica escrita

---

👉 Esto cumple exactamente con un diseño NoSQL orientado a analítica real.

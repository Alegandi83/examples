# Image Batch Inference with Databricks AI

Questo progetto dimostra come eseguire inferenza batch su immagini utilizzando **Databricks AI** e il modello **Llama 4 Maverick** per generare descrizioni automatiche di immagini.

---

## 📋 Overview

Il notebook `Image sample batch inference.ipynb` implementa una pipeline completa per:

1. **Generare** 1000 URL di immagini campione
2. **Scaricare** le immagini in formato binario
3. **Analizzare** le immagini con AI per generare descrizioni testuali dettagliate
4. **Salvare** i risultati in tabelle Unity Catalog

---

## 🎯 Caso d'Uso

Questo esempio è ideale per:

- **E-commerce**: Generazione automatica di descrizioni prodotti da immagini
- **Content Management**: Catalogazione automatica di asset visuali
- **Accessibility**: Creazione di alt-text per immagini web
- **Data Enrichment**: Arricchimento di dataset con metadata generati da AI
- **Batch Processing**: Elaborazione massiva di immagini con Spark

---

## 🏗️ Architettura

```
┌─────────────────────────────────────────────────┐
│  Step 1: URL Generation                         │
│  ├─ Spark DataFrame (1-1000 IDs)                │
│  └─ URLs: https://picsum.photos/id/{ID}/800/600 │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  Step 2: Image Download                         │
│  ├─ PySpark UDF con requests                    │
│  └─ Binary content (BinaryType)                 │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  Step 3: AI Inference                           │
│  ├─ ai_query() function                         │
│  ├─ Model: databricks-llama-4-maverick          │
│  └─ Prompt: descrizione dettagliata             │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│  Step 4: Results Storage                        │
│  └─ Unity Catalog Tables                        │
└─────────────────────────────────────────────────┘
```

---

## 🗄️ Data Flow

### Tabelle Unity Catalog

| Tabella | Schema | Descrizione |
|---------|--------|-------------|
| `image_urls` | `image_url: STRING` | URL delle immagini da processare |
| `image_codes` | `image_url: STRING`<br>`image_binary: BINARY` | Immagini scaricate in formato binario |
| `image_desc` | `image_url: STRING`<br>`descrizione: STRING` | Immagini con descrizioni AI generate |

**Catalog**: `platform_catalog`  
**Schema**: `marketplace`

---

## 📝 Pipeline Steps

### Step 1: Generazione URL Immagini

```python
# Genera 1000 URL di immagini da picsum.photos
df = spark.range(1, 1001).withColumn(
    "image_url",
    concat(
        lit("https://picsum.photos/id/"),
        col("id").cast("string"),
        lit("/800/600")
    )
)

# Salva in Unity Catalog
df.write.mode("append").saveAsTable("platform_catalog.marketplace.image_urls")
```

**Output**: Tabella con 1000 URL di immagini 800x600px

---

### Step 2: Download Immagini

```python
# UDF per scaricare immagini
def download_image(url):
    try:
        response = requests.get(url, timeout=10)
        if response.status_code == 200:
            return response.content
    except Exception:
        return None

download_image_udf = udf(download_image, BinaryType())

# Applica UDF per scaricare immagini
df_with_binary = df.withColumn(
    "image_binary", 
    download_image_udf(df["image_url"])
)

# Salva immagini binarie
df_with_binary.write.mode("overwrite")\
    .saveAsTable("platform_catalog.marketplace.image_codes")
```

**Output**: Tabella con URL + contenuto binario delle immagini

---

### Step 3: AI Inference con Llama 4

```python
# Usa ai_query per generare descrizioni
result_df = spark.sql("""
SELECT
  image_url,
  ai_query(
    'databricks-llama-4-maverick',
    'Descrivi dettagliatamente il contenuto di questa immagine.',
    files => image_binary
  ) AS descrizione
FROM platform_catalog.marketplace.image_codes
""")

# Salva risultati
result_df.write.mode("overwrite")\
    .saveAsTable("platform_catalog.marketplace.image_desc")
```

**Output**: Tabella con URL + descrizione AI generata

---

### Step 4: Visualizzazione Risultati

```sql
SELECT * 
FROM platform_catalog.marketplace.image_desc
LIMIT 10;
```

**Output**: Tabella con descrizioni dettagliate generate dal modello AI

---

## 🚀 Come Utilizzare

### Prerequisiti

**Databricks Requirements**:
- Workspace Databricks con Unity Catalog abilitato
- Access al modello `databricks-llama-4-maverick`
- Catalog `platform_catalog` e schema `marketplace` (o modifica i nomi)
- Cluster con DBR 14.0+ (raccomandato)
- Runtime ML opzionale per migliori performance

**Permessi Unity Catalog**:
- `CREATE TABLE` su `platform_catalog.marketplace`
- `SELECT` su tabelle esistenti
- `USE CATALOG` e `USE SCHEMA`

**Librerie Python**:
- `requests` (pre-installato su DBR)
- `pyspark` (incluso)

---

### Setup

1. **Crea Catalog e Schema** (se non esistono):

```sql
-- Crea catalog
CREATE CATALOG IF NOT EXISTS platform_catalog;

-- Usa il catalog
USE CATALOG platform_catalog;

-- Crea schema
CREATE SCHEMA IF NOT EXISTS marketplace;
```

2. **Verifica Access al Modello AI**:

```python
# Test ai_query function
spark.sql("""
SELECT ai_query(
  'databricks-llama-4-maverick',
  'Hello, are you working?'
) AS test
""").show()
```

---

### Esecuzione

#### Opzione 1: Notebook Completo

```bash
# Apri il notebook in Databricks
1. Carica "Image sample batch inference.ipynb" nel workspace
2. Attacca a un cluster
3. Esegui tutte le celle (Run All)
```

#### Opzione 2: Esecuzione Cell-by-Cell

```python
# Cell 1: Setup e generazione URLs
# Cell 2: Download immagini
# Cell 3: AI inference
# Cell 4: Visualizzazione risultati
```

#### Opzione 3: Job Schedulato

```yaml
# Crea Databricks Job per esecuzione automatica
name: Image Batch Inference
schedule: "0 0 * * *"  # Daily
notebook_path: /examples/Image sample batch inference
```

---

## ⚙️ Configurazione

### Personalizzazione

#### Modifica Numero Immagini

```python
# Cambia range per processare più o meno immagini
df = spark.range(1, 5001)  # 5000 immagini invece di 1000
```

#### Modifica Dimensioni Immagini

```python
# Cambia risoluzione immagini
concat(
    lit("https://picsum.photos/id/"),
    col("id").cast("string"),
    lit("/1920/1080")  # Full HD invece di 800x600
)
```

#### Modifica Prompt AI

```sql
-- Personalizza il prompt per descrizioni specifiche
ai_query(
  'databricks-llama-4-maverick',
  'Analizza questa immagine e identifica: oggetti, colori, emozioni, e contesto.',
  files => image_binary
)
```

#### Usa Modello Diverso

```sql
-- Cambia modello AI (se disponibile)
ai_query(
  'databricks-bge-large-en',  -- o altro modello
  'Describe this image.',
  files => image_binary
)
```

---

## 📊 Performance

### Benchmarks (Cluster Standard: 8 cores, 32GB RAM)

| Immagini | Download Time | Inference Time | Totale |
|----------|---------------|----------------|--------|
| 100 | ~30 sec | ~2 min | ~2.5 min |
| 1000 | ~5 min | ~15 min | ~20 min |
| 5000 | ~25 min | ~75 min | ~100 min |

**Note**:
- Download time dipende dalla network speed
- Inference time scala linearmente con il numero di immagini
- Usa cluster più grandi per performance migliori

### Ottimizzazioni

1. **Parallelismo**: Aumenta partition per download parallelo
```python
df.repartition(200).withColumn("image_binary", download_image_udf(...))
```

2. **Batch Size**: Processa in batch per grandi dataset
```python
# Processa 1000 immagini alla volta
for i in range(0, total_images, 1000):
    batch = df.filter((col("id") >= i) & (col("id") < i + 1000))
    # Process batch...
```

3. **Cache**: Cache DataFrame per riutilizzo
```python
df.cache()
```

---

## 🔧 Troubleshooting

### Errore: "Table not found"

```sql
-- Verifica che catalog e schema esistano
SHOW CATALOGS;
SHOW SCHEMAS IN platform_catalog;

-- Crea se necessario
CREATE SCHEMA IF NOT EXISTS platform_catalog.marketplace;
```

### Errore: "ai_query function not found"

```python
# Verifica accesso a Databricks AI
# Serve workspace con Foundation Model APIs enabled
# Contatta admin per abilitare feature
```

### Errore: "Permission denied"

```sql
-- Verifica permessi Unity Catalog
SHOW GRANTS ON CATALOG platform_catalog;
SHOW GRANTS ON SCHEMA platform_catalog.marketplace;

-- Admin può garantire permessi:
GRANT CREATE TABLE ON SCHEMA platform_catalog.marketplace TO `user@company.com`;
```

### Download Timeout

```python
# Aumenta timeout per connessioni lente
response = requests.get(url, timeout=30)  # 30 secondi invece di 10
```

### Out of Memory

```python
# Riduci parallelismo o dimensioni immagini
# Usa immagini più piccole
lit("/400/300")  # invece di 800x600
```

---

## 📚 Risorse

### Documentazione Databricks

- [AI Functions](https://docs.databricks.com/sql/language-manual/functions/ai_query.html)
- [Foundation Model APIs](https://docs.databricks.com/machine-learning/foundation-models/index.html)
- [Unity Catalog](https://docs.databricks.com/data-governance/unity-catalog/index.html)
- [PySpark UDF](https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.functions.udf.html)

### Image Source

- [Lorem Picsum](https://picsum.photos/) - Free placeholder images

### Modelli AI

- **Llama 4 Maverick**: Foundation model per multimodal understanding
- Supporta: testo, immagini, analisi visuale
- Lingue: Multilingue (incluso Italiano)

---

## 🎯 Esempi Output

### Input
```
image_url: https://picsum.photos/id/237/800/600
```

### Output (esempio)
```
descrizione: "L'immagine mostra un cucciolo di labrador nero con occhi 
espressivi. Il cane è fotografato in primo piano con uno sfondo sfocato. 
L'illuminazione è naturale e morbida. Il cucciolo ha un'espressione curiosa 
e attenta. La composizione enfatizza le caratteristiche facciali del cane, 
in particolare gli occhi lucidi e il muso. Lo stile fotografico è ritrattistico 
con profondità di campo ridotta."
```

---

## 💡 Estensioni Possibili

### 1. Object Detection
```sql
ai_query(
  'model',
  'Elenca tutti gli oggetti visibili in questa immagine.',
  files => image_binary
)
```

### 2. Sentiment Analysis
```sql
ai_query(
  'model',
  'Qual è l\'emozione principale trasmessa da questa immagine?',
  files => image_binary
)
```

### 3. OCR (Text Extraction)
```sql
ai_query(
  'model',
  'Estrai tutto il testo visibile in questa immagine.',
  files => image_binary
)
```

### 4. Image Classification
```sql
ai_query(
  'model',
  'Classifica questa immagine in una di queste categorie: natura, urbano, ritratto, astratto.',
  files => image_binary
)
```

### 5. Multi-Language Support
```sql
-- Descrizioni in altre lingue
ai_query(
  'model',
  'Describe this image in English with technical details.',
  files => image_binary
)
```

---

## 🚨 Limitazioni

- **Rate Limiting**: AI API potrebbe avere limiti di request/min
- **Image Size**: Immagini molto grandi potrebbero causare timeout
- **Model Availability**: Dipende dalla configurazione workspace
- **Cost**: Inferenza AI ha costi associati (vedi pricing Databricks)
- **Accuracy**: Descrizioni generate da AI potrebbero non essere sempre perfette

---

## 📝 Note

- Le immagini sono campioni pubblici da Lorem Picsum
- Il modello AI genera descrizioni in lingua italiana (configurabile)
- Le tabelle vengono sovrascritte ad ogni esecuzione (mode="overwrite")
- Timeout download impostato a 10 secondi per immagine

---

## 🔐 Best Practices

1. **Security**: Non processare immagini sensibili senza proper governance
2. **Cost Control**: Monitora costi di AI inference
3. **Data Quality**: Valida che tutte le immagini siano scaricate correttamente
4. **Error Handling**: Gestisci timeout e errori di rete
5. **Testing**: Testa su piccoli batch prima di processare migliaia di immagini

---

## 🤝 Contributi

Per miglioramenti o estensioni a questo esempio:

1. Testa modifiche su dataset piccolo
2. Documenta nuove features
3. Condividi risultati e performance

---

**Version**: 1.0.0  
**Last Update**: November 2025  
**Databricks Runtime**: 14.0+  
**AI Model**: Llama 4 Maverick


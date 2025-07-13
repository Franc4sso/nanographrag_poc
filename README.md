# Nano-GraphRAG POC - Guida Completa

## 🎯 Panoramica del Progetto

**Nano-GraphRAG** è un'implementazione semplificata di GraphRAG (Graph-based Retrieval-Augmented Generation) che combina:
- **Estrazione di entità e relazioni** dal testo
- **Creazione di grafi di conoscenza** 
- **Query intelligenti** basate su grafi
- **Integrazione con database di grafi** (Memgraph)

## 🚀 Setup Iniziale

### 1. Prerequisiti
```bash
# Verifica che Docker sia installato
docker --version

# Verifica Python 3.9+
python3 --version
```

### 2. Clonazione e Setup
```bash
# Clona il repository
git clone <repository-url>
cd nanographrag_poc

# Crea ambiente virtuale
python3 -m venv .venv
source .venv/bin/activate

# Installa dipendenze
pip install -r requirements.txt
pip install -e .
```

### 3. Dipendenze Aggiuntive
```bash
# Per i notebook
pip install notebook

# Per PyTorch (se necessario)
pip install torch

# Per transformers
pip install transformers
```

## 🐳 Setup Docker e Memgraph

### 1. Avvia Memgraph
```bash
# Avvia il container Memgraph
docker compose up -d

# Verifica che sia attivo
docker ps
```

### 2. Verifica Connessione
- **Memgraph Lab**: http://localhost:3000
- **Porta Bolt**: localhost:7687

## 📝 Preparazione Dati

### 1. Crea il File di Testo
Crea `data/alleanze.txt` con il tuo contenuto:

```txt
ALLEANZE DELLA SECONDA GUERRA MONDIALE

L'ASSE (Potenze dell'Asse)
La Germania nazista, guidata da Adolf Hitler, e l'Italia fascista, sotto Benito Mussolini, formarono l'Asse Roma-Berlino nel 1936...

[continua con il tuo testo]
```

### 2. Struttura Directory
```
nanographrag_poc/
├── data/
│   └── alleanze.txt          # Il tuo file di testo
├── notebooks/
│   └── my_poc_memgraph.ipynb # Notebook principale
├── nano_cache/               # Cache generata automaticamente
└── .venv/                    # Ambiente virtuale
```

## 🔑 Configurazione API OpenAI

### 1. Ottieni Chiave API
- Vai su [OpenAI Platform](https://platform.openai.com/account/api-keys)
- Crea una nuova chiave API
- Copia la chiave (inizia con `sk-`)

### 2. Imposta la Chiave nel Notebook
Nel notebook `my_poc_memgraph.ipynb`, modifica:

```python
# Imposta la tua chiave API OpenAI
os.environ["OPENAI_API_KEY"] = "sk-tua-chiave-reale-qui"
```

## 📊 Esecuzione del Flusso Completo

### 1. Avvia Jupyter (Opzionale)
```bash
# Se vuoi usare Jupyter web
jupyter notebook --no-browser --port=8888 --ip=0.0.0.0
# Poi apri: http://localhost:8888
```

### 2. Esegui il Notebook
Apri `notebooks/my_poc_memgraph.ipynb` nel tuo IDE e esegui le celle in ordine:

#### Cella 1: Setup e Inserimento Dati
```python
import os
import asyncio
from nano_graphrag import GraphRAG
import networkx as nx

# ✅ STEP 1 – Carica il testo da un file
with open("../data/alleanze.txt", "r", encoding="utf-8") as f:
    text = f.read()

# ✅ STEP 2 – Imposta la tua chiave API OpenAI
os.environ["OPENAI_API_KEY"] = "sk-tua-chiave-reale-qui"

# ✅ STEP 3 – Inizializza GraphRAG
graphrag = GraphRAG(
    working_dir="./nano_cache",
    enable_local=True,
    tokenizer_type="tiktoken",
    tiktoken_model_name="gpt-3.5"
)

# ✅ STEP 4 – Inserisci il documento
await graphrag.ainsert(text)

# ✅ STEP 5 – Carica il grafo salvato da file
G = nx.read_graphml("./nano_cache/graph_chunk_entity_relation.graphml")

# ✅ STEP 6 – Stampa le triplette soggetto --[relazione]--> oggetto
print("\n📌 Triplette estratte:")
for u, v, data in G.edges(data=True):
    relation = data.get("description", "(nessuna relazione)")
    print(f"{u} --[{relation}]--> {v}")
```

#### Cella 2: Inserimento in Memgraph
```python
from neo4j import GraphDatabase
import networkx as nx

# ✅ Carica grafo da GraphML generato da GraphRAG
G = nx.read_graphml("./nano_cache/graph_chunk_entity_relation.graphml")

# ✅ Estrai le triplette (soggetto, relazione, oggetto)
triples = []
for u, v, data in G.edges(data=True):
    subject = u.replace('"', '')
    obj = v.replace('"', '')
    relation = data.get("description", "relazione_non_specificata").replace('"', '')
    triples.append((subject, relation, obj))

# ✅ Connessione a Memgraph
driver = GraphDatabase.driver("bolt://localhost:7687", auth=("", ""))

# ✅ Inserimento delle triplette in Memgraph
with driver.session() as session:
    for s, r, o in triples:
        session.run("""
            MERGE (a:Entity {name: $s})
            MERGE (b:Entity {name: $o})
            MERGE (a)-[:RELATION {description: $r}]->(b)
        """, s=s, o=o, r=r)

driver.close()
print("✅ Triplette inserite in Memgraph.")
```

#### Cella 3: Query con OpenAI
```python
from neo4j import GraphDatabase
from openai import OpenAI
import os
from dotenv import load_dotenv

load_dotenv()

# ✅ Nuova API OpenAI
client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

# ✅ Connessione a Memgraph
driver = GraphDatabase.driver("bolt://localhost:7687", auth=("", ""))

# ✅ Query per ottenere i dati
with driver.session() as session:
    result = session.run("""
        MATCH (a)-[r]->(b) 
        RETURN a.name, r.description, b.name
    """)
    
    context = "\n".join([f"{record['a.name']} {record['r.description']} {record['b.name']}" 
                        for record in result])

driver.close()

# ✅ Query con nuova API OpenAI
response = client.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "system", "content": "Rispondi solo in base al contesto fornito."},
        {"role": "user", "content": f"Contesto:\n{context}\n\nDomanda: Quali alleanze ha stretto la Francia?"}
    ]
)

print("Risposta LLM:")
print(response.choices[0].message.content)
```

## 🔍 Visualizzazione del Grafo

### 1. Memgraph Lab (Interfaccia Web)
- **URL**: http://localhost:3000
- **Funzionalità**: Visualizzazione interattiva del grafo

### 2. Come Visualizzare i Grafi

1. **Apri Memgraph Lab**: Vai su http://localhost:3000
2. **Vai alla scheda "Query"**: Clicca su "Query" nella barra laterale
3. **Incolla una delle query qui sotto** nella barra di testo
4. **Clicca "Run"** o premi **Ctrl+Enter**
5. **Vedrai i risultati** sia in formato tabella che grafico

### 3. Query Utili in Memgraph Lab

#### Query Base (Sempre Funzionanti)

**Conta i nodi**
```cypher
MATCH (n) RETURN count(n) as numero_nodi
```

**Vedi tutte le entità**
```cypher
MATCH (n:Entity) RETURN n.name
```

**Vedi tutte le relazioni**
```cypher
MATCH ()-[r:RELATION]->() RETURN r.description
```

**Vedi il grafo completo**
```cypher
MATCH (a)-[r]->(b) RETURN a, r, b
```

#### Query di Ricerca

**Cerca entità contenenti "Germania"**
```cypher
MATCH (n:Entity) 
WHERE n.name CONTAINS "Germania" 
RETURN n.name
```

**Vedi entità e relazioni specifiche**
```cypher
MATCH (a:Entity)-[r:RELATION]->(b:Entity) 
RETURN a.name, r.description, b.name
```

**Cerca entità con relazioni specifiche**
```cypher
MATCH (a:Entity)-[r:RELATION]->(b:Entity) 
WHERE a.name CONTAINS "Germania" OR b.name CONTAINS "Germania"
RETURN a.name, r.description, b.name
```

### 4. Query da Python
```python
from neo4j import GraphDatabase

driver = GraphDatabase.driver("bolt://localhost:7687", auth=("", ""))

# Conta nodi
with driver.session() as session:
    result = session.run("MATCH (n) RETURN count(n) as numero_nodi")
    count = result.single()['numero_nodi']
    print(f"Numero totale di nodi: {count}")

# Vedi relazioni
with driver.session() as session:
    result = session.run("MATCH (a)-[r]->(b) RETURN a.name, r.description, b.name LIMIT 10")
    for record in result:
        print(f"{record['a.name']} --[{record['r.description']}]--> {record['b.name']}")

driver.close()
```



## 🔍 Esempi di Query Avanzate

### Query per Analisi Specifiche

**Trova tutti i leader**
```cypher
MATCH (a:Entity)-[r:RELATION]->(b:Entity) 
WHERE r.description CONTAINS "guidato" OR r.description CONTAINS "leader"
RETURN a.name, r.description, b.name
```

**Trova alleanze specifiche**
```cypher
MATCH (a:Entity)-[r:RELATION]->(b:Entity) 
WHERE r.description CONTAINS "alleato" OR r.description CONTAINS "alleanza"
RETURN a.name, r.description, b.name
```

**Trova eventi per anno**
```cypher
MATCH (a:Entity)-[r:RELATION]->(b:Entity) 
WHERE b.name CONTAINS "1936" OR b.name CONTAINS "1940" OR b.name CONTAINS "1941"
RETURN a.name, r.description, b.name
```

**Conta le relazioni**
```cypher
MATCH ()-[r:RELATION]->() 
RETURN count(r) as numero_relazioni
```

## 🎯 Flusso Completo Riepilogo

1. **Setup**: Docker, Python, dipendenze
2. **Dati**: Crea file `data/alleanze.txt`
3. **API**: Imposta chiave OpenAI
4. **GraphRAG**: Estrai entità e relazioni
5. **Memgraph**: Inserisci triplette nel database
6. **Visualizza**: Usa Memgraph Lab per esplorare
7. **Query**: Interroga con Cypher o Python
8. **LLM**: Combina con OpenAI per risposte intelligenti

## 📚 Risorse Aggiuntive

- **Documentazione Memgraph**: https://memgraph.com/docs
- **Cypher Query Language**: https://neo4j.com/developer/cypher/
- **OpenAI API**: https://platform.openai.com/docs
- **Nano-GraphRAG**: https://github.com/gusye1234/nano-graphrag

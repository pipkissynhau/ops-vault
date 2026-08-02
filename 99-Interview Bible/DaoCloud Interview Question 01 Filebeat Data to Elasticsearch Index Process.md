# DaoCloud Interview Question 01: How does Filebeat-collected data land in Elasticsearch indices  
## Question Description  
Interview question: After Filebeat collects data, such as an index/data stream, how does Elasticsearch perform indexing?  

---  

## Core Conclusion  
This question fundamentally examines the log ingestion pipeline, not just whether Filebeat can collect logs.  

If the current pipeline uses Elasticsearch's data stream mode:  

- Filebeat is responsible for sending events to Elasticsearch  
- Elasticsearch identifies or creates a data stream based on matching index templates  
- The actual data storage occurs in the backing indices behind the data stream  
- Subsequently, with rollover, new backing indices are created and the write index is switched  

Elastic official explanation:  
- Filebeat's `output.elasticsearch` sends events directly to Elasticsearch via the Elasticsearch HTTP API :contentReference[oaicite:0]{index=0}  
- Data streams require matching index templates, and data streams consist of one or more hidden backing indices :contentReference[oaicite:1]{index=1}  
- Rollover replaces the current write index with a new one; for data streams, the new backing index becomes the new write index :contentReference[oaicite:2]{index=2}  

---  

## Correct Understanding  
Do not simply answer that "Filebeat writes directly to a regular index".  

More accurate pipeline:  

1. Filebeat collects logs  
2. Filebeat sends events to Elasticsearch via `output.elasticsearch`  
3. Elasticsearch matches index templates based on the target name  
4. If the template defines a data stream, it creates or identifies the corresponding data stream  
5. Write requests are automatically routed to the current backing index  
6. Backing indices are the actual physical indices storing documents  
7. When lifecycle conditions are met, rollover generates new backing indices  

If not using data stream mode, it may also write directly to a regular index or alias; Filebeat documentation also states that `index` targets can point to indexes, aliases, or data streams. :contentReference[oaicite:3]{index=3}  

---  

## How to Answer in an Interview  

### One-sentence Version  
After collecting logs, Filebeat sends events to Elasticsearch via its output; if the target is a data stream, Elasticsearch creates or identifies the data stream based on matching index templates, then writes documents to the current backing index, and switches to a new write index through rollover. :contentReference[oaicite:4]{index=4}  

### More Detailed Version  
Filebeat is responsible for collecting and sending logs, not for managing the lifecycle of underlying backing indices.  
The actual data persistence logic is handled by Elasticsearch: when receiving write requests, Elasticsearch first matches index templates based on the name; if the template defines a data stream, it creates or identifies the corresponding data stream and writes documents to the current backing index.  
Thus, it appears as if "writing to a data stream," but physically, the data is stored in backing indices.  
After data growth, Elasticsearch creates new backing indices through rollover, and the new backing index becomes the new write index. :contentReference[oaicite:5]{index=5}  

---  

## Key Knowledge Points  

### 1. Filebeat handles sending, not underlying index lifecycle  
Filebeat sends events to Elasticsearch via `output.elasticsearch`.  
It can set the target to an index, alias, or data stream, but Elasticsearch primarily handles creating indices, applying templates, and rollover. :contentReference[oaicite:6]{index=6}  

### 2. Data streams are not regular single indices  
Data streams are an abstraction layer for time-series data like logs and metrics.  
They are composed of one or more hidden backing indices. :contentReference[oaicite:7]{index=7}  

### 3. Index templates determine data stream and backing index configurations  
When a document is first written and triggers index creation, the matched template determines:  
- Mappings  
- Settings  
- Lifecycle/ILM configurations  
- Whether it is used as a data stream  

Templates are Elasticsearch's core mechanism for applying configurations when creating indices or data streams. :contentReference[oaicite:8]{index=8}  

### 4. Backing indices are where data is actually stored  
Data streams are logical entry points.  
Backing indices are the actual physical storage indices. :contentReference[oaicite:9]{index=9}  

### 5. Rollover switches to a new write index  
When size, age, or document count conditions are met, rollover creates a new write index.  
For data streams, the newly generated backing index becomes the new write index. :contentReference[oaicite:10]{index=10}  

---  

## Common Mistakes  

### Common Mistake 1: Confusing data streams with regular indices  
Inaccurate.  
Data streams are abstractions for time-series data, corresponding to one or more backing indices. :contentReference[oaicite:11]{index=11}

### Common Mistake 2: Saying Filebeat itself decides the backing index name  
Inaccurate.  
Filebeat is responsible for sending events, but the creation and switching of data stream / backing index is primarily determined by Elasticsearch's template and rollover mechanism. :contentReference[oaicite:12]{index=12}

### Common Mistake 3: Ignoring index template  
If the answer completely omits template, it indicates incomplete understanding of the ES indexing pipeline.  
Elastic officially states that templates are the mechanism for applying settings, mappings, etc., when creating an index or data stream. :contentReference[oaicite:13]{index=13}

### Common Mistake 4: Only saying "writing to index" but unable to distinguish logical entry point and physical disk storage  
More accurate expression:  
- Logical write object: data stream  
- Physical storage object: backing index  

---

## Interview Verbal Template  
I understand this pipeline as follows:  
Filebeat collects logs and sends events directly to Elasticsearch via `output.elasticsearch`.  
If this pipeline uses data stream, ES will first match the index template by target name; if the template defines a data stream, it will create or identify this data stream.  
Subsequently, documents are not arbitrarily written to a regular index, but instead written by ES to the current backing index of this data stream.  
The backing index is the underlying index that actually stores data. Later, as data volume grows or lifecycle conditions are met, a new backing index will be generated via rollover and become the new write index. :contentReference[oaicite:14]{index=14}

---

## If the interviewer asks further  
If the interviewer asks "What else can be done during the process," you can add:  

- Filebeat can combine with ingest pipeline to perform parsing and field processing when sending to ES  
- Preprocessing like field renaming, structure parsing, and tagging can be done before data entry  

These processes still occur within the Elasticsearch write pipeline, not completed entirely by Filebeat. The relevant pipeline entry point remains Elasticsearch output. :contentReference[oaicite:15]{index=15}

---

## Memory Template  
First remember this main line:  

Filebeat sends events  
→ Elasticsearch output  
→ Match index template  
→ data stream  
→ backing index  
→ rollover  

---

## Tags  
#ElasticSearch  
#Filebeat  
#DataStream  
#BackingIndex  
#IndexTemplate  
#Rollover  
#LogPlatform  
#Observation  
#TransportInterview  
#TheYunwonInterview.
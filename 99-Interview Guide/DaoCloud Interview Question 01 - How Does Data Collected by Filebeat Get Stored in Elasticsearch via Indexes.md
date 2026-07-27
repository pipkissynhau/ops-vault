# DaoCloud Interview Question 01: How Does Data Collected by Filebeat Get Stored in Elasticsearch via Indexes?
## Question Description
Interview question: After Filebeat collects data, for example, a certain index/data stream is collected, how does Elasticsearch create indexes?

---

## Key Points
This question essentially examines the log writing process, rather than simply whether Filebeat can collect logs.

If the current process uses Elasticsearch's data stream mode, then:

- Filebeat is responsible for sending events to Elasticsearch.
- Elasticsearch identifies or creates a data stream based on the matching index template.
- The actual storage of data occurs in the backing indices behind the data stream.
- Subsequently, when rollover occurs, new backing indices are created, and the writing index is switched.

Elasticsearch official documentation states:
- Filebeat's `output.elasticsearch` sends events directly to Elasticsearch using the HTTP API:contentReference[oaicite:0]{index=0}.
- A data stream requires a matching index template and consists of one or more hidden backing indices :contentReference[oaicite:1]{index=1}.
- Rollover replaces the current writing index with a new one; for data streams, the new backing index becomes the new writing index :contentReference[oaicite:2]{index=2].

---

## Correct Understanding
Do not simply say that "Filebeat writes data directly to an ordinary index."

The more accurate process is:

1. Filebeat collects logs.
2. Filebeat sends events to Elasticsearch through `output.elasticsearch`.
3. Elasticsearch matches the target name against the index template.
4. If the template defines a data stream, it creates or identifies the corresponding data stream.
5. The write request is automatically routed to the current backing index.
6. The backing index is the actual physical storage for the documents.
7. When the lifecycle conditions are met, rollover occurs, generating a new backing index.

If it's not in data stream mode, the data may also be written directly to an ordinary index or alias; Filebeat documentation also states that the `index` target can point to an index, alias, or data stream :contentReference[oaicite:3]{index=3%.

---

## How to Answer in an Interview

### One-Sentence Version
After Filebeat collects logs, it sends them to Elasticsearch via the `output.elasticsearch` setting. If the target is a data stream, Elasticsearch creates or identifies the corresponding data stream and automatically writes the documents to the current backing index. Subsequently, rollover generates new backing indices :contentReference[oaicite:4]{index=4].

### More Detailed Version
Filebeat itself is responsible for collecting and sending logs but does not manage the lifecycle of the underlying backing indices.  
The actual storage logic occurs in Elasticsearch: upon receiving a write request, ES first matches the target name against the index template. If the template defines a data stream, it creates or identifies the corresponding data stream and writes the documents to the current backing index.  
Therefore, although it appears that data is "written to a data stream," it is actually the backing indices that store the physical documents.  
As data volume increases, ES performs rollover to generate new backing indices, which then become the new writing indexes :contentReference[oaicite:5]{index=5].

---

## Key Knowledge Points

### 1. Filebeat Sends Data but Does Not Manage Index Lifecycle
Filebeat sends events to Elasticsearch using `output.elasticsearch`. It can set the target as an index, alias, or data stream, but Elasticsearch handles the actual creation of indexes, application of templates, and rollover :contentReference[oaicite:6]{index=6].

### 2. Data Streams Are Not Ordinary Indexes
Data streams are an abstraction layer for time-series data such as logs and metrics. They consist of one or more hidden backing indices :contentReference[oaicite:7]{index=7>.

### 3. Index Templates Determine Data Stream and Backing Index Configurations
When a document is first written, the matching template determines:
- Mappings
- Settings
- Lifecycle/ILM configurations
- Whether it should be used as a data stream

Templates are key to applying settings and mappings when creating indexes or data streams :contentReference[oaicite:8]{index=8].

### 4. Backing Indices Are Where Data Is Actually Stored
Data streams serve as the logical entry point, while backing indices are the physical storage locations :contentReference[oaicite:9]{index=9].

### 5. Rollover Switches New Writing Indexes
When conditions such as size, age, or document count are met, rollover creates new writing indexes. For data streams, the new backing index becomes the new writing index :contentReference[oaicite:10]{index=10}.

---

## Common Mistakes

### Mistake 1: Referring to Data Streams as Ordinary Indexes
This is incorrect.
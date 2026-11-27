---
permalink: /projects/social-distro/
title: "Social-Distro (DataHub): an async RAG indexing and query backend in Spring Boot"
description: "A Spring Boot service that indexes per-user text collections through a two-stage Kafka fan-out into pgvector and answers questions with RAG."
type: project
order: 2
stack: "Java 25, Spring Boot, Kafka, PostgreSQL with pgvector, Hugging Face TEI, Docker Compose"
tags: [java, spring-boot, kafka, pgvector, rag, docker]
repo: https://github.com/Raghavendrapara/social-distro
abstract: >
  I wanted a backend that turns a user's collection of notes into something you can ask questions of, without letting
  one large collection block the others. Indexing runs as a background job that fans out over two Kafka topics into
  per-item embedding workers writing to pgvector; queries do a scoped nearest-neighbour search and pass the hits to an
  LLM, falling back to plain text if the vector path fails. The project shows async job design, idempotent consumers,
  vector storage in Postgres, and a full Docker Compose stack with metrics and tracing.
---

## The problem

The repository is `social-distro`; inside the code the application is called DataHub (`com.raghav.datahub`). The unit of
data is a **pod**: a named collection of text items owned by a user. A user creates a pod, adds items to it, starts an
indexing job, and then asks questions that should be answered only from that pod's content.

Two things made this more than a CRUD app. Turning text into embeddings is slow (each item is a network call to a model
server), so indexing has to run in the background and report progress. And one user with a large pod should not hold up
everyone else's indexing. I built this to work through those problems in Java, and moved it through three versions:
an in-memory prototype, an in-process virtual-thread worker pool, and finally the Kafka-based pipeline described here.

## Architecture

<div class="diagram">
<svg viewBox="0 0 740 370" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Architecture: the React UI calls the Spring Boot API. The API stores pods and jobs in Postgres and publishes a job event to the pod-indexing-jobs Kafka topic. A job worker claims the job and fans out one event per item to the item-indexing-events topic. Item workers call the TEI embedding server, which is cached in Redis, and upsert vectors into pgvector. Failed items go to a dead-letter topic. Queries embed the question, run a nearest-neighbour search in pgvector, and send the context to an LLM.">
  <defs>
    <marker id="social-distro-a1" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="dg-arrowhead"/></marker>
    <marker id="social-distro-a2" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="dg-arrowhead-accent"/></marker>
  </defs>
  <rect x="20" y="30" width="140" height="56" rx="8" class="dg-box"/>
  <text x="90" y="54" text-anchor="middle" class="dg-text">React UI</text>
  <text x="90" y="72" text-anchor="middle" class="dg-sub">Vite, TanStack Query</text>
  <rect x="200" y="30" width="150" height="56" rx="8" class="dg-box-accent"/>
  <text x="275" y="54" text-anchor="middle" class="dg-text">REST API</text>
  <text x="275" y="72" text-anchor="middle" class="dg-sub">Spring Boot 4, Java 25</text>
  <rect x="390" y="30" width="150" height="56" rx="8" class="dg-box"/>
  <text x="465" y="54" text-anchor="middle" class="dg-text">pod-indexing-jobs</text>
  <text x="465" y="72" text-anchor="middle" class="dg-sub">Kafka, key = podId</text>
  <rect x="580" y="30" width="140" height="56" rx="8" class="dg-box"/>
  <text x="650" y="54" text-anchor="middle" class="dg-text">Job worker</text>
  <text x="650" y="72" text-anchor="middle" class="dg-sub">claim, stream items</text>
  <rect x="20" y="150" width="140" height="56" rx="8" class="dg-box"/>
  <text x="90" y="174" text-anchor="middle" class="dg-text">Postgres</text>
  <text x="90" y="192" text-anchor="middle" class="dg-sub">pods, items, jobs</text>
  <rect x="200" y="150" width="150" height="56" rx="8" class="dg-box"/>
  <text x="275" y="174" text-anchor="middle" class="dg-text">pgvector</text>
  <text x="275" y="192" text-anchor="middle" class="dg-sub">vector(384), HNSW</text>
  <rect x="390" y="150" width="150" height="56" rx="8" class="dg-box"/>
  <text x="465" y="174" text-anchor="middle" class="dg-text">Item worker</text>
  <text x="465" y="192" text-anchor="middle" class="dg-sub">embed, upsert chunk</text>
  <rect x="580" y="150" width="140" height="56" rx="8" class="dg-box"/>
  <text x="650" y="174" text-anchor="middle" class="dg-text">item-indexing-events</text>
  <text x="650" y="192" text-anchor="middle" class="dg-sub">Kafka, key = itemId</text>
  <rect x="20" y="270" width="140" height="56" rx="8" class="dg-box"/>
  <text x="90" y="294" text-anchor="middle" class="dg-text">LLM</text>
  <text x="90" y="312" text-anchor="middle" class="dg-sub">Groq API or Ollama</text>
  <rect x="200" y="270" width="150" height="56" rx="8" class="dg-box"/>
  <text x="275" y="294" text-anchor="middle" class="dg-text">Redis</text>
  <text x="275" y="312" text-anchor="middle" class="dg-sub">embedding cache</text>
  <rect x="390" y="270" width="150" height="56" rx="8" class="dg-box"/>
  <text x="465" y="294" text-anchor="middle" class="dg-text">TEI server</text>
  <text x="465" y="312" text-anchor="middle" class="dg-sub">bge-small-en-v1.5</text>
  <rect x="580" y="270" width="140" height="56" rx="8" class="dg-box"/>
  <text x="650" y="294" text-anchor="middle" class="dg-text">Dead-letter topic</text>
  <text x="650" y="312" text-anchor="middle" class="dg-sub">failed items</text>
  <line x1="160" y1="58" x2="198" y2="58" class="dg-line" marker-end="url(#social-distro-a1)"/>
  <line x1="350" y1="58" x2="388" y2="58" class="dg-line-accent" marker-end="url(#social-distro-a2)"/>
  <line x1="540" y1="58" x2="578" y2="58" class="dg-line-accent" marker-end="url(#social-distro-a2)"/>
  <line x1="650" y1="86" x2="650" y2="148" class="dg-line-accent" marker-end="url(#social-distro-a2)"/>
  <line x1="580" y1="178" x2="542" y2="178" class="dg-line-accent" marker-end="url(#social-distro-a2)"/>
  <line x1="390" y1="178" x2="352" y2="178" class="dg-line-accent" marker-end="url(#social-distro-a2)"/>
  <line x1="465" y1="206" x2="465" y2="268" class="dg-line" marker-end="url(#social-distro-a1)"/>
  <line x1="390" y1="298" x2="352" y2="298" class="dg-line-dashed" marker-end="url(#social-distro-a1)"/>
  <line x1="650" y1="206" x2="650" y2="268" class="dg-line-dashed" marker-end="url(#social-distro-a1)"/>
  <line x1="205" y1="86" x2="125" y2="148" class="dg-line" marker-end="url(#social-distro-a1)"/>
  <line x1="275" y1="86" x2="275" y2="148" class="dg-line" marker-end="url(#social-distro-a1)"/>
  <line x1="222" y1="86" x2="142" y2="268" class="dg-line" marker-end="url(#social-distro-a1)"/>
  <text x="282" y="124" class="dg-sub">query: kNN</text>
  <text x="20" y="356" class="dg-label">accent: indexing path · dashed: cache and failure paths</text>
</svg>
<p class="diagram-caption">Figure 1. Indexing fans out over two Kafka topics into pgvector; queries embed the question, search pgvector within the pod, and send the hits to the LLM.</p>
</div>

**Write path.** `POST /pods` and `POST /pods/{id}/data` store pods and text items in Postgres through JPA. The schema is
managed by Liquibase (`db/changelog`), and Hibernate runs with `ddl-auto: validate` so the entities have to match it.

**Indexing path.** `POST /indexing/pods/{podId}` saves an `IndexingJob` as `PENDING`, publishes a `PodIndexingEvent` to
`pod-indexing-jobs` keyed by pod id, and returns `202 Accepted` with a job id. The job worker (3 listener threads)
claims the job with a conditional `UPDATE`, streams the pod's items out of Postgres with a JDBC fetch size of 500, and
publishes one `ItemIndexingEvent` per item to `item-indexing-events`. The item workers (5 listener threads, 5 partitions)
call a Hugging Face Text Embeddings Inference (TEI) container running `BAAI/bge-small-en-v1.5`, and upsert a row into
`vector_chunks`, a `vector(384)` column with an HNSW index. A message that fails is sent to a dead-letter topic.
The client polls `GET /indexing/jobs/{jobId}`.

**Query path.** `POST /pods/{podId}/query` embeds the question, runs `ORDER BY embedding <-> :q LIMIT 5` filtered by
`pod_id`, builds a context-plus-question prompt and sends it to the LLM client. In the current config that is
Llama 3.1 8B through Groq's OpenAI-compatible API; earlier versions ran `phi3:mini` locally in Ollama.

**Frontend.** A separate repo, `social-distro-ui`, is a small React 19 + Vite + TypeScript client using TanStack Query
and Zustand, with pages for pods, indexing (it polls job status) and querying.

## Key design decisions

**1. Kafka fan-out instead of an in-process virtual-thread pool.** The December 2025 version ran indexing inside the
API process: `Executors.newVirtualThreadPerTaskExecutor()`, items chunked into `CompletableFuture`s with `orTimeout` and
a retry counter. A test with 5,000 mocked items and 100 ms of simulated latency each finished under its 4-second
assertion, so virtual threads handled the I/O wait well. The problem was durability and isolation: a restart lost
in-flight jobs, and the work could only scale with the API pod. Moving to two Kafka topics made each item a
durable, retryable unit, and partitioning by pod id and item id spreads work across consumers. The cost is more moving
parts (ZooKeeper, a broker, serializer config) and a job status that is harder to define (see "What I'd do next").

**2. Two-level fan-out (job, then items).** The first topic carries one small event per job; the second carries one per
item. A single huge pod becomes thousands of independent item messages instead of one long-running message that pins
a consumer, which was the bottleneck I wrote down in the implementation plan ("one large pod blocks the entire worker
queue"). The trade-off is ordering: items of one pod are spread across partitions, so there is no "all items done"
signal without extra bookkeeping.

**3. Atomic job claim plus deterministic chunk ids, instead of locks.** Kafka gives at-least-once delivery, so the same
message can arrive twice. The job worker only proceeds if `UPDATE ... SET status = RUNNING WHERE status = PENDING`
changes a row, so a redelivered job event is skipped. The item worker writes chunks with id `podId:itemId`, so a
redelivered item overwrites the same row rather than creating a duplicate. I also added `@Version` columns for optimistic
locking on pods and jobs. The alternative, a distributed lock or exactly-once Kafka transactions, would have been
heavier than the problem needed.

**4. pgvector in the existing Postgres instead of a separate vector database.** Vectors sit next to the relational data,
so a pod-scoped search is one SQL query with a `WHERE pod_id = ?`, backups cover everything, and there is one less
service to run. The HNSW index (`m = 16, ef_construction = 64`, `vector_l2_ops`) matches the `<->` operator the queries
use. The trade-off: filtering by pod on top of an approximate index can return fewer than `k` rows when a pod is a small
share of the table, which a dedicated store with native metadata filtering handles better.

**5. A small embedding server and a degraded answer path.** I replaced Ollama for embeddings with TEI, which is built
only for embeddings, and swapped the vector column from 768 to 384 dimensions to match the smaller model. Embedding
calls go through a Resilience4j circuit breaker and are cached in Redis. If the embedding or vector search step throws,
the query service falls back to the pod's aggregated text (`pod_indexes`) rather than returning an error. That keeps the
endpoint answering when the model server is down, at the cost of a less focused prompt.

## Inside the code

The job worker's claim and fan-out (`IndexingWorker.java`). The conditional update is what makes duplicate delivery
harmless, and streaming the items keeps memory flat for large pods:

```java
boolean claimed = jobRepository.updateStatusConditionally(
        event.jobId(), JobStatus.PENDING, JobStatus.RUNNING);
if (!claimed) {
    log.warn("Job {} already claimed by another worker, skipping.",
            event.jobId());
    return;
}
// …
podRepository.streamItems(event.podId(), item -> {
    ItemIndexingEvent itemEvent = new ItemIndexingEvent(event.podId(),
            item.getId(), item.getContent(), llmProperties.getEmbeddingModel());
    // Use item.getId() as key for partitioning
    kafkaTemplate.send("item-indexing-events", item.getId(), itemEvent);
    combinedText.append(item.getContent()).append("\n");
    count.incrementAndGet();
});
podIndexRepository.save(
        new PodIndex(event.podId(), combinedText.toString()));
jobRepository.markAsCompleted(event.jobId());
```

The query path with its fallback (`QueryService.java`). Retrieval is scoped to one pod, and a failure in the vector path
degrades to the aggregated text instead of failing the request:

```java
try {
    List<Double> questionEmbedding =
            embeddingClient.generateEmbedding(request.question());
    float[] embeddingArray = toFloatArray(questionEmbedding);
    List<VectorChunkEntity> similarChunks =
            vectorStore.findSimilarByPodId(podId, embeddingArray, TOP_K_CHUNKS);
    context = buildContextFromChunks(similarChunks);
    usedItemIds = similarChunks.stream()
            .map(VectorChunkEntity::getId).toArray(String[]::new);
} catch (Exception e) {
    log.warn("RAG search failed, falling back to PodIndex. Error: {}",
            e.getMessage());
    PodIndex podIndex = podIndexRepository.findByPodId(podId);
    context = podIndex != null ? podIndex.getCombinedText() : "";
    usedItemIds = new String[] { "fallback:pod-index" };
}
```

## Running it

Everything runs from one Compose file: Postgres with pgvector, Redis, TEI, ZooKeeper, Kafka, Prometheus, Grafana,
Zipkin and the app, each with a health check and CPU/memory limits. The app image is a three-stage Dockerfile
(Maven build, Spring Boot layer extraction, non-root JRE 25 Alpine runtime).

```bash
# .env: POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB, GROQ_API_KEY
docker compose up -d --build
# create a pod, add 50 items, start indexing, poll the job
python stress_test.py
# then: POST /pods/{podId}/query  {"question": "..."}
```

## What I'd do next

- **Finish or revert the last refactor.** The latest commit on `main` (July 2026) started moving to LangChain4j
  (`IngestionPipeline`, `HybridSearchService`, `AiPlatformConfig`). Those classes are not wired to any endpoint yet and
  use placeholder text and config, and the same commit removed the Kafka producer/consumer config that the workers
  depend on (JSON serializer, manual acks). The design above is the January 2026 version (`758fe98`). I'd restore that
  config first, then bring LangChain4j in behind the existing `EmbeddingClient` and `VectorStore` interfaces.
- **Fix the float conversion in `ItemIndexingWorker`.** Its `toFloatArray` collector writes every value to index 0, so
  stored vectors are wrong. `QueryService` has a correct loop; both should share one helper with a unit test.
- **Make job status mean "indexed".** A job is marked `COMPLETED` once the item events are published, not once they
  are embedded. I'd track an expected/processed item count per job and complete it from the item worker side.
- **Make retries real.** `processWithRetry` is called from inside the same bean, so Spring's proxies never apply
  `@Retry` or `@Transactional` to it. It should move to a separate bean. The dead-letter topic also has no consumer or
  replay tool yet.
- **Bring tests back and add CI.** An early stress test was removed during the Kafka migration and there are no tests
  now. I'd add Testcontainers tests for Postgres+pgvector and Kafka, and a GitHub Actions build.
- **Measure before claiming.** There is a latency script (`latency_test.ps1`) but no recorded results. I'd run it at a
  few data sizes and publish the numbers, including the effect of the pod filter on HNSW recall.

## Skills this shows

- **Backend (Java/Spring):** Spring Boot 4 on Java 25, a ports-and-adapters layout (domain `VectorStore` /
  repository interfaces, JPA adapters, MapStruct mappers), Liquibase migrations, optimistic locking, streamed JPA queries.
- **Distributed systems:** two-stage Kafka fan-out keyed for partitioning, manual acknowledgements, idempotent consumers
  (conditional status update, deterministic ids), a dead-letter topic.
- **AI / RAG engineering:** embedding server selection (TEI, 384-dim bge-small), pgvector schema and HNSW index, scoped
  kNN retrieval, prompt assembly, pluggable LLM providers (Groq/OpenAI-compatible, Ollama, a fake client for tests).
- **Resilience:** Resilience4j circuit breaker on the embedding client, retry config, fallback to aggregated text.
- **Platform / SRE:** multi-stage non-root Docker image, a nine-service Compose stack with health checks and resource
  limits, Micrometer counters and timers, Zipkin tracing, Prometheus and Grafana containers.
- **Concurrency:** an earlier virtual-thread executor with `CompletableFuture` timeouts and retries, and the reasons I
  moved past it.

---
title: "Example: how a research entry renders"
description: "A reference page showing every element the entry layout supports. Drafts are never published."
type: research
tags: [example, layout]
math: true
abstract: >
  This draft exists so you can see how headings, code, tables, quotes and maths render.
  Copy `_templates/entry.md` to start a real entry.
repo: https://github.com/Raghavendrapara/rc-says
---

## Headings and paragraphs

Body text sits at a comfortable reading width. Inline `code` and [links](https://github.com/Raghavendrapara) are styled.

## Code

```java
@Bean
public ExecutorService ingestionPool() {
    // One virtual thread per task; blocking I/O no longer pins a platform thread.
    return Executors.newVirtualThreadPerTaskExecutor();
}
```

## Tables

| Setting | Before | After |
|---|---|---|
| p99 latency | 420 ms | 95 ms |
| GC pause | 1.2 s | 80 ms |

## Quotes and maths

> A short pull-quote or a note to the reader.

Little's law, as a display equation:

$$L = \lambda W$$

and inline, inside a sentence: $$\lambda$$ is the arrival rate.

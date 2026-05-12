# Part 3: Prompt Preparation Document

**Selected PR:** [aiokafka #217 — `seek_to_beginning` and `seek_to_end` APIs](https://github.com/aio-libs/aiokafka/pull/217)
**Repository:** [aio-libs/aiokafka](https://github.com/aio-libs/aiokafka)

---

## 3.1.1 Repository Context 

aiokafka is a Python library for working with Apache Kafka, and it’s built on Python's `asyncio` framework. It mainly features two high-level classes — `AIOKafkaProducer` and `AIOKafkaConsumer` — which let Python apps publish and consume messages from Kafka topics using `async/await` syntax, all without blocking the event loop.

This library is especially useful for backend developers who are creating data pipelines, microservice event buses, or any setup that needs high throughput and non-blocking interactions with a Kafka cluster. It takes care of managing TCP connections to Kafka brokers, coordinating consumer groups, rebalancing partitions, tracking partition offsets, and implementing the Kafka wire protocol. Plus, there's an optional Cython extension for speeding up the serialization and deserialization of records.

The code is split into several subpackages: `aiokafka/consumer/` for the consumer logic, fetcher, coordinator, and subscription state; `aiokafka/producer/` for the producer, message accumulator, and sender; `aiokafka/protocol/` for the Kafka binary protocol message definitions; and `aiokafka/` for client connections, errors, and structs. The consumer part is a bit more intricate, handling concurrent tasks like heartbeating, offset committing, group coordination, and record fetching, all organized through asyncio primitives.

The library is meant for Python 3.6 and up and is released under the Apache 2.0 license. It's become the go-to asyncio Kafka client in many production Python async services. Testing it out does require a live Kafka broker, which is often set up using Docker through the project's `Makefile`-based test runner.

---

## 3.1.2 Pull Request Description 

This pull request introduces a couple of new public methods to the `AIOKafkaConsumer` class: `seek_to_beginning(*partitions)` and `seek_to_end(*partitions)`. Both of these methods are `async` coroutines that take an optional list of `TopicPartition` objects. If you don’t specify any partitions, the methods will apply to all currently assigned ones.

**Previous behavior**: Previously, there wasn't an easy way to reset the fetch position for a consumer to the earliest or latest offset all in one go. Developers had to call `consumer.beginning_offsets(partitions)` or `consumer.end_offsets(partitions)` to get the offset values, and then loop through the results to call `consumer.seek(tp, offset)` for each partition. This process could be quite verbose, required dealing with the returned dictionary, and had the risk of issues arising if the assigned partitions changed between those calls.

**New behavior**: Now, the developer can simply call `await consumer.seek_to_beginning()` or `await consumer.seek_to_end()`, with the option to focus on specific partitions. The library takes care of resolving the offsets and updating the positions behind the scenes. It employs a lazy-resolution method: instead of making a broker request right away, it sets a reset flag in the relevant partition's state. The actual `ListOffsetsRequest` will be sent to the broker the next time the fetcher sends out a `FetchRequest` for those partitions. This helps avoid unnecessary network calls, especially if the consumer is paused or the partitions get reassigned before the next fetch.

Validation is pretty strict: if you try to call either method with a partition that isn't in the current assignment, you'll get an `IllegalStateError` right away. Also, remember that the consumer needs to be started (you should call `await consumer.start()`) before using these new methods.

---

## 3.1.3 Acceptance Criteria

- If you call `await consumer.seek_to_beginning()` without any arguments, it sets the fetch position for all assigned partitions back to the earliest available offset before returning the next message.

- Similarly, if you call await `consumer.seek_to_end()` with no arguments, it resets the fetch position for all assigned partitions to the latest offset. This means going forward, only new messages will be consumed.

- If you use either method with a specific list of `TopicPartition` objects, only those specified partitions will be repositioned. The others will keep fetching from their current positions as they are.

- If you try to call either method with a `TopicPartition` that isn’t currently assigned to the consumer, it should immediately raise an `IllegalStateError`, or an equivalent exception, without making any requests to the broker.

- Once you call `seek_to_beginning()` or `seek_to_end()`, the next time the consumer calls `getone()` or `getmany()`, it will return messages that match the reset position — that is, the oldest available message after `seek_to_beginning(`) or a message produced after the call if you used `seek_to_end()`.

- Remember, both of these methods are coroutines (`async def`) and need to be awaited; calling them synchronously should raise a `TypeError` or something similar.

- Lastly, both methods function correctly within a `ConsumerRebalanceListene`r callback without causing deadlocks or leaving the consumer in an inconsistent state.

---

## 3.1.4 Edge Cases (Minimum 3)

**Edge Case 1 — Empty partition list vs. no argument:**
So, the `*partitions` signature means that when you call `seek_to_beginning()` without any arguments, it should affect all assigned partitions. But if you do `seek_to_beginning(*[])`, which is an empty unpacked list, the behavior might change if the implementation checks `if partitions` without making a distinction between "no argument" and an "empty sequence." The implementation needs to treat both situations the same way, or at least make it crystal clear in the documentation.

**Edge Case 2 — Consumer not yet started or already stopped:**
If a user tries to call `seek_to_beginning()` before running `await consumer.start()` or after `await consumer.stop()`, there won't be any assigned partitions or an active connection. The method should either raise an `IllegalStateError` or return quietly but with a clear explanation. If it just returns silently when the consumer is stopped, that could hide bugs in the application code.

**Edge Case 3 — Partition reassignment race condition:**
In a consumer group that has dynamic partition assignment, it's possible for partitions to be revoked between the moment `seek_to_beginning()` sets the reset flag and when the fetcher tries to resolve the offset. The fetcher needs to handle situations where a flagged partition isn’t assigned anymore by the time it sends out the `ListOffsetsRequest`, without throwing an uncaught exception or messing up the subscription state.

**Edge Case 4 — `read_committed` isolation level:**
When the consumer is set to `isolation_level="read_committed"`, the `seek_to_end()` call should actually go to the Last Stable Offset (LSO) instead of the high watermark. The implementation has to respect the isolation level when figuring out what "end" means, and this should line up with how `end_offsets()` is already described to work in this mode.

---

## 3.1.5 Initial Prompt

```
You are implementing two new methods — `seek_to_beginning()` and `seek_to_end()` — for
the `AIOKafkaConsumer` class in the aiokafka Python library (https://github.com/aio-libs/aiokafka).

### Repository Context

aiokafka is an asyncio-based Python client for Apache Kafka. The consumer is implemented
in `aiokafka/consumer/consumer.py`. It uses a `Fetcher` object (in `aiokafka/consumer/fetcher.py`)
to manage fetch requests and offset tracking, and a `SubscriptionState` object
(in `aiokafka/consumer/subscription_state.py`) to track partition assignments and positions.

### What You Need to Implement

Add the following two async methods to the `AIOKafkaConsumer` class:

1. `async def seek_to_beginning(self, *partitions: TopicPartition) -> None`
   - Resets the fetch position of the specified partitions to the earliest available offset.
   - If no partitions are provided, resets ALL currently assigned partitions.

2. `async def seek_to_end(self, *partitions: TopicPartition) -> None`
   - Resets the fetch position of the specified partitions to the latest available offset.
   - If no partitions are provided, resets ALL currently assigned partitions.

### Implementation Approach

Use a lazy-resolution strategy. Do NOT issue a broker `ListOffsetsRequest` at call time.
Instead, mark the relevant partition entries in `SubscriptionState` with an offset reset
strategy (`OffsetResetStrategy.EARLIEST` for `seek_to_beginning`, `OffsetResetStrategy.LATEST`
for `seek_to_end`). The `Fetcher` already knows how to resolve these flags when building
the next `FetchRequest` — use that existing mechanism.

### Validation Requirements

Before setting the reset flag, validate that each specified partition is currently assigned
to this consumer. If any partition is not assigned, raise `IllegalStateError` with a message
like `"Partition {tp} is not currently assigned."`. Do this check before modifying any state,
so the method is atomic — either all partitions are updated or none are.

### Acceptance Criteria to Satisfy

- `seek_to_beginning()` with no args resets all assigned partitions to earliest offset.
- `seek_to_end()` with no args resets all assigned partitions to the latest offset.
- Both methods with specific `TopicPartition` args only affect those partitions.
- Passing an unassigned `TopicPartition` raises `IllegalStateError` immediately.
- After calling either method, `getone()` / `getmany()` returns messages consistent with
  the new position.
- Both methods work correctly inside a `ConsumerRebalanceListener` callback.

### Edge Cases to Handle

- `*partitions` called with an empty unpacked list must behave the same as no-argument call.
- If the consumer is not started, raise `IllegalStateError`.
- Race condition: if a partition is unassigned between the validation check and the actual
  state update, handle gracefully without corrupting `SubscriptionState`.
- Respect `isolation_level="read_committed"` for `seek_to_end()` — seek to LSO, not
  the high watermark.

### Testing

Add integration tests in `tests/test_consumer.py` that:
1. Start a consumer, produce known messages, call `seek_to_beginning()`, and assert the
   first consumed message matches the earliest produced message.
2. Call `seek_to_end()` and assert no old messages are returned — only messages produced
   after the seek are consumed.
3. Call either method with an unassigned partition and assert `IllegalStateError` is raised.
4. Call either method with a subset of assigned partitions and assert only those partitions
   are repositioned.

Reference the existing `test_consumer_seek` tests for setup patterns. Use the `@pytest.mark.asyncio`
decorator and the existing Docker-based Kafka fixture.
```
## Declaration
"I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words."
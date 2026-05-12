# Part 2: Pull Request Analysis

**Repository selected:** [aio-libs/aiokafka](https://github.com/aio-libs/aiokafka)

**Rationale for repository choice:** aiokafka is all about Python and focuses specifically on async Kafka input/output. Its pull requests are aimed at specific features or bug fixes, which makes it easier to analyze technically without having to dive into a huge codebase that handles multiple services.

---

## PR 1: [#193 — Add `offsets_for_times`, `beginning_offsets`, `end_offsets` APIs](https://github.com/aio-libs/aiokafka/pull/193)

*(Part of the v0.3.0 release milestone, addressing issue #164)*

### PR Summary 

Before this pull request, aiokafka's consumer didn’t have a way for users to query partition offsets unless they were seeking by an exact offset number. This left those who needed to replay events from a certain time, or move to the beginning or end of a partition, without a straightforward API. With this PR, we’re introducing three new methods for consumers — `offsets_for_times`, `beginning_offsets`, and `end_offsets` — which are similar to the Kafka Java client APIs. The implementation makes `ListOffsets` requests to the broker and gives back the resolved offset values. Users can then use these with `consumer.seek()` to adjust where they’re consuming from. This change really enhances usability for time-based event replay, monitoring lag, and handling recovery tasks.

### Technical Changes (files/components modified)

- `aiokafka/consumer/consumer.py` — Added three new public async methods: `offsets_for_times()`, `beginning_offsets()`, and `end_offsets()`
- `aiokafka/consumer/fetcher.py` — Added the underlying `_fetcher.offsets_for_times()` and `_fetcher.end_offsets()` / `_fetcher.beginning_offsets()` coroutine implementations that construct and send `ListOffsetsRequest` protocol messages to the broker
- `tests/test_consumer.py` — Added integration tests for each new API, verifying correct offset resolution against a live Kafka broker
- `docs/consumer.rst` — Updated consumer documentation to demonstrate time-based seeking via `offsets_for_times` combined with `seek()`
- `CHANGES.rst` — Changelog entry added for v0.3.0

### Implementation Approach 

The three new methods work in a similar way: they can take either a list of `TopicPartition` objects (for `beginning_offsets` or `end_offsets`) or a dictionary that maps TopicPartition to a timestamp in milliseconds (for `offsets_for_times`). Behind the scenes, these methods call on the `Fetcher` component, which is responsible for building a Kafka `ListOffsetsRequest` protocol message. For the `offsets_for_times`, there are some special timestamps to note: `-2` refers to the earliest available offset, and `-1` corresponds to the latest. If you use a real millisecond epoch timestamp, it will retrieve the offset of the first message that’s sent at or after that specified time.

Once the Fetcher sends the request to the leader broker for each partition, it waits for the response without blocking. The returned offsets are sent back to the caller in a `dict[TopicPartition, OffsetAndTimestamp]` format for `offsets_for_times`, or as `dict[TopicPartition, int]` for the others. This design keeps the public consumer API distinct from the lower-level protocol details, which helps to keep the consumer class tidy and allows for independent testing of the fetcher.

### Potential Impact 

This update impacts the public API of `AIOKafkaConsumer` by introducing three new methods that you can wait on. Don’t worry, any existing code using the consumer will continue to work since these changes are just additive. The Fetcher component will have new routes that make extra requests to the broker, so if there’s a bug with the serialization or response parsing in the `ListOffsetsRequest`, it could affect time-based seeking. Also, keep in mind that the integration test suite in `tests/test_consumer.py` is expanding, which means you'll need to have the Docker-based Kafka broker up and running.

---

## PR 2: [#217 — Add `seek_to_end` and `seek_to_beginning` APIs to `AIOKafkaConsumer`](https://github.com/aio-libs/aiokafka/pull/217)

*(Part of the v0.3.0 release milestone, addressing issue #154)*

### PR Summary 

This PR introduces two handy methods — `seek_to_beginning()` and `seek_to_end()` — to `AIOKafkaConsumer`. Before this update, if someone wanted to reset a consumer to the earliest or latest position in a partition, they had to go through a couple of steps: first, call `beginning_offsets()` or `end_offsets()` to get the offset value, then use that value with `consumer.seek()`. It was pretty cumbersome and could lead to mistakes. Now, with these new methods, you can just make a single awaitable call, and you can even pass in a list of `TopicPartition` objects if you want to limit the reset to certain partitions (by default, it resets all assigned partitions). This is particularly useful for rebalance listeners and situations where you manage offsets manually, especially when you need to keep things simple and accurate while dealing with concurrent access.

### Technical Changes (files/components modified)

- `aiokafka/consumer/consumer.py` — Added `async def seek_to_beginning(*partitions)` and `async def seek_to_end(*partitions)` methods to `AIOKafkaConsumer`
- `aiokafka/consumer/fetcher.py` — Added internal `seek_to_beginning()` and `seek_to_end()` methods that mark the relevant `TopicPartitionState` objects in the subscription state for an offset reset on the next fetch cycle
- `aiokafka/consumer/subscription_state.py` — Modified `PartitionAssignmentState` to support a pending "reset to beginning" or "reset to end" flag per partition, which the fetcher resolves before the next `FetchRequest`
- `tests/test_consumer.py` — Tests added for both methods, including edge cases such as calling with an unassigned partition
- `CHANGES.rst` — Changelog updated for v0.3.0

### Implementation Approach 

Unlike a direct `seek()` call, using `seek_to_beginning()` or `seek_to_end()` doesn’t give you an immediate offset number – that would mean waiting for a broker round-trip, which can block the call. Instead, what happens is that the system marks the target partitions with a reset strategy flag (`EARLIEST` or `LATEST`) in the `SubscriptionState`. This is similar to how Kafka handles the `auto_offset_reset` configuration. When the fetcher needs to send a FetchRequest for those partitions later, it sees that a reset is pending, sends a ListOffsetsRequest to find the actual offset, updates where it’s fetching from, and then clears that flag.

This setup is designed to be lazy on purpose: it prevents any asynchronous lock from being held during the seek call, which keeps the consumer's event loop functioning smoothly. Validation is done right at the start; if someone tries to pass in a partition that isn’t currently assigned to the consumer, it immediately raises an `IllegalStateError` instead of failing silently during the fetch later on. Plus, the optional `*partitions` variadic argument lets you reset specific partitions without messing with the others.

### Potential Impact 

We've added something new to the consumer public API, and don’t worry—nothing breaks existing code. That said, the lazy resolution does mean there’s an extra network round-trip the first time you fetch after seeking. So, keep that in mind when you're looking at latency. Also, the updates in `subscription_state.py` are part of the shared infrastructure for both the rebalance coordinator and the fetcher. This means any issues here could potentially mess up how all partitions are positioned after a group rebalance.

## Declaration
"I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words."
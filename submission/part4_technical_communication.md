# Part 4: Technical Communication

## Task 4.1: Scenario Response

**Reviewer's question:** *"Why did you choose this specific PR over the others? What made it comprehensible to you, and what challenges do you anticipate in implementing it?"*

---

I chose PR #217 (`seek_to_beginning` / `seek_to_end`) over the other nine PRs in the aiokafka list for three concrete reasons.

To start, the problem we're looking at is pretty focused. The PR adds just two new public methods to the `AIOKafkaConsumer` class. It also interacts with a specific set of internal components, like the fetcher and subscription state. Plus, it has a straightforward purpose: move the consumer around, check the input, and raise an error if the state isn’t right. There’s no need to negotiate protocol versions, and we don’t have to worry about security or multiple services interacting. This means I can think through the correctness of its functionality all within one subsystem.

Next, my experience fits perfectly with this. I’ve built and debugged asyncio-based Python services, so I’m aware of the tricky parts that can come with lazy state resolution in async situations. For instance, there's that gap between setting a flag and actually acting on it when the event loop takes control. Also, there's the difference between calling `seek()` (which is synchronous) and figuring out an actual offset (which needs a network call). This PR operates right at that point. I can see how the current `OffsetResetStrategy` enum is used in the fetcher, observe how `auto_offset_reset` ready functions through the same flag mechanism, and expand on it without having to guess.

Lastly, this PR is additive and doesn't break anything. When PRs refactor internal parts or modify existing behaviors, implementing them safely can be tough because a mistake can have a big impact. But this PR introduces new features instead of altering the existing ones.

As for the challenges I anticipate, there are two main ones. The first is a potential race condition between validating assignments and changing the state. In a live consumer group, partitions can be revoked asynchronously during a rebalance while my method is running. I plan to tackle this by locking the subscription state (or using an equivalent asyncio synchronization tool already in the code) during both the validation and flag-setting processes. Alternatively, I could design the fetcher to smoothly skip partitions that are no longer assigned when resolving the pending reset.

The next hurdle is dealing with the `read_committed` isolation level, especially when it comes to `seek_to_end`. For a transactional consumer, what we consider the 'end' is actually the Last Stable Offset, not just the raw high watermark. It would be smart to check out how the current `end_offsets()` method approaches this and just mirror that behavior instead of trying to create a separate logic.


## Declaration
"I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words."

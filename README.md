### Marlin Hiring Assessment — aiokafka Technical Analysis

Here’s my submission for the Marlin hiring assessment. I took a closer look at the `aiokafka` codebase, which is a Python asyncio client for Apache Kafka. My analysis covered four key areas: figuring out the repository, understanding pull requests, engineering prompts, and communicating technical details.

I focused on two pull requests that made their way into the v0.3.0 release. One of them introduces time-based offset querying methods (`offsets_for_times`, `beginning_offsets`, `end_offsets`), while the other brings in the `seek_to_beginning` and `seek_to_end` convenience methods. These are both minor but well-scoped changes, sitting nicely at the crossroads of async Python and Kafka's wire protocol, making them quite engaging to dissect rather than just document mechanically.

The prompt prep document (Part 3) is crafted to function as a direct LLM instruction, rather than just describing what the pull request does. In Part 4, I also pointed out where things get a bit tricky, particularly with the partition reassignment race condition and the `read_committed` LSO edge case.

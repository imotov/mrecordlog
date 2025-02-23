# What is it?

This crate implements a solution to efficiently handle several record logs.
Each recordlog has its own "local" notion of position.
It is possible to truncate each of the queues individually.

# Goals

- be durable, offer some flexibility on `fsync` strategies.
- offer a way to truncate a queue after a specific position
- handle an arbitrary number of queues
- have limited IO
- be fast
- offer the possibility to implement push back

```rust
pub struct MultiRecordLog {
    pub async fn create_queue(&mut self, queue: &str) -> Result<(), CreateQueueError>;
    pub async fn delete_queue(&mut self, queue: &str) -> Result<(), DeleteQueueError>;
    pub fn queue_exists(&self, queue: &str) -> bool;
    pub fn list_queues(&self) -> impl Iterator<Item = &str> {
    pub async fn append_record(
        &mut self,
        queue: &str,
        position_opt: Option<u64>,
        payload: &[u8],
    );
    pub async fn truncate(&mut self, queue: &str, position: u64) -> Result<(), TruncateError>;
    pub fn range<R>(
        &self,
        queue: &str,
        range: R,
    ) -> Option<impl Iterator<Item = (u64, &[u8])> + '_>;
}
```

# Non-goals

This is not Kafka. That recordlog is designed for a "small amount of data".
All retained data can fit in RAM.

In the context of quickwit, that queue is used in the PushAPI and is meant to contain
1 worth of data. (At 60MB/s, means 3.6 GB of RAM)

Reading the recordlog files only happens on startup.
High-performance when reading the recordlog files is not a goal.
Writing fast on the other hand is important.

# Implementation details.

`mrecordlog` is multiplexing several independent queues into the same record log.
This approach has the merit of limiting the number of file descriptor necessary,
and more importantly, to limit the number of `fsync`.

It also offers the possibility to truncate the queue for a given record log.
The actual deletion of the data happens when a file only contains deleted records.
Then, and only then, the entire file is deleted.

That recordlog emits a new file every 1GB.
A recordlog file is deleted, once all queues have been truncated after the
last record of a  of a file.

There are no compaction logic.

# TODO

- add backpressure.
- add fsync policy
- better testing.
- non auto-inc position
- less Arc

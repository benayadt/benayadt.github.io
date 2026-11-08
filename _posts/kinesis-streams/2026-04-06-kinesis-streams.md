---
title: Checkpointing Batched Events in Amazon Kinesis Data Streams
date: 2026-04-06 09:00:00 +00:00
tags: [AWS, Kinesis, Streaming, Data Engineering]
description: How to design safe checkpointing when each Kinesis record contains multiple logical events.
---

When batching is enabled, one Kinesis record can contain several logical events. This improves throughput efficiency, but it also changes how you should checkpoint on the consumer side.

This article focuses on that exact problem: how to checkpoint safely without losing data or creating uncontrolled duplicates.

## What is Kinesis Data Streams?

Amazon Kinesis Data Streams is a managed AWS service for ingesting and processing streaming data in near real time.

At a high level:

- Producers write records continuously to a stream.
- The stream stores records durably across multiple Availability Zones.
- Consumers read and process those records independently.

Kinesis is commonly used for clickstreams, telemetry, application logs, and event-driven data pipelines where low latency and scalability matter.

## Why checkpointing gets tricky with batching

With no batching, checkpointing one processed Kinesis record is straightforward.

With batching, each Kinesis record may contain, for example, 10, 50, or 100 logical events. If your consumer fails after processing only part of the batch, a checkpoint policy that is too aggressive can skip unprocessed events.

The key principle is simple:

- Checkpoint only after every logical event in the current Kinesis record is safely processed.

## Mental model: record-level vs event-level progress

Kinesis sequence numbers are record-level positions. There is no native sequence number per logical event inside a manually batched payload.

So you need two layers of progress:

- **Kinesis checkpoint**: the latest fully completed record.
- **Application progress**: optional internal tracking for events inside a record (useful for recovery logic).

## How sequence handling works with KPL aggregation

If you use KPL aggregation (multiple KPL user records packed into one Kinesis record), the outer Kinesis record still has a normal sequence number.

Inside that aggregated record, user records are indexed with a **subsequence number** starting at `0`.

Practical implications:

- De-aggregation handles unpacking for consumers using KCL or SDK de-aggregation support.
- If your logic uniquely identifies user records, do not rely only on the outer sequence number.
- When needed, treat identity as sequence number + subsequence (or use your own event ID).

Reference: AWS documentation on KPL aggregation and de-aggregation: https://docs.aws.amazon.com/streams/latest/dev/kinesis-producer-adv-aggregation.html

### Java pseudo-code: checkpoint with subsequence (KPL de-aggregation)

```java
class KplDeaggregatedProcessor {

	void processUserRecords(List<UserRecord> userRecords, IRecordProcessorCheckpointer checkpointer) {
		for (UserRecord userRecord : userRecords) {
			String sequence = userRecord.getSequenceNumber();
			long subSequence = userRecord.getSubSequenceNumber();

			if (!alreadyProcessed(sequence, subSequence)) {
				handleBusinessLogic(userRecord);
				markProcessed(sequence, subSequence);
			}

			checkpointer.checkpoint(userRecord);

			// Equivalent explicit form:
			// checkpointer.checkpoint(sequence, subSequence);
		}
	}
}
```

This pattern applies to KPL de-aggregated user records. For manual batching (for example a JSON array in one record), subsequence checkpointing is not available.

According to the AWS docs, `checkpoint()` with no arguments and `checkpoint(String sequenceNumber)` use implicit subsequence `0`, while `checkpoint(Record)` and `checkpoint(String, long)` preserve subsequence-aware progress.

## Safe checkpointing pattern

For each fetched record:

1. Decode and split the batched payload into logical events.
2. Process events one by one (or with controlled parallelism).
3. Ensure side effects are committed safely (database write, API call, or downstream publish).
4. Checkpoint only after the full record is completed.

This is the safest baseline for at-least-once pipelines.

## Java pseudo-code example (KCL + batched payload)

```java
class BatchedRecordProcessor {

	void processRecords(ProcessRecordsInput input) {
		for (KinesisClientRecord record : input.records()) {
			String sequenceNumber = record.sequenceNumber();

			try {
				List<Event> events = decodeBatch(record.data());

				for (int index = 0; index < events.size(); index++) {
					Event event = events.get(index);
					String idempotencyKey = event.eventId();

					if (alreadyProcessed(idempotencyKey)) {
						continue;
					}

					handleBusinessLogic(event);
					markProcessed(idempotencyKey);

					// Optional: local recovery marker for crash diagnostics.
					saveIntraRecordProgress(sequenceNumber, index);
				}

				// Manual batching has no subsequence checkpoint in Kinesis.
				// Checkpoint only after the whole record is processed.
				input.checkpointer().checkpoint(sequenceNumber);
				clearIntraRecordProgress(sequenceNumber);
			} catch (Exception processingError) {
				// Do not checkpoint on failure.
				// The record will be replayed; idempotency handles duplicates.
				logError(sequenceNumber, processingError);
				throw processingError;
			}
		}
	}

	List<Event> decodeBatch(ByteBuffer payload) {
		return BatchCodec.decode(payload);
	}
}
```

In this manual-batching pattern, checkpointing happens once per fully processed Kinesis record (no subsequence checkpoint exists). If the worker crashes mid-record, replay is expected and idempotency prevents incorrect double-processing.

## Failure scenarios and behavior

### Failure before checkpoint

If the consumer crashes before checkpointing, Kinesis will redeliver the same record. Some logical events may run again.

This is expected. Your processing must be idempotent.

### Failure after checkpoint

If checkpoint happens only after all events are done, no logical event from that record is skipped.

### Failure mid-record with partial side effects

This is the hardest case. You avoid data loss by reprocessing the whole record, and you control duplicates with idempotency keys.

## Idempotency is mandatory with batching

Because replay happens at record level, duplicates are normal when failures occur.

Good options:

- Include a stable event ID per logical event.
- Use upserts or conditional writes downstream.
- Store processed-event markers with TTL when needed.

Without idempotency, safer checkpointing becomes very hard in production.

## Throughput vs safety trade-off

Batching improves efficiency, but larger batches increase replay cost on failure.

If one record contains too many events, a single retry can cause a large duplicate burst. Tune batch size based on:

- acceptable duplicate window
- processing time per record
- downstream idempotency strength

## Practical recommendations

- Keep batched records reasonably sized, not maximized by default.
- Validate and parse payload early; fail fast on malformed records.
- Emit metrics for events-per-record, checkpoint lag, and replay count.
- Alert on repeated failures for the same sequence number.
- Test crash/restart paths, not only happy path.

## Conclusion

When batching is enabled, checkpointing strategy becomes a data correctness concern, not just an operational detail.

The safest default is: process all logical events in a Kinesis record, ensure idempotent side effects, then checkpoint once per completed record.

That pattern gives you predictable at-least-once behavior without silent data loss.

## References

- [Implement consumer de-aggregation](https://docs.aws.amazon.com/streams/latest/dev/kinesis-kpl-consumer-deaggregation.html)
- [Apply best practices to KPL aggregation](https://docs.aws.amazon.com/streams/latest/dev/kinesis-producer-adv-aggregation.html)
- [Develop producers using the Amazon Kinesis Producer Library (KPL)](https://docs.aws.amazon.com/streams/latest/dev/developing-producers-with-kpl.html)
- [What is Amazon Kinesis Data Streams?](https://docs.aws.amazon.com/streams/latest/dev/introduction.html)

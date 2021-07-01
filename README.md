# fallbaq

Fallbaq is currently under active development.

Problem: Streaming message bus technologies such as Kinesis, Kafka, RabbitMQ,
NATS Jetstream, etc.. don't allow for back-off strategies and queue technologies
such as Sidekiq and Faktory, which aim to solve the back-off problem by turning
the message bus into a job server, do not scale past a single node. Further, in
the majority of these technologies, partitioning of data is tacked on as an
afterthought by running multiple copies of the same service, leaving it up to
the end-user to manage copies (partitions) of the infrastructure manually and
the developers to invent partitioning strategies for producing to and consuming
from them.

Background: Most of these systems were designed as message busses for
high-volume / low-latency message transport. These systems fall short in
environments where messages may need to be retried. With the nature of the
internet, downstream consumers may go offline for an unspecified amount of time.
It is only a matter of time before a downstream consumer goes offline and the
message queue begins to back up, sending off alarms and waking up SREs and
developers in the middle of the night. What's more, under a single message bus
multi-tenant system, a given tenant may overload the message bus, [thus causing
issues for other tenants](https://segment.com/blog/introducing-centrifuge/).
Many of us have experienced this first hand in systems such as Kafka and
Kinesis, where messages must be consumed and acknowledged in sequence. Often, we
do not want to offload data into a dead letter queue for the sake of continuing
to make progress on a streaming consumer. We want to retry consuming the
messages at some point in the future --maybe when the downstream consumer comes
back online.

I have created previous iterations of this service over the last 14 years. A
recent one can be found here:
[https://github.com/pantry-io/pantry](https://github.com/pantry-io/pantry). This
system had its own requirements. Due to the nature of those requirements, it was
designed to publish messages as quickly as possible at downstream consumers,
retrying upon failure - [lazy pirate
pattern](https://zguide.zeromq.org/docs/chapter4/#Client-Side-Reliability-Lazy-Pirate-Pattern).
Also, message replication was achieved at the data storage layer instead of
clustering amongst a group of nodes [1](https://aws.amazon.com/efs/). Message
replication as disk replication eventually proved too costly. What's more, in a
pub-sub system, the message flow rate can be challenging to tune correctly. This
is where systems such as Kafka and Kinesis shine to allow the consumption of
messages at a consumer desired rate as a pull mechanism. Throughput is increased
by scaling up workers (consumers), creating partitions, and consumer groups: the
more workers, the greater the theoretical throughput in processing messages.

Experience with the problems mentioned above, I have set out to build a system
that meets the following requirements.

- [ ] The system should require little to no dev-ops maintenance. Any manual
      dev-ops maintenance [is considered an
      error](https://twitter.com/NickPoorman/status/1410689001938374661). Any
      manual tuning of partitions should be considered an error.
- [ ] A single tenant should not be able to starve out another tenant of the
      system.
- [ ] The system should accept messages for any number of subjects.
- [ ] The system should scale horizontally to allow for an unbounded message
      producer rate.
- [ ] The system should allow multiple consumers to pull from a single subject
      as a group; thus, only one consumer in the group receives a given message
      in a subject.
- [ ] The system should allow multiple consumer groups on a given subject, a
      copy of each message sent to a single consumer in each group.
- [ ] Messages and system state should be replicated to allow for failure of
      nodes.
- [ ] Message travel latency between producer and consumer should be relatively
      low but should be expected to be greater than that of an in-memory /
      non-clustered / non-replicated system.
- [ ] Producers should be able to produce at an unbounded rate and be resilient
      to throughput spikes.
- [ ] Consumer groups should be able to request data starting from a particular
      offset.
- [ ] High-level APIs may be provided for determining offsets at a given
      timestamp.
- [ ] High-level queue APIs should be provided for leasing messages and
      automated retries of failed messages.
- [ ] Failed messages should be retried with a back-off to allow consumers to
      continue making progress without stalling upon processing a particularly
      troublesome message in a subject.
- [ ] Messages in queues may have a max time to live set by producers. This
      allows messages to be discarded, which may no longer be relevant after a
      specific time has elapsed.
- [ ] Leases on messages in queues should be able to be renewed by consumers.
      This is useful if a worker processing a message is making progress. For
      example, a message that signals a download of a large file. While the
      download is in progress, the worker must be able to renew the lease to
      prevent the message from going back into the queue and picked up by
      another worker.
- [ ] Queues should be pull-based but may not provide order guarantees. Having
      more than one worker consuming messages in the queue group would remove
      any order guarantees anyway. If order guarantees are necessary, consumer
      access to the distributed log should be provided to clients. Further,
      messages should have a sequential message-id only within a given
      partition.
- [ ] Distributed write-ahead-log data must be able to be truncated efficiently
      to make room on the storage device.
- [ ] Before truncating distributed write-ahead-log data, users should have the
      ability to archive and offload the data elsewhere for longer-term storage.
- [ ] A push-based consumer type may be provided to remove the need for thin
      consumers that simply forward HTTP data onto downstream destinations.
- [ ] Consumer groups may provide throttling capabilities to avoid overloading
      downstream consumers. This theoretically could be implemented in every
      consumer, but it is a nice-to-have feature that's easily solved with the
      retry capability.
- [ ] Another nice-to-have would be the ability to only accept unique messages
      into a queue within a time window. Thus de-duplicating messages from
      producers.
- [ ] Messages should be encrypted in transit and at rest.
- [ ] Producers should be able to write batches of messages that are committed
      atomically by the distributed log.

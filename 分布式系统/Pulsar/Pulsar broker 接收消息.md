# Pulsar broker 接收消息

Pulsar broker 处理客户端请求的核心类是 `ServerCnx` 类，该类的继承结构如下，其底层依赖 *Netty* 通过 `ChannelInboundHandle` 接口中的方法可以实现消息的接收和处理，当 *Netty* 接收到客户端发送过来的消息时，会调用 `ChannelInboundHandle` 接口中的 `channelRead(ChannelHandlerContext ctx, Object msg)` 方法将消息传递给该接口的实现类进行处理。`ServerCnx` 类上面有三层父类，顶级父类 `ChannelInboundHandlerAdapter` 实现了 `ChannelInboundHandle` 中的所有方法，并且实现都很简单，仅仅是把 `ChannelHandlerContext` 对象传递给下一级 `ChannelInboundHandler`；`PulsarDecoder`类只覆盖了 `channelRead(ChannelHandlerContext ctx, Object msg)` 方法，该方法中会取出消息中的 `BaseCommand` 字段，根据该字段的类型执行不同的处理逻辑，如 `SEND` 类型是接收客户端发送的消息，会调用其内部的 `protected handleSend()` 方法进行处理，子类可以覆盖该方法实现具体的消息处理逻辑；`PulsarHandle` 类实现了 *channel* 连接的保活机制；`ServerCnx` 类覆盖了其父类的 `handleSend(CommandSend send, ByteBuf headersAndPayload)` 方法，实现了对消息的处理。

![ServerCnx](_v_images/20200905154725792_6755.png =800x)

在 `handleSend()` 方法中会根据命令中的 *producer id* 找到在 *broker* 中对应的实例。然后判断这个 *producer* 对应的 *topic* 是不是持久化的，我们目前的 *topic* 都是持久化的，所以最终会调用 `Producer.publishMessage()` 对消息进行持久化。

```java
@Override
protected void handleSend(CommandSend send, ByteBuf headersAndPayload) {
    checkArgument(state == State.Connected);

    CompletableFuture<Producer> producerFuture = producers.get(send.getProducerId());

    if (producerFuture == null || !producerFuture.isDone() || producerFuture.isCompletedExceptionally()) {
        log.warn("[{}] Producer had already been closed: {}", remoteAddress, send.getProducerId());
        return;
    }

    Producer producer = producerFuture.getNow(null);
    if (log.isDebugEnabled()) {
        printSendCommandDebug(send, headersAndPayload);
    }

    if (producer.isNonPersistentTopic()) {
        // avoid processing non-persist message if reached max concurrent-message limit
        if (nonPersistentPendingMessages > MaxNonPersistentPendingMessages) {
            final long producerId = send.getProducerId();
            final long sequenceId = send.getSequenceId();
            service.getTopicOrderedExecutor().executeOrdered(producer.getTopic().getName(), SafeRun.safeRun(() -> {
                ctx.writeAndFlush(Commands.newSendReceipt(producerId, sequenceId, -1, -1), ctx.voidPromise());
            }));
            producer.recordMessageDrop(send.getNumMessages());
            return;
        } else {
            nonPersistentPendingMessages++;
        }
    }

    startSendOperation();

    // Persist the message
    producer.publishMessage(send.getProducerId(), send.getSequenceId(), headersAndPayload, send.getNumMessages());
}
```

每一个 *producer* 都唯一对应一个 *topic*，在 `Producer.publishMessage()` 方法中会判断 *topic* 是否已经关闭、如果消息的校验和可用的话还会验证校验和、并会判断消息是不是被加密了，被加密的消息会向客户端发送错误码并返回。最终会调用 `Topic` 接口的 `publishMessage()` 方法。

```java
public void publishMessage(long producerId, long sequenceId, ByteBuf headersAndPayload, long batchSize) {
    if (isClosed) {
        cnx.ctx().channel().eventLoop().execute(() -> {
            cnx.ctx().writeAndFlush(Commands.newSendError(producerId, sequenceId, ServerError.PersistenceError,
                    "Producer is closed"));
            cnx.completedSendOperation(isNonPersistentTopic);
        });

        return;
    }

    if (!verifyChecksum(headersAndPayload)) {
        cnx.ctx().channel().eventLoop().execute(() -> {
            cnx.ctx().writeAndFlush(
                    Commands.newSendError(producerId, sequenceId, ServerError.ChecksumError, "Checksum failed on the broker"));
            cnx.completedSendOperation(isNonPersistentTopic);
        });
        return;
    }

    if (topic.isEncryptionRequired()) {

        headersAndPayload.markReaderIndex();
        MessageMetadata msgMetadata = Commands.parseMessageMetadata(headersAndPayload);
        headersAndPayload.resetReaderIndex();

        // Check whether the message is encrypted or not
        if (msgMetadata.getEncryptionKeysCount() < 1) {
            log.warn("[{}] Messages must be encrypted", getTopic().getName());
            cnx.ctx().channel().eventLoop().execute(() -> {
                cnx.ctx().writeAndFlush(Commands.newSendError(producerId, sequenceId, ServerError.MetadataError,
                        "Messages must be encrypted"));
                cnx.completedSendOperation(isNonPersistentTopic);
            });
            return;
        }
    }

    startPublishOperation();
    topic.publishMessage(headersAndPayload,
            MessagePublishContext.get(this, sequenceId, msgIn, headersAndPayload.readableBytes(), batchSize));
}
```

`Topic` 接口有两个实现类：`PersistentTopic` 和 `NonPersistentTopic`。`PersistentTopic` 内部有一个 `ManagedLedger` 类的实例，该类负责管理该 *topic* 中所有的 *ledger*，消息最终是追加写入到一个一个的 *ledger* 中的。最终会调用 `ManagedLedger.asyncAddEntry()` 方法由 `ManagedLedger` 持久化消息。并且 `PersistentTopic` 类实现了 `AddEntryCallback` 接口，其将自身作为 `ManagedLedger.asyncAddEntry()` 方法的参数，`ManagedLedger` 会根据持久化的结果调用 `addComplete()` 或者 `closeFailed()` 方法。

```java
@Override
public void publishMessage(ByteBuf headersAndPayload, PublishContext publishContext) {
    if (messageDeduplication.shouldPublishNextMessage(publishContext, headersAndPayload)) {
        ledger.asyncAddEntry(headersAndPayload, this, publishContext);
    } else {
        // Immediately acknowledge duplicated message
        publishContext.completed(null, -1, -1);
    }
}
```

在 `ManagedLedger` 类中，会根据消息中的数据创建一个 `OpAddEntry` 对象，该对象内部实现了具体的持久化逻辑，也即如何写入数据。但是目前 `ManagedLedger` 还未设置需要将数据写入到哪一个 *ledger* 中。

```java
@Override
public void asyncAddEntry(ByteBuf buffer, AddEntryCallback callback, Object ctx) {
    if (log.isDebugEnabled()) {
        log.debug("[{}] asyncAddEntry size={} state={}", name, buffer.readableBytes(), state);
    }

    OpAddEntry addOperation = OpAddEntry.create(this, buffer, callback, ctx);

    // Jump to specific thread to avoid contention from writers writing from different threads
    executor.executeOrdered(name, safeRun(() -> {
        pendingAddEntries.add(addOperation);

        internalAsyncAddEntry(addOperation);
    }));
}
```

决定向哪一个 *ledger* 中写入数据的逻辑是在 `internalAsyncAddEntry()` 方法中，该方法会根据 `ManagedLedger` 的状态来完成 `OpAddEntry`。

```java
enum State {
    None, // Uninitialized
    LedgerOpened, // A ledger is ready to write into
    ClosingLedger, // Closing current ledger 写满了
    ClosedLedger, // Current ledger has been closed and there's no pending
                  // operation
    CreatingLedger, // Creating a new ledger
    Closed, // ManagedLedger has been closed
    Fenced, // A managed ledger is fenced when there is some concurrent
            // access from a different session/machine. In this state the
            // managed ledger will throw exception for all operations, since
            // the new instance will take over
    Terminated, // Managed ledger was terminated and no more entries
                // are allowed to be added. Reads are allowed
}
```

如果 `ManagedLedger` 当前的状态是 `Fenced`、`Terminated` 或者 `Closed` 则会直接使 `OpAddEntry` 失败，调用 `OpAddEntry.failed()` 方法。如果状态是 `ClosingLedger` 或者 `CreatingLedger` 则不执行任何操作，`OpAddEntry` 会在队列中等待。如果状态是 `ClosedLedger` 则说明此 `OpAddEntry` 到达时 *ledger* 已经被关闭了，新的 *ledger* 正在创建中，这里会判断一下 `OpAddEntry` 到达的时间距离最新的 *ledger* 创建失败的时间是否超过了给定值（重试时间间隔），如果没超过则直接失败，如果超过了则会将状态修改为 `CreatingLedger` 并开始创建 *ledger*，当前的这个 `OpAddEntry` 会留在队列中等待。如果状态是 `LedgerOpened` 则说明当前的 *ledger* 是就绪状态，将 `OpAddEntry` 设置为当前的 *ledger*，然后会检查当前的 *ledger* 是不是写满了，如果写满了则会将状态修改为 `ClosingLedger`，当前的这个 `OpAddEntry` 会留在队列中等待。如果当前的 *ledger* 没有写满，则会调用 `OpAddEntry.initiate()` 方法。

```java
public void initiate() {
    ByteBuf duplicateBuffer = data.retainedDuplicate();

    // internally asyncAddEntry() will take the ownership of the buffer and release it at the end
    lastInitTime = System.nanoTime();
    ledger.asyncAddEntry(duplicateBuffer, this, ctx);
}
```

在 `OpAddEntry.initiate()` 方法中会调用 `LedgerHandle` 类的 `asyncAddEntry()` 方法。
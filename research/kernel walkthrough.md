
### 1. Epoll loop and socket readiness

- The native `ConnectionsManager` owns an `epolFd` and a single shared read buffer `networkBuffer`, and registers each TCP socket as an `EventObject` of type `Connection`.

```cpp

//76:118:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

networkBuffer = new NativeByteBuffer((uint32_t) READ_BUFFER_SIZE);
...
EventObject::onEvent(uint32_t events) {
    switch (eventType) {
        case EventObjectTypeConnection: {
            Connection *connection = (Connection *) eventObject;
            connection->onEvent(events);
            break;
        }
        ...
    }
}
```

### 2. Connection → ConnectionSocket → recv()

- Each `Connection` is‑a `ConnectionSocket`. When epoll fires, `Connection::onEvent()` (inherited) calls `ConnectionSocket::onEvent(events)`.  
- For readable sockets (`EPOLLIN`), it loops `recv()` into the shared `networkBuffer`, then routes data through TLS/proxy handling and finally to `onReceivedData()`.

```cpp

//22:47:TMessagesProj/jni/tgnet/ConnectionSocket.h

class ConnectionSocket {
...
protected:
    void onEvent(uint32_t events);
    virtual void onReceivedData(NativeByteBuffer *buffer) = 0;
...
};
```

```cpp

//781:993:TMessagesProj/jni/tgnet/ConnectionSocket.cpp

void ConnectionSocket::onEvent(uint32_t events) {
    if (events & EPOLLIN) {
        ...
        NativeByteBuffer *buffer = ConnectionsManager::getInstance(instanceNum).networkBuffer;
        while (true) {
            buffer->rewind();
            readCount = recv(socketFd, buffer->bytes(), READ_BUFFER_SIZE, 0);
            ...
            if (readCount > 0) {
                buffer->limit((uint32_t) readCount);
                lastEventTime = ConnectionsManager::getInstance(instanceNum).getCurrentTimeMonotonicMillis();
                ...
                if (proxyAuthState == 0) {
                    if (ConnectionsManager::getInstance(instanceNum).delegate != nullptr) {
                        ConnectionsManager::getInstance(instanceNum).delegate->onBytesReceived((int32_t) readCount, currentNetworkType, instanceNum);
                    }
                    if (tlsState != 0) {
                        // reassemble TLS records, then:
                        tlsBuffer->rewind();
                        onReceivedData(tlsBuffer);   // decrypted TLS payload
                        ...
                    } else {
                        onReceivedData(buffer);      // raw MTProto transport payload
                    }
                }
            } else if (readCount == 0) {
                break; // orderly close
            }
        }
    }
    ...
}
```

- If MTProxy/TLS disguise is enabled, the first bytes perform a TLS client‑hello dance (`proxyAuthState == 10/11`) with HMAC verification; after that, incoming TLS records are validated and chunked, and the decrypted record body (still MTProto‑level encrypted) is passed to `onReceivedData()`.

### 3. Transport‑level decryption & MTProto packet framing

- `Connection::onReceivedData()` is the first MTProto‑aware step. It:
  - Runs **AES‑CTR** over the just‑read buffer (this is the stream cipher between proxy and Telegram DC, not the MTProto object encryption).
  - Coalesces or buffers partial MTProto packets (`restOfTheData` / `lastPacketLength`).
  - For each full frame, parses the **transport header** (length / quick‑ack bit, EF vs non‑EF variant) and calls `ConnectionsManager::onConnectionDataReceived()` with exactly one MTProto packet in `buffer`.

```cpp

//71:104:TMessagesProj/jni/tgnet/Connection.cpp

void Connection::onReceivedData(NativeByteBuffer *buffer) {
    AES_ctr128_encrypt(buffer->bytes(), buffer->bytes(), buffer->limit(),
                       &decryptKey, decryptIv, decryptCount, &decryptNum);
    ...
    while (buffer->hasRemaining()) {
        uint32_t currentPacketLength = 0;
        uint32_t mark = buffer->position();
        uint32_t len;
        if (currentProtocolType == ProtocolTypeEF) {
            uint8_t fByte = buffer->readByte(nullptr);
            if ((fByte & (1 << 7)) != 0) { ... quick ack ...; continue; }
            if (fByte != 0x7f) {
                currentPacketLength = ((uint32_t) fByte) * 4;
            } else {
                ... read extended length ...
                currentPacketLength = ((uint32_t) buffer->readInt32(nullptr) >> 8) * 4;
            }
            len = currentPacketLength + (fByte != 0x7f ? 1 : 4);
        } else {
            uint32_t fInt = buffer->readUint32(nullptr);
            if ((fInt & 0x80000000) != 0) { ... quick ack ...; continue; }
            currentPacketLength = fInt;
            len = currentPacketLength + 4;
        }
        ...
        buffer->limit(buffer->position() + currentPacketLength);
        uint32_t current_generation = generation;
        ConnectionsManager::getInstance(currentDatacenter->instanceNum)
            .onConnectionDataReceived(this, buffer, currentPacketLength);
        ...
        buffer->position(buffer->limit());
        buffer->limit(old);
        ...
    }
    ...
}
```


- `ConnectionsManager::onConnectionDataReceived()` reads the MTProto **auth_key_id**, uses `Datacenter::decryptServerResponse()` (AES‑IGE + SHA‑256) to obtain the inner plaintext message, validates session id, seqno, etc., and then:
  - For handshake packets (`keyId == 0`), directly calls `TLdeserialize()` with a known `request` type.
  - For normal encrypted messages, calls `TLdeserialize(nullptr, ...)`, which:
    - Peeks the TL constructor id
    - Uses the native `TLClassStore` to instantiate the right C++ TL class, or falls back to `request->deserializeResponse(...)` for RPC calls.
- If native parsing can’t fully interpret the message (e.g. some `TL_msg_container` elements or new constructors), it sets `unparsedBody` and dispatches that **raw TL payload** up to Java via `delegate->onUnparsedMessageReceived(...)` → `TgNetWrapper` → `ConnectionsManager.onUnparsedMessageReceived()`; Java wraps the native buffer, reads the constructor int, and calls `TLClassStore.Instance().TLdeserialize()` to get a Java `TLObject` like `TLRPC.Updates`.

So the complete path is:

**`recv()` → TLS/proxy framing (optional) → AES‑CTR decrypt to MTProto transport bytes → MTProto packet length header parsing → `auth_key_id` / AES‑IGE decrypt to inner plaintext TL stream → constructor id read → TLClassStore → concrete TLObject (or raw TL body forwarded to Java for final TLClassStore there).**
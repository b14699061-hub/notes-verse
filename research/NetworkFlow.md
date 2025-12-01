## Telegram Android – Network Receive Flow

This document explains, step by step, how an incoming Telegram update travels:

- from **the kernel socket buffer**
- through **native tgnet** (C++ networking + MTProto)
- into **Java**, as an **unencrypted, de‑obfuscated `TLObject` (typically `TLRPC.Updates`)**

It also calls out **threads**, so it’s clear what runs where.

---

## 1. Threads & Responsibility

- **Network thread (native, C++)**
  - Owned by native `tgnet::ConnectionsManager` (C++).
  - Started once per account when `ConnectionsManager::loadConfig()` finishes:

```cpp
//3696:3700:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

pthread_create(&networkThread, nullptr, (ConnectionsManager::ThreadProc), this);
```

  - Entry point:

```cpp
//345:361:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

void *ConnectionsManager::ThreadProc(void *data) {
    auto networkManager = (ConnectionsManager *) (data);
    ...
    do {
        networkManager->select();
    } while (!done);
    return nullptr;
}
```

  - Runs the **epoll loop**, timers, and socket I/O; all MTProto framing and decryption happen here.

- **Java “stageQueue” thread**
  - `Utilities.stageQueue` is a single‑thread executor on the Java side.
  - All update processing (`MessagesController.processUpdates`) is dispatched here from native callbacks to keep ordering.

- **Android main thread**
  - UI updates, notifications, etc. are posted with `AndroidUtilities.runOnUIThread`.

Thread lifecycle:
- **Start:** `ConnectionsManager::loadConfig()` (C++) calls `pthread_create`, typically triggered from Java via `ConnectionsManager.native_init(...)` during app startup (`ApplicationLoader.postInitApplication()`).
- **Stop:** controlled by the static `done` flag in `ConnectionsManager.cpp` (set during global cleanup / process teardown); the loop in `ThreadProc` exits when `done == true`.

---

## 2. High‑Level Data Path Overview

Visually:

1. **Kernel socket buffer**
2. `recv()` → `ConnectionSocket::onEvent(EPOLLIN)`
3. Optional **TLS / proxy** framing (MTProxy disguise)
4. **AES‑CTR stream decryption** (transport level, between proxy and DC)
5. **MTProto transport frame parsing** (length + quick‑ack)
6. **MTProto v2 object decryption** (AES‑IGE with message key)
7. **TL constructor dispatch** (native `TLClassStore`, `ConnectionsManager::TLdeserialize`)
8. If unparsed: **JNI callback to Java** with raw TL body
9. Java: `NativeByteBuffer.wrap` → `TLClassStore.Instance().TLdeserialize` → `TLRPC.Updates`
10. Java: `MessagesController.processUpdates()` on `stageQueue` → UI & storage.

The rest of this file walks these steps in detail.

---

## 3. Socket → Native Buffer (C++ network thread)

### 3.1 Epoll initialization

Per account, native `ConnectionsManager` creates its own `epoll` instance and shared read buffer:

```cpp

//47:63:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

ConnectionsManager::ConnectionsManager(int32_t instance) {
    instanceNum = instance;
    if ((epolFd = epoll_create(128)) == -1) { ... }
    ...
    if ((epollEvents = new epoll_event[128]) == nullptr) { ... }
    ...
    sizeCalculator = new NativeByteBuffer(true);
    networkBuffer = new NativeByteBuffer((uint32_t) READ_BUFFER_SIZE);
}
```

Each `Connection` is wrapped in an `EventObject` and registered with epoll; when the socket is readable/writable, `EventObject::onEvent()` forwards to the `Connection`:

```cpp

//20:26:TMessagesProj/jni/tgnet/EventObject.cpp

void EventObject::onEvent(uint32_t events) {
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

The network thread loop:

```cpp

//203:212:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

void ConnectionsManager::select() {
    checkPendingTasks();
    int eventsCount = epoll_wait(epolFd, epollEvents, 128, callEvents(getCurrentTimeMonotonicMillis()));
    checkPendingTasks();
    int64_t now = getCurrentTimeMonotonicMillis();
    callEvents(now);
    for (int32_t a = 0; a < eventsCount; a++) {
        auto eventObject = (EventObject *) epollEvents[a].data.ptr;
        eventObject->onEvent(epollEvents[a].events);
    }
    ...
}
```

### 3.2 Reading from the socket (`recv`)

`Connection` inherits `ConnectionSocket`, which holds the OS socket FD and handles epoll events:

```cpp

//22:47:TMessagesProj/jni/tgnet/ConnectionSocket.h

class ConnectionSocket {
public:
    ...
    void openConnection(std::string address, uint16_t port, std::string secret, bool ipv6, int32_t networkType);
    ...
protected:
    void onEvent(uint32_t events);
    virtual void onReceivedData(NativeByteBuffer *buffer) = 0;
    ...
private:
    int socketFd = -1;
    ...
};
```

When `EPOLLIN` fires, the network thread pulls bytes from the kernel into a shared `NativeByteBuffer`:

```cpp

//781:804:TMessagesProj/jni/tgnet/ConnectionSocket.cpp

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
                // later: onReceivedData(buffer);
            } else if (readCount == 0) {
                break; // FIN, socket closed
            }
        }
    }
    ...
}
```

From here, the flow diverges slightly depending on whether **TLS disguise / SOCKS proxy** is in use.

---

## 4. TLS / Proxy Framing (Optional)

Before Telegram bytes are visible, the code may:

- perform a **SOCKS5 handshake** (`proxyAuthState` 1–6)
- perform a **TLS ClientHello and verification** using a shared secret (`proxyAuthState` 10–11, `tlsState != 0`)

Key idea:

- While `proxyAuthState != 0`, the code interprets `buffer->bytes()` as proxy/TLS protocol, not MTProto, and only calls `onReceivedData()` once the TLS layer has delivered a full ciphertext record.

For TLS response records:

```cpp

//921:993:TMessagesProj/jni/tgnet/ConnectionSocket.cpp

} else if (proxyAuthState == 0) {
    ...
    if (tlsState != 0) {
        while (buffer->hasRemaining()) {
            ...
            // Reassemble TLS record header + payload into tlsBuffer
            if (newBytesRead >= 5) {
                ...
                uint8_t header[5];
                ...
                static std::string header1 = std::string("\x17\x03\x03", 3);
                if (std::memcmp(header1.data(), header, header1.size()) != 0) { ... }
                uint32_t len1 = (header[3] << 8) + header[4];
                ...
                tlsBuffer = BuffersStorage::getInstance().getFreeBuffer(len1);
                ...
            }
            ...
            tlsBuffer->writeBytes(buffer);
            ...
            if (tlsBuffer->remaining() == 0) {
                tlsBuffer->rewind();
                onReceivedData(tlsBuffer);  // TLS record payload delivered
                ...
            }
        }
    } else {
        onReceivedData(buffer);          // no TLS, raw transport payload
    }
}
```

At this point, `onReceivedData()` sees **transport‑level encrypted MTProto frames** (not yet MTProto object plaintext).

---

## 5. Transport‑Level Decryption & Framing (MTProto transport)

### 5.1 AES‑CTR (proxy/DC transport)

`Connection::onReceivedData()` runs on the **native network thread** and is the first MTProto‑aware stage:

- **Decrypts** the incoming chunk in place using AES‑CTR (`decryptKey`, `decryptIv`).
- Deals with packet boundaries, buffering partial frames in `restOfTheData` if needed.
- For each full MTProto transport frame, extracts the length and forwards the body to `ConnectionsManager::onConnectionDataReceived()`.

```71:104:TMessagesProj/jni/tgnet/Connection.cpp
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
            if ((fByte & (1 << 7)) != 0) {
                // quick ack
                buffer->position(mark);
                ...
                ConnectionsManager::getInstance(...).onConnectionQuickAckReceived(...);
                continue;
            }
            if (fByte != 0x7f) {
                currentPacketLength = ((uint32_t) fByte) * 4;
            } else {
                buffer->position(mark);
                ...
                currentPacketLength = ((uint32_t) buffer->readInt32(nullptr) >> 8) * 4;
            }
            len = currentPacketLength + (fByte != 0x7f ? 1 : 4);
        } else {
            if (buffer->remaining() < 4) { ... stash partial ...; break; }
            uint32_t fInt = buffer->readUint32(nullptr);
            if ((fInt & (0x80000000)) != 0) {
                ConnectionsManager::getInstance(...).onConnectionQuickAckReceived(...);
                continue;
            }
            currentPacketLength = fInt;
            len = currentPacketLength + 4;
        }
        ...
        buffer->limit(buffer->position() + currentPacketLength);
        ConnectionsManager::getInstance(currentDatacenter->instanceNum)
            .onConnectionDataReceived(this, buffer, currentPacketLength);
        ...
        buffer->position(buffer->limit());
        buffer->limit(old);
        ...
    }
}
```

The result: each call to `onConnectionDataReceived()` sees exactly one **MTProto v2 message frame**, still encrypted at the MTProto layer.

---

## 6. MTProto v2 Decryption & TL Parsing (native)

### 6.1 Encrypted vs unencrypted (handshake)

`ConnectionsManager::onConnectionDataReceived()` starts by reading the 64‑bit `auth_key_id`:

- `keyId == 0` → **unencrypted** message, used during the initial Diffie‑Hellman handshake (auth key negotiation).
- `keyId != 0` → **encrypted** message body (standard MTProto).

```824:889:TMessagesProj/jni/tgnet/ConnectionsManager.cpp
void ConnectionsManager::onConnectionDataReceived(Connection *connection, NativeByteBuffer *data, uint32_t length) {
    bool error = false;
    if (length <= 24 + 32) { ... handle noop / error codes ...; return; }
    uint32_t mark = data->position();

    int64_t keyId = data->readInt64(&error);
    ...
    Datacenter *datacenter = connection->getDatacenter();
    ...
    if (keyId == 0) {
        int64_t messageId = data->readInt64(&error);
        ...
        uint32_t messageLength = data->readUint32(&error);
        ...
        TLObject *request;
        if (datacenter->isHandshaking(connection->isMediaConnection)) {
            request = datacenter->getCurrentHandshakeRequest(connection->isMediaConnection);
            ...
        } else {
            return;
        }
        deserializingDatacenter = datacenter;
        TLObject *object = TLdeserialize(request, messageLength, data);
        if (object != nullptr) {
            datacenter->processHandshakeResponse(..., object, messageId);
            ...
        }
    } else {
        // Encrypted case (normal MTProto traffic)
        ...
    }
}
```

### 6.2 MTProto object decryption (AES‑IGE + SHA‑256)

For encrypted messages (`keyId != 0`):

1. **Optional custom padding** is stripped if used (for MTProxy obfuscation).
2. `Datacenter::decryptServerResponse()` is called to:
   - Look up the correct MTProto `authKey` for this DC/connection type.
   - Derive AES‑IGE keys from (\(authKey\), `messageKey`) via `generateMessageKey`.
   - AES‑IGE decrypt the payload in place.
   - Validate:
     - message length
     - padding sanity
     - SHA‑256 checksum (recompute message key, compare to header).

```cpp

//924:935:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

} else {
    if (connection->allowsCustomPadding()) {
        uint32_t padding = (length - 24) % 16;
        if (padding != 0) {
            length -= padding;
        }
    }
    if (length < 24 + 32
        || (!connection->allowsCustomPadding() && (length - 24) % 16 != 0)
        || !datacenter->decryptServerResponse(keyId, data->bytes() + mark + 8,
                                              data->bytes() + mark + 24, length - 24, connection)) {
        ... reconnect on failure ...
        return;
    }
    data->position(mark + 24);
    ...
}
```

```cpp

//1245:1273:TMessagesProj/jni/tgnet/Datacenter.cpp

bool Datacenter::decryptServerResponse(int64_t keyId, uint8_t *key, uint8_t *data, uint32_t length, Connection *connection) {
    ByteArray *authKey = getAuthKey(connection->getConnectionType(), false, &authKeyId, 1);
    if (authKey == nullptr) {
        return false;
    }
    bool error = authKeyId != keyId;
    thread_local static uint8_t messageKey[96];
    generateMessageKey(instanceNum, authKey->bytes, key, messageKey + 32, true, 2);
    aesIgeEncryption(data, messageKey + 32, messageKey + 64, false, false, length);
    ...
    memcpy(&messageLength, data + 28, sizeof(uint32_t));
    uint32_t paddingLength = length - (messageLength + 32);
    error |= (messageLength > length - 32);
    error |= (paddingLength < 12);
    error |= (paddingLength > 1024);
    ...
    SHA256_Update(&sha256Ctx, authKey->bytes + 88 + 8, 32);
    SHA256_Update(&sha256Ctx, data, length);
    SHA256_Final(messageKey, &sha256Ctx);
    for (uint32_t i = 0; i < 16; i++) {
        error |= (messageKey[i + 8] != key[i]);
    }
    return !error;
}
```

After this, `data` contains **cleartext MTProto message body**:

```text
server_salt | session_id | message_id | seq_no | msg_len | TL-serialized object | padding
```

Parsed immediately after decryption:

```cpp

//935:948:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

data->position(mark + 24);

int64_t messageServerSalt = data->readInt64(&error);
int64_t messageSessionId = data->readInt64(&error);
...
int64_t messageId = data->readInt64(&error);
int32_t messageSeqNo = data->readInt32(&error);
uint32_t messageLength = data->readUint32(&error);
...
```

### 6.3 TL constructor → native TLObject

Once the header is parsed and the message is accepted (seqno, duplicate checks), `ConnectionsManager::TLdeserialize()` converts the now‑plaintext TL stream into a C++ `TLObject`:

```cpp

//957:983:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

TLObject *object = nullptr;
...
if (processedStatus != 1) {
    deserializingDatacenter = datacenter;
    object = TLdeserialize(nullptr, messageLength, data);
    TL_rpc_result* res = dynamic_cast<TL_rpc_result*>(object);
    if (res != nullptr) {
        req_msg_id = res->req_msg_id;
    }
    ...
}
...
if (!processedStatus) {
    if (object != nullptr) {
        lastProtocolUsefullData = true;
        connection->setHasUsefullData();
        processServerResponse(object, messageId, messageSeqNo, messageServerSalt, connection, 0, 0);
        ...
    } else {
        if (delegate != nullptr) {
            delegate->onUnparsedMessageReceived(messageId, data, connection->getConnectionType(), instanceNum);
        }
    }
}
```

The helper:

```cpp

//1042:1082:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

TLObject *ConnectionsManager::TLdeserialize(TLObject *request, uint32_t bytes, NativeByteBuffer *data) {
    bool error = false;
    uint32_t position = data->position();
    uint32_t constructor = data->readUint32(&error);
    if (error) {
        data->position(position);
        return nullptr;
    }

    TLObject *object = TLClassStore::TLdeserialize(data, bytes, constructor, instanceNum, error);

    if (error) {
        delete object;
        data->position(position);
        return nullptr;
    }

    if (object == nullptr) {
        if (request != nullptr) {
            ...
            object = request->deserializeResponse(data, constructor, instanceNum, error);
            ...
        } else {
            ...
        }
    }
    if (object == nullptr) {
        data->position(position);
    }
    return object;
}
```

This uses the native `TLClassStore` (C++) built from `MTProtoScheme.h` to map constructor IDs to generated C++ types.

### 6.4 Containers, gzip and RPC plumbing

`processServerResponse()` is the central dispatcher for **parsed** TL messages:

- `TL_new_session_created`, `TL_future_salts` etc. update local state.
- `TL_msg_container` iterates inner messages; any inner message with `unparsedBody` is forwarded to the Java layer as raw TL payload.
- `TL_rpc_result` unwraps RPC responses, including transparent inflation of `TL_gzip_packed` and error handling.

For container messages:

```cpp

//1127:1157:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

} else if (typeInfo == typeid(TL_msg_container)) {
    auto response = (TL_msg_container *) message;
    size_t count = response->messages.size();
    for (uint32_t a = 0; a < count; a++) {
        TL_message *innerMessage = response->messages[a].get();
        int64_t innerMessageId = innerMessage->msg_id;
        ...
        if (innerMessage->unparsedBody != nullptr) {
            if (delegate != nullptr) {
                delegate->onUnparsedMessageReceived(innerMessageId, innerMessage->unparsedBody.get(),
                                                    connection->getConnectionType(), instanceNum);
            }
        } else {
            processServerResponse(innerMessage->body.get(), 0, innerMessage->seqno, messageSalt,
                                  connection, innerMessageId, messageId);
        }
        connection->addProcessedMessageId(innerMessageId);
    }
}
```

For `TL_rpc_result` and gzip:

```cpp

//1228:1289:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

} else if (typeInfo == typeid(TL_rpc_result)) {
    auto response = (TL_rpc_result *) message;
    ...
    if (request->onCompleteRequestCallback != nullptr) {
        ...
        TLObject *result = response->result.get();
        if (typeid(*result) == typeid(TL_gzip_packed)) {
            auto innerResponse = (TL_gzip_packed *) result;
            unpacked_data = decompressGZip(innerResponse->packed_data.get());
            TLObject *object = TLdeserialize(request->rawRequest, unpacked_data->limit(), unpacked_data);
            if (object != nullptr) {
                response->result = std::unique_ptr<TLObject>(object);
            } else {
                response->result = std::unique_ptr<TLObject>(nullptr);
            }
        }
        ...
    }
}
```

At this point, some messages have been fully handled in native code; others, especially `TLRPC.Updates`, are forwarded **up**.

---

## 7. Native → Java Bridge (Unparsed TL → Java TLObject)

### 7.1 JNI delegate callback

When native code decides a TL payload should be handled in Java (typically updates), it calls the delegate:

```cpp

//1150:1154:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

if (innerMessage->unparsedBody != nullptr) {
    if (delegate != nullptr) {
        delegate->onUnparsedMessageReceived(innerMessageId, innerMessage->unparsedBody.get(),
                                            connection->getConnectionType(), instanceNum);
    }
}
```

The delegate implementation lives in `TgNetWrapper.cpp`:

```326:343:TMessagesProj/jni/TgNetWrapper.cpp
class Delegate : public ConnectiosManagerDelegate {
    ...
    void onUnparsedMessageReceived(int64_t reqMessageId, NativeByteBuffer *buffer,
                                   ConnectionType connectionType, int32_t instanceNum) {
        if (connectionType == ConnectionTypeGeneric) {
            jniEnv[instanceNum]->CallStaticVoidMethod(
                jclass_ConnectionsManager,
                jclass_ConnectionsManager_onUnparsedMessageReceived,
                (jlong) (intptr_t) buffer, instanceNum, reqMessageId);
        }
    }
    ...
};
```

This calls the **static Java method**:

```757:771:TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java
public static void onUnparsedMessageReceived(long address, final int currentAccount, long messageId) {
    try {
        NativeByteBuffer buff = NativeByteBuffer.wrap(address);
        buff.setDataSourceType(TLDataSourceType.NETWORK);
        buff.reused = true;
        int constructor = buff.readInt32(true);
        final TLObject message = TLClassStore.Instance().TLdeserialize(buff, constructor, true);
        FileLog.dumpUnparsedMessage(message, messageId, currentAccount);
        if (message instanceof TLRPC.Updates) {
            if (BuildVars.LOGS_ENABLED) {
                FileLog.d("java received " + message);
            }
            KeepAliveJob.finishJob();
            Utilities.stageQueue.postRunnable(() ->
                AccountInstance.getInstance(currentAccount)
                    .getMessagesController()
                    .processUpdates((TLRPC.Updates) message, false)
            );
        } else {
            ...
        }
    } catch (Exception e) {
        FileLog.e(e);
    }
}
```

### 7.2 Java TLObject re‑creation

- `NativeByteBuffer.wrap(address)` obtains a Java‑side view of the same native memory (no copy).
- Java pre‑reads the TL constructor ID, then uses the Java `TLClassStore` to instantiate the proper generated `TLRPC.*` class:

```53:64:TMessagesProj/src/main/java/org/telegram/tgnet/TLClassStore.java
public TLObject TLdeserialize(NativeByteBuffer stream, int constructor, boolean exception) {
    Class objClass = classStore.get(constructor);
    if (objClass != null) {
        TLObject response;
        try {
            response = (TLObject) objClass.newInstance();
        } catch (Throwable e) {
            FileLog.e(e);
            return null;
        }
        response.readParams(stream, exception);
        return response;
    }
    return null;
}
```

Now we finally have an **unencrypted, de‑obfuscated Java `TLObject`** – for updates, usually `TLRPC.Updates`, `TLRPC.TL_updateShortMessage`, `TLRPC.TL_updateShortChatMessage`, etc.

---

## 8. Java‑Side Update Handling

Once `ConnectionsManager.onUnparsedMessageReceived()` has a Java `TLObject`, it:

- Logs and finishes any keep‑alive job.
- Posts work to `Utilities.stageQueue` (a single background queue).
- `MessagesController.processUpdates()` runs on that queue, transforming TL updates into:
  - `MessageObject`s
  - DB writes via `MessagesStorage`
  - UI notifications via `NotificationCenter`
  - Push notifications via `NotificationsController`.

Example snippet (short messages case):

```16776:16964:TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java
public void processUpdates(final TLRPC.Updates updates, boolean fromQueue) {
    ...
    } else if (updates instanceof TLRPC.TL_updateShortChatMessage
            || updates instanceof TLRPC.TL_updateShortMessage) {
        ...
        if (!missingData && getMessagesStorage().getLastPtsValue() + updates.pts_count == updates.pts) {
            TLRPC.TL_message message = new TLRPC.TL_message();
            ...
            MessageObject obj = new MessageObject(currentAccount, message, isDialogCreated, isDialogCreated);
            ...
            AndroidUtilities.runOnUIThread(() -> {
                updateInterfaceWithMessages(dialogId, objArr, 0);
                getNotificationCenter().postNotificationName(NotificationCenter.dialogsNeedReload);
            });
            if (!obj.isOut()) {
                getMessagesStorage().getStorageQueue().postRunnable(() ->
                    AndroidUtilities.runOnUIThread(() ->
                        getNotificationsController().processNewMessages(objArr, true, false, null)
                    )
                );
            }
            getMessagesStorage().putMessages(arr, false, true, false, 0, 0, 0);
        }
        ...
    }
}
```

---

## 9. Visual Summary

In terms of layers and threads:

1. **Linux kernel / TCP stack**  
   ➜ (native network thread, `epoll_wait`)  
   ➜ `ConnectionSocket::onEvent(EPOLLIN)` reads bytes (`recv`) into shared `NativeByteBuffer`.

2. **Proxy / TLS disguise (optional)** – still on **network thread**  
   ➜ validates SOCKS/TLS, reconstructs TLS records  
   ➜ `Connection::onReceivedData()` gets MTProto transport ciphertext.

3. **MTProto transport framing + AES‑CTR** – **network thread**  
   ➜ `Connection::onReceivedData()` decrypts and splits into MTProto frames  
   ➜ `ConnectionsManager::onConnectionDataReceived()` per frame.

4. **MTProto v2 object decryption (AES‑IGE)** – **network thread**  
   ➜ `Datacenter::decryptServerResponse()` verifies `auth_key_id`, SHA‑256, padding  
   ➜ parse header (`server_salt`, `session_id`, `message_id`, `seqno`, `msg_len`).

5. **Native TL parsing** – **network thread**  
   ➜ `ConnectionsManager::TLdeserialize()` reads constructor and builds C++ `TLObject`  
   ➜ `processServerResponse()` handles standard types or forwards unparsed bodies.

6. **JNI bridge to Java** – transitions from **network thread** to **JVM**  
   ➜ `Delegate::onUnparsedMessageReceived()` → `ConnectionsManager.onUnparsedMessageReceived()`  
   ➜ `NativeByteBuffer.wrap(address)` + Java `TLClassStore` → Java `TLObject`.

7. **Java update processing** – **stageQueue thread + main/UI thread**  
   ➜ `Utilities.stageQueue.postRunnable(... processUpdates ...)`  
   ➜ UI / DB / notifications on main thread & storage queues.

This is the full path from **raw socket packet** → **decrypted TLObject** → **user‑visible message** in the Android client.



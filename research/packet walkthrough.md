

## Packet Walkthrough

- **Ciphertext arrives at the socket**  
  `Connection::onReceivedData()` is invoked by the epoll loop once OpenSSL feeds decrypted transport bytes (AES‑CTR used by MTProxy/TLS disguise) into a `NativeByteBuffer`. It immediately AES‑CTR decrypts the chunk in place, coalesces split MTProto packets, honours quick‑ack markers, enforces sane lengths, and extracts individual MTProto frames before handing them to the manager.

```cpp

//71:214:TMessagesProj/jni/tgnet/Connection.cpp

void Connection::onReceivedData(NativeByteBuffer *buffer) {
    AES_ctr128_encrypt(buffer->bytes(), buffer->bytes(), buffer->limit(), &decryptKey, decryptIv, decryptCount, &decryptNum);
    ...
    while (buffer->hasRemaining()) {
        if (currentProtocolType == ProtocolTypeEF) {
            uint8_t fByte = buffer->readByte(nullptr);
            if ((fByte & (1 << 7)) != 0) {
                ... onConnectionQuickAckReceived(...); continue;
            }
            currentPacketLength = fByte != 0x7f ? ((uint32_t) fByte) * 4
                                               : ((uint32_t) buffer->readInt32(nullptr) >> 8) * 4;
            len = currentPacketLength + (fByte != 0x7f ? 1 : 4);
        } else {
            uint32_t fInt = buffer->readUint32(nullptr);
            if ((fInt & 0x80000000) != 0) { onConnectionQuickAckReceived(...); continue; }
            currentPacketLength = fInt;
            len = currentPacketLength + 4;
        }
        ...
        ConnectionsManager::getInstance(...).onConnectionDataReceived(this, buffer, currentPacketLength);
        ...
    }
}
```

- **MTProto envelope parsing + decryption**  
  `ConnectionsManager::onConnectionDataReceived()` inspects the 64‑bit `auth_key_id` at the start of every frame.

  * If `auth_key_id == 0`, the packet is part of the Diffie‑Hellman handshake: the payload is read unencrypted and dispatched to the current handshake request via `TLdeserialize()`.

  * Otherwise the buffer still contains MTProto v2 ciphertext. The manager trims optional padding, delegates to `Datacenter::decryptServerResponse()` (which derives the message key from SHA‑256(authKey + payload), runs AES‑IGE decryption, and validates the SHA‑256 trailer), then rewinds to parse the cleartext session salt, session id, message id, seq no, and body length.

```cpp

//824:999:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

void ConnectionsManager::onConnectionDataReceived(Connection *connection, NativeByteBuffer *data, uint32_t length) {
    ...
    int64_t keyId = data->readInt64(&error);
    if (keyId == 0) {
        int64_t messageId = data->readInt64(&error);
        uint32_t messageLength = data->readUint32(&error);
        TLObject *request = datacenter->isHandshaking(...) ? datacenter->getCurrentHandshakeRequest(...) : nullptr;
        TLObject *object = TLdeserialize(request, messageLength, data);
        datacenter->processHandshakeResponse(..., object, messageId);
    } else {
        if (!datacenter->decryptServerResponse(keyId, data->bytes() + mark + 8,
                                               data->bytes() + mark + 24, length - 24, connection)) {
            connection->reconnect(); return;
        }
        data->position(mark + 24);
        int64_t messageServerSalt = data->readInt64(&error);
        int64_t messageSessionId = data->readInt64(&error);
        int64_t messageId = data->readInt64(&error);
        int32_t messageSeqNo = data->readInt32(&error);
        uint32_t messageLength = data->readUint32(&error);
        ...
        TLObject *object = processedStatus != 1 ? TLdeserialize(nullptr, messageLength, data) : nullptr;
        if (!processedStatus) {
            if (object != nullptr) {
                processServerResponse(object, messageId, messageSeqNo, messageServerSalt, connection, 0, 0);
            } else {
                delegate->onUnparsedMessageReceived(messageId, data, connection->getConnectionType(), instanceNum);
            }
        }
    }
}
```

```cpp

//1245:1273:TMessagesProj/jni/tgnet/Datacenter.cpp

bool Datacenter::decryptServerResponse(int64_t keyId, uint8_t *key, uint8_t *data, uint32_t length, Connection *connection) {
    ByteArray *authKey = getAuthKey(connection->getConnectionType(), false, &authKeyId, 1);
    if (authKey == nullptr || authKeyId != keyId) { return false; }
    generateMessageKey(instanceNum, authKey->bytes, key, messageKey + 32, true, 2);
    aesIgeEncryption(data, messageKey + 32, messageKey + 64, false, false, length);
    ...
    SHA256_Init(&sha256Ctx);
    SHA256_Update(&sha256Ctx, authKey->bytes + 96, 32);
    SHA256_Update(&sha256Ctx, data, length);
    SHA256_Final(messageKey, &sha256Ctx);
    for (uint32_t i = 0; i < 16; i++) {
        if (messageKey[i + 8] != key[i]) { return false; }
    }
    return true;
}
```

- **Deserialization into TLObjects**  
  Once plaintext is available, `ConnectionsManager::TLdeserialize()` reads the constructor id, routes to `TLClassStore`, and, when the constructor is missing (e.g., RPC responses), asks the original request object to parse itself. This is also where gzip‑packed results are transparently inflated before instantiating the TL class.

```cpp

//1042:1083:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

TLObject *ConnectionsManager::TLdeserialize(TLObject *request, uint32_t bytes, NativeByteBuffer *data) {
    uint32_t position = data->position();
    uint32_t constructor = data->readUint32(&error);
    TLObject *object = TLClassStore::TLdeserialize(data, bytes, constructor, instanceNum, error);
    if (object == nullptr && request != nullptr) {
        object = request->deserializeResponse(data, constructor, instanceNum, error);
        if (object != nullptr && error) { delete object; object = nullptr; }
    }
    if (object == nullptr) { data->position(position); }
    return object;
}
```

- **Containers, gzip, and RPC plumbing**  
  `processServerResponse()` handles message containers (`TL_msg_container`), decompression of `TL_gzip_packed`, and `TL_rpc_result` unwinding. Containers are iterated recursively, and any sub‑message tagged as `unparsedBody` is bubbled up to Java via `onUnparsedMessageReceived()`, where `TLClassStore` finally rehydrates the `TLRPC.Updates` objects that `MessagesController` consumes.

```cpp

//1128:1160:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

} else if (typeInfo == typeid(TL_msg_container)) {
    for (auto &inner : response->messages) {
        if (inner->unparsedBody != nullptr) {
            delegate->onUnparsedMessageReceived(inner->msg_id, inner->unparsedBody.get(),
                                                connection->getConnectionType(), instanceNum);
        } else {
            processServerResponse(inner->body.get(), 0, inner->seqno, messageSalt, connection, inner->msg_id, messageId);
        }
        connection->addProcessedMessageId(inner->msg_id);
    }
}
```

```cpp

//1243:1296:TMessagesProj/jni/tgnet/ConnectionsManager.cpp

} else if (typeInfo == typeid(TL_rpc_result)) {
    auto response = (TL_rpc_result *) message;
    ...
    if (request->onCompleteRequestCallback != nullptr) {
        if (typeid(*result) == typeid(TL_gzip_packed)) {
            auto innerResponse = (TL_gzip_packed *) result;
            NativeByteBuffer *unpacked = decompressGZip(innerResponse->packed_data.get());
            TLObject *object = TLdeserialize(request->rawRequest, unpacked->limit(), unpacked);
            response->result = std::unique_ptr<TLObject>(object);
        }
        ...
    }
}
```

- **Bridge back to Java**  
  When the native delegate hits `delegate->onUnparsedMessageReceived(...)`, `TgNetWrapper` invokes `ConnectionsManager.onUnparsedMessageReceived()` on the Java side, wrapping the native buffer into a `NativeByteBuffer`. Java then peeks the TL constructor and uses `TLClassStore.Instance().TLdeserialize()` to return a concrete `TLRPC.Updates`, completing the journey from raw socket bytes to usable Java TL objects.

```cpp

//340:355:TMessagesProj/jni/TgNetWrapper.cpp

void onUnparsedMessageReceived(int64_t reqMessageId, NativeByteBuffer *buffer, ConnectionType connectionType, int32_t instanceNum) {
    if (connectionType == ConnectionTypeGeneric) {
        jniEnv[instanceNum]->CallStaticVoidMethod(jclass_ConnectionsManager,
            jclass_ConnectionsManager_onUnparsedMessageReceived,
            (jlong) (intptr_t) buffer, instanceNum, reqMessageId);
    }
}
```

```cpp

//757:771:TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java

public static void onUnparsedMessageReceived(long address, final int currentAccount, long messageId) {
    NativeByteBuffer buff = NativeByteBuffer.wrap(address);
    buff.setDataSourceType(TLDataSourceType.NETWORK);
    int constructor = buff.readInt32(true);
    final TLObject message = TLClassStore.Instance().TLdeserialize(buff, constructor, true);
    if (message instanceof TLRPC.Updates) {
        Utilities.stageQueue.postRunnable(() ->
            AccountInstance.getInstance(currentAccount).getMessagesController().processUpdates((TLRPC.Updates) message, false));
    }
}
```

### Key Points
- The only plaintext TLObjects Java ever sees are ones that survive the native pipeline above; everything else (including gzip and containers) is resolved before the callback.
- Packet integrity hinges on matching the `auth_key_id` and SHA‑256 checksum in `Datacenter::decryptServerResponse`; any mismatch forces a reconnect, so corrupted packets never reach TL deserialization.
- Containers or unknown constructors are forwarded untouched to Java, letting higher layers evolve without rewriting native code each time the TL schema expands.
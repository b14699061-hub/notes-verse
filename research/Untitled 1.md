# Telegram Send Message Flow: A Software Architecture Analysis

## 1. Introduction

  

The Telegram Android client implements a sophisticated multi-layer architecture for message transmission, spanning Java application code, JNI bridges, and native C++ network libraries. This analysis traces a single message send operation through the complete system stack, examining:

- Request construction and serialization
- Asynchronous callback mechanisms
- Task scheduling and execution
- Request state management and lifecycle
- Message ID generation and tracking
- Response matching and delivery
- Connection session management
- Protocol wrapping and encryption

  

---

  

## 2. Architectural Overview

  

### 2.1 System Layers

  
The send message flow traverses five distinct architectural layers:
  

```

┌─────────────────────────────────────────────────────────────────┐

│ Layer 1: UI Layer (Java)                                        │

│   - User interaction handling                                    │

│   - Message composition                                          │

│   - Input validation                                              │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ Layer 2: Application Layer (Java)                              │

│   - Message object creation                                      │

│   - Request construction (TLRPC)                                │

│   - Callback registration                                        │

│   - State management                                             │

└───────────────────────────┬─────────────────────────────────────┘

                             │ JNI Bridge

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ Layer 3: Network Manager (C++ - Native Thread)                │

│   - Request queuing (requestsQueue)                             │

│   - Task scheduling (pendingTasks)                              │

│   - Request lifecycle management                                │

│   - Message ID generation                                        │

│   - MTProto wrapping                                             │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ Layer 4: Connection Layer (C++)                                │

│   - Protocol selection (EF/EE/DD/TLS)                          │

│   - AES-256-CTR encryption                                      │

│   - Session management                                           │

│   - Sequence number generation                                   │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ Layer 5: Socket Layer (C++)                                    │

│   - ByteStream buffer management                                 │

│   - Epoll event handling                                         │

│   - Non-blocking I/O                                             │

│   - System call: send()                                          │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

                    ┌─────────────────┐

                    │  Network Stack  │

                    │  (Linux Kernel) │

                    └─────────────────┘

```

  

### 2.2 Key Data Structures

  

**Request Object** (`Request.h:24-81`):

  

```cpp

class Request {

    int64_t messageId = 0;                    // MTProto message ID

    int32_t messageSeqNo = 0;                 // Sequence number

    int32_t requestToken = 0;                 // Unique request identifier

    uint32_t datacenterId = 0;                // Target datacenter

    ConnectionType connectionType;             // Connection type

    bool completed = false;                   // Completion flag

    bool cancelled = false;                   // Cancellation flag

  

    // Callback functions

    onCompleteFunc onCompleteRequestCallback;  // Response callback

    onQuickAckFunc onQuickAckCallback;         // Quick ACK callback

    onRequestClearFunc onRequestClearCallback; // Cleanup callback

  

    TLObject *rawRequest;                     // Original request object

    std::unique_ptr<TLObject> rpcRequest;     // Serialized request

    std::vector<int64_t> respondsToMessageIds; // Response tracking

};

```

  

**NetworkMessage** (`Defines.h:60-66`):



```cpp

typedef struct NetworkMessage {

    std::unique_ptr<TL_message> message;      // MTProto message

    bool invokeAfter = false;                 // Invoke-after flag

    bool needQuickAck = false;                // Quick ACK required

    bool forceContainer = false;              // Force container

    int32_t requestId;                        // Request identifier

} NetworkMessage;

```

  

---

  

## 3. Layer-by-Layer Analysis

  

### 3.1 Layer 1: UI Layer (Java)

  

**Location**: `TMessagesProj/src/main/java/org/telegram/ui/ChatActivity.java`

The UI layer captures user input and initiates the send process. While the exact entry point varies by UI implementation, the flow typically involves:
1. User types message in `ChatActivity`
2. Send button triggers message composition
3. Message parameters collected (text, peer, reply info, etc.)
4. `SendMessagesHelper.sendMessage()` invoked

**Key Observation**: The UI layer is decoupled from network operations, delegating all network concerns to the application layer.

---
### 3.2 Layer 2: Application Layer (Java)

**Location**: `TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java`

#### 3.2.1 Message Creation: `sendMessage()`

**File**: `SendMessagesHelper.java:3816-4785`

The `sendMessage()` method is the primary entry point for message sending:
```java
public void sendMessage(SendMessageParams sendMessageParams) {

    // Extract parameters (line 3817-3853)

    String message = sendMessageParams.message;

    long peer = sendMessageParams.peer;

    MessageObject replyToMsg = sendMessageParams.replyToMsg;

    // ... additional parameters

  

    // Validation (lines 3855-3860)

    if (peer == 0) return;

    if (message == null && caption == null) {

        caption = "";

    }

  

    // Message object creation (lines 3888-4646)

    TLRPC.Message newMsg = new TLRPC.Message();

    newMsg.id = getNextLocalMessageId();

    newMsg.random_id = getNextRandomId();

    newMsg.peer_id = getMessagesController().getPeer(peer);

    newMsg.message = message;

    newMsg.date = getCurrentTime();

    newMsg.send_state = MessageObject.MESSAGE_SEND_STATE_SENDING;

  

    // Store in database (line 4622)

    MessagesStorage.getInstance(currentAccount).putMessages(arr, ...);

  

    // Update UI (line 4623)

    MessagesController.getInstance(currentAccount).updateInterfaceWithMessages(peer, objArr, mode);

}

```


**Key Operations**:
1. **Message ID Generation** (`SendMessagesHelper.java:3888-4646`):
   - Local message ID: Negative integer for unsent messages
   - Random ID: 64-bit random identifier for deduplication
   - Server-assigned ID: Replaced upon successful send

2. **Request Construction** (`SendMessagesHelper.java:4723-4781`):

   ```java

   final TLRPC.TL_messages_sendMessage reqSend = new TLRPC.TL_messages_sendMessage();

   reqSend.message = message;

   reqSend.peer = sendToPeer;

   reqSend.random_id = newMsg.random_id;

   reqSend.clear_draft = retryMessageObject == null;

   reqSend.silent = newMsg.silent;

  

   // Reply information (lines 4734-4739)

   if (newMsg.reply_to instanceof TLRPC.TL_messageReplyHeader) {

       reqSend.reply_to = createReplyInput((TLRPC.TL_messageReplyHeader) newMsg.reply_to);

       reqSend.flags |= 1;

   }

  

   // Entities (text formatting) (lines 4755-4758)

   if (entities != null && !entities.isEmpty()) {

       reqSend.entities = entities;

       reqSend.flags |= 8;

   }

   ```

3. **Request Submission** (`SendMessagesHelper.java:6952`):

   ```java

   newMsgObj.reqId = getConnectionsManager().sendRequest(

       req,                                    // TLObject request

       (response, error) -> {                  // Completion callback

           if (error == null) {

               // Process successful response

               processSentMessage(newMsgObj.id);

           } else {

               // Handle error

               handleSendError(newMsgObj, error);

           }

       },

       null,                                   // Quick ACK callback

       null,                                   // Clear callback

       0,                                      // Flags

       DEFAULT_DATACENTER_ID,                  // Datacenter

       ConnectionTypeGeneric,                  // Connection type

       false                                   // Immediate

   );

   ```

**State Management** (`SendMessagesHelper.java:6945`):
- Message stored in `sendingMessages` map
- State tracked: `MESSAGE_SEND_STATE_SENDING`
- Request ID stored in `newMsgObj.reqId` for cancellation

---

### 3.3 Layer 3: Network Manager (C++)


**Location**: `TMessagesProj/jni/tgnet/ConnectionsManager.cpp`

The `ConnectionsManager` runs on a dedicated native thread (`ThreadProc`, line 345) and manages all network operations through an epoll-based event loop.

#### 3.3.1 Thread Architecture
**File**: `ConnectionsManager.cpp:345-362`

```cpp

void *ConnectionsManager::ThreadProc(void *data) {

    auto networkManager = (ConnectionsManager *) data;

  

    // Attach JNI environment for callbacks

    #ifdef ANDROID

    javaVm->AttachCurrentThread(&jniEnv[networkManager->instanceNum], nullptr);

    #endif

  

    // Event loop

    do {

        networkManager->select();  // Epoll-based event loop

    } while (!done);

  

    return nullptr;

}

```

  

**Event Loop** (`ConnectionsManager.cpp:96-200`):
- Uses `epoll_wait()` for I/O multiplexing
- Processes socket events, timers, and task queue
- Single-threaded execution ensures thread safety

#### 3.3.2 Task Queue System

**File**: `ConnectionsManager.cpp:309-312`, `187-198`

The task queue enables thread-safe communication from Java to native thread:  

```cpp

// Task queue declaration (ConnectionsManager.h:198)

std::queue<std::function<void()>> pendingTasks;

  

// Task scheduling (ConnectionsManager.cpp:309-312)

void ConnectionsManager::scheduleTask(std::function<void()> task) {

    pthread_mutex_lock(&mutex);

    pendingTasks.push(task);

    pthread_mutex_unlock(&mutex);

    wakeup();  // Signal event loop

}

  

// Task execution (ConnectionsManager.cpp:187-198)

void ConnectionsManager::checkPendingTasks() {

    std::function<void()> task;

    int32_t count = 0;

  

    pthread_mutex_lock(&mutex);

    if (!pendingTasks.empty()) {

        count = (int32_t) pendingTasks.size();

        task = pendingTasks.front();

        pendingTasks.pop();

    }

    pthread_mutex_unlock(&mutex);

  

    if (task != nullptr) {

        task();  // Execute task

    }

}

```

  

**Key Properties**:
- Thread-safe via mutex (`pthread_mutex_t mutex`)
- Wakeup mechanism via pipe (`pipeFd`) for event loop notification
- Tasks executed synchronously on network thread

#### 3.3.3 Request Submission: `sendRequest()`

**File**: `ConnectionsManager.cpp:1917-2200` (approximate)

When `sendRequest()` is called from Java via JNI:
```cpp

int32_t ConnectionsManager::sendRequest(

    TLObject *object,

    onCompleteFunc onComplete,

    onQuickAckFunc onQuickAck,

    onRequestClearFunc onClear,

    uint32_t flags,

    uint32_t datacenterId,

    ConnectionType connectionType,

    bool immediate) {

  

    // Schedule task on network thread

    scheduleTask([&, requestToken, object, onComplete, onQuickAck, onClear,

                  flags, datacenterId, connectionType, immediate] {

  

        // Create request object

        std::unique_ptr<Request> request = std::make_unique<Request>(

            instanceNum,

            ++lastRequestToken,        // Unique token (ConnectionsManager.h:151)

            connectionType,

            flags,

            datacenterId,

            onComplete,               // Stored for response callback

            onQuickAck,

            nullptr,

            onClear

        );

  

        // Store original request

        request->rawRequest = object;

  

        // Determine target datacenter

        if (datacenterId == DEFAULT_DATACENTER_ID) {

            datacenterId = currentDatacenterId;

        }

  

        // Add to appropriate queue

        if (currentUserId == 0 && !(flags & RequestFlagWithoutLogin)) {

            waitingLoginRequests.push_back(std::move(request));

        } else {

            requestsQueue.push_back(std::move(request));

        }

  

        // Process queue

        processRequestQueue(connectionType, datacenterId);

    });

  

    return lastRequestToken;

}

```

**Request Queues** (`ConnectionsManager.h:216-218`):
- `waitingLoginRequests`: Requests waiting for authentication
- `requestsQueue`: Ready-to-send requests
- `runningRequests`: Requests in-flight (awaiting response)

#### 3.3.4 Request Processing: `processRequestQueue()`

**File**: `ConnectionsManager.cpp:2434-2800` (approximate)

  

```cpp

void ConnectionsManager::processRequestQueue(uint32_t connectionTypes, uint32_t dc) {

    // Select requests from queue

    for (auto iter = requestsQueue.begin(); iter != requestsQueue.end();) {

        Request *request = iter->get();

  

        // Check cancellation

        if (request->cancelled) {

            iter = requestsQueue.erase(iter);

            continue;

        }

  

        // Get target datacenter

        Datacenter *datacenter = getDatacenterWithId(request->datacenterId);

        if (datacenter == nullptr) {

            iter++;

            continue;

        }

  

        // Check authentication

        if (!datacenter->hasAuthKey(request->connectionType, 0)) {

            // Initiate handshake

            datacenter->beginHandshake(...);

            iter++;

            continue;

        }

  

        // Get connection

        Connection *connection = datacenter->getConnectionByType(

            request->connectionType, true, 0);

  

        if (connection == nullptr) {

            iter++;

            continue;

        }

  

        // Wrap request in MTProto layer

        std::unique_ptr<TLObject> wrapped = wrapInLayer(

            request->rawRequest, datacenter, request.get());

  

        // Create NetworkMessage

        auto networkMessage = std::make_unique<NetworkMessage>();

        networkMessage->message = std::move(wrapped);

        networkMessage->requestId = request->requestToken;

  

        // Move to running requests

        request->messageId = generateMessageId();

        request->messageSeqNo = connection->generateMessageSeqNo(true);

        request->connectionToken = connection->getConnectionToken();

  

        runningRequests.push_back(std::move(*iter));

        iter = requestsQueue.erase(iter);

  

        // Add to send queue

        addMessageToDatacenter(request->datacenterId,

                               networkMessage.release(),

                               genericMessagesToDatacenters);

    }

  

    // Send batched messages

    for (auto &iter : genericMessagesToDatacenters) {

        Datacenter *datacenter = getDatacenterWithId(iter.first);

        Connection *connection = datacenter->getGenericConnection(true, 0);

        sendMessagesToConnection(iter.second, connection, false);

    }

}

```


**Key Operations**:

1. **Message ID Generation** (`ConnectionsManager.cpp:131`):

   ```cpp

   int64_t ConnectionsManager::generateMessageId() {

       int64_t time = (int64_t) (getCurrentTimeMonotonicMillis() / 1000);

       int64_t messageId = ((time + timeDifference) << 32) |

                          ((lastOutgoingMessageId++) & 0xFFFFFFFF);

       return messageId;

   }

   ```

  

   - Time-based: Upper 32 bits = server time

   - Monotonic: Lower 32 bits = sequence counter

   - Ensures message ordering and prevents replay attacks

  

2. **Sequence Number Generation** (`ConnectionSession.cpp:24`):

   ```cpp

   uint32_t ConnectionSession::generateMessageSeqNo(bool increment) {

       if (increment) {

           nextSeqNo += 2;  // Odd numbers for client requests

       }

       return nextSeqNo;

   }

   ```

   - Odd numbers: Client requests

   - Even numbers: Server responses

   - Enables response matching

  

#### 3.3.5 Message Transmission: `sendMessagesToConnection()`

  

**File**: `ConnectionsManager.cpp:2285-2335`

  

```cpp

void ConnectionsManager::sendMessagesToConnection(

    std::vector<std::unique_ptr<NetworkMessage>> &messages,

    Connection *connection,

    bool reportAck) {

  

    if (messages.empty() || connection == nullptr) {

        return;

    }

  

    std::vector<std::unique_ptr<NetworkMessage>> currentMessages;

    uint32_t currentSize = 0;

    size_t count = messages.size();

  

    // Batch messages (up to 3KB)

    for (uint32_t a = 0; a < count; a++) {

        NetworkMessage *networkMessage = messages[a].get();

        currentMessages.push_back(std::move(messages[a]));

        currentSize += networkMessage->message->bytes;

  

        // Send batch when size limit reached

        if (currentSize >= 3 * 1024 || a == count - 1) {

            int32_t quickAckId = 0;

  

            // Create transport data with MTProto wrapping

            NativeByteBuffer *transportData =

                datacenter->createRequestsData(

                    currentMessages,

                    reportAck ? &quickAckId : nullptr,

                    connection,

                    false

                );

  

            if (transportData != nullptr) {

                // Map quick ACK to request IDs

                if (reportAck && quickAckId != 0) {

                    std::vector<int32_t> requestIds;

                    for (uint32_t b = 0; b < currentMessages.size(); b++) {

                        if (currentMessages[b]->requestId != 0) {

                            requestIds.push_back(currentMessages[b]->requestId);

                        }

                    }

                    quickAckIdToRequestIds[quickAckId] = requestIds;

                }

  

                // Send via connection

                connection->sendData(transportData, reportAck, true);

            }

  

            currentSize = 0;

            currentMessages.clear();

        }

    }

}

```

  

**Batching Strategy**:

  

- Maximum batch size: 3KB (`ConnectionsManager.cpp:2300`)

- Reduces network overhead

- Maintains request ordering within batch

  

---

  

### 3.4 Layer 4: Connection Layer (C++)

  

**Location**: `TMessagesProj/jni/tgnet/Connection.cpp`

  

The `Connection` class handles MTProto protocol wrapping and encryption.

  

#### 3.4.1 Data Transmission: `sendData()`

  

**File**: `Connection.cpp:430-622`

  

```cpp

void Connection::sendData(NativeByteBuffer *buff, bool reportAck, bool encrypted) {

    if (buff == nullptr) {

        return;

    }

  

    buff->rewind();

  

    // Ensure connection is established

    if (connectionState == TcpConnectionStageIdle ||

        connectionState == TcpConnectionStageReconnecting) {

        connect();

    }

  

    if (isDisconnected()) {

        buff->reuse();

        return;

    }

  

    // Determine protocol type (lines 448-472)

    uint8_t useSecret = 0;

    if (!firstPacketSent) {

        // Check for proxy/secret configuration

        if (!overrideProxyAddress.empty() && !overrideProxySecret.empty()) {

            useSecret = 1;

        } else if (!secret.empty()) {

            useSecret = 2;

        }

  

        // Select protocol based on secret

        std::string *currentSecret = getCurrentSecret(useSecret);

        if (currentSecret->length() >= 17 && (*currentSecret)[0] == '\xdd') {

            currentProtocolType = ProtocolTypeDD;

        } else if (currentSecret->length() > 17 && (*currentSecret)[0] == '\xee') {

            currentProtocolType = ProtocolTypeTLS;

        } else {

            currentProtocolType = ProtocolTypeEF;

        }

    }

  

    // Calculate packet length and padding (lines 475-504)

    uint32_t packetLength;

    uint32_t additinalPacketSize = 0;

  

    if (currentProtocolType == ProtocolTypeEF) {

        packetLength = buff->limit() / 4;  // Length in 4-byte words

        if (packetLength < 0x7f) {

            bufferLen++;

        } else {

            bufferLen += 4;

        }

    } else {

        packetLength = buff->limit();  // Length in bytes

        // Add random padding

        RAND_bytes((uint8_t *) &additinalPacketSize, 4);

        if (!encrypted) {

            additinalPacketSize = additinalPacketSize % 257;

        } else {

            additinalPacketSize = additinalPacketSize % 16;

        }

        packetLength += additinalPacketSize;

        bufferLen += 4;

    }

  

    // First packet handshake (lines 507-586)

    if (!firstPacketSent) {

        bufferLen += 64;  // Handshake header

  

        NativeByteBuffer *buffer = BuffersStorage::getInstance().getFreeBuffer(bufferLen);

        uint8_t *bytes = buffer->bytes();

        buffer->position(64);

  

        // Generate random handshake data

        while (true) {

            RAND_bytes(bytes, 64);

            uint32_t val = (bytes[3] << 24) | (bytes[2] << 16) |

                          (bytes[1] << 8) | (bytes[0]);

            uint32_t val2 = (bytes[7] << 24) | (bytes[6] << 16) |

                           (bytes[5] << 8) | (bytes[4]);

  

            // Validate handshake (avoid certain patterns)

            if (currentProtocolType == ProtocolTypeTLS ||

                (bytes[0] != 0xef && val != 0x44414548 && val != 0x54534f50 &&

                 val != 0x20544547 && val != 0x4954504f && val != 0xeeeeeeee &&

                 val != 0xdddddddd && val != 0x02010316 && val2 != 0x00000000)) {

  

                // Set protocol marker

                if (currentProtocolType == ProtocolTypeEF) {

                    bytes[56] = bytes[57] = bytes[58] = bytes[59] = 0xef;

                } else if (currentProtocolType == ProtocolTypeDD ||

                          currentProtocolType == ProtocolTypeTLS) {

                    bytes[56] = bytes[57] = bytes[58] = bytes[59] = 0xdd;

                } else if (currentProtocolType == ProtocolTypeEE) {

                    bytes[56] = bytes[57] = bytes[58] = bytes[59] = 0xee;

                }

  

                // Embed datacenter ID (lines 536-553)

                if (useSecret != 0) {

                    int16_t datacenterId;

                    if (isMediaConnection) {

                        datacenterId = -(int16_t) currentDatacenter->getDatacenterId();

                    } else {

                        datacenterId = (int16_t) currentDatacenter->getDatacenterId();

                    }

                    bytes[60] = (uint8_t) (datacenterId & 0xff);

                    bytes[61] = (uint8_t) ((datacenterId >> 8) & 0xff);

                }

                break;

            }

        }

  

        // Derive encryption keys from handshake (lines 558-580)

        for (int32_t a = 0; a < 48; a++) {

            temp[a] = bytes[a + 8];

        }

        encryptKeyWithSecret(temp, useSecret);

        AES_set_encrypt_key(temp, 256, &encryptKey);

        memcpy(encryptIv, temp + 32, 16);

  

        for (int32_t a = 0; a < 48; a++) {

            temp[a] = bytes[55 - a];

        }

        encryptKeyWithSecret(temp, useSecret);

        AES_set_encrypt_key(temp, 256, &decryptKey);

        memcpy(decryptIv, temp + 32, 16);

  

        // Encrypt handshake header

        AES_ctr128_encrypt(bytes, temp, 64, &encryptKey, encryptIv,

                          encryptCount, &encryptNum);

        memcpy(bytes + 56, temp + 56, 8);

  

        firstPacketSent = true;

    }

  

    // Write protocol header (lines 587-611)

    NativeByteBuffer *buffer = BuffersStorage::getInstance().getFreeBuffer(bufferLen);

    uint8_t *bytes = buffer->bytes();

  

    if (currentProtocolType == ProtocolTypeEF) {

        if (packetLength < 0x7f) {

            if (reportAck) {

                packetLength |= (1 << 7);  // ACK flag

            }

            buffer->writeByte((uint8_t) packetLength);

            bytes += (buffer->limit() - 1);

            AES_ctr128_encrypt(bytes, bytes, 1, &encryptKey, encryptIv,

                              encryptCount, &encryptNum);

        } else {

            packetLength = (packetLength << 8) + 0x7f;

            if (reportAck) {

                packetLength |= (1 << 7);

            }

            buffer->writeInt32(packetLength);

            bytes += (buffer->limit() - 4);

            AES_ctr128_encrypt(bytes, bytes, 4, &encryptKey, encryptIv,

                              encryptCount, &encryptNum);

        }

    } else {

        if (reportAck) {

            packetLength |= 0x80000000;  // ACK flag in MSB

        }

        buffer->writeInt32(packetLength);

        bytes += (buffer->limit() - 4);

        AES_ctr128_encrypt(bytes, bytes, 4, &encryptKey, encryptIv,

                          encryptCount, &encryptNum);

    }

  

    // Encrypt payload (lines 614-621)

    buffer->rewind();

    writeBuffer(buffer);  // Write encrypted header

  

    buff->rewind();

    AES_ctr128_encrypt(buff->bytes(), buff->bytes(), buff->limit(),

                      &encryptKey, encryptIv, encryptCount, &encryptNum);

    writeBuffer(buff);  // Write encrypted payload

  

    if (buffer2 != nullptr) {

        AES_ctr128_encrypt(buffer2->bytes(), buffer2->bytes(), buffer2->limit(),

                          &encryptKey, encryptIv, encryptCount, &encryptNum);

        writeBuffer(buffer2);  // Write padding

    }

}

```

  

**Encryption Details**:

  

1. **AES-256-CTR Mode**:

  

   - Key size: 256 bits (32 bytes)

   - IV size: 128 bits (16 bytes)

   - Counter mode enables parallel encryption/decryption

   - Separate keys for encrypt/decrypt directions

  

2. **Key Derivation** (`Connection.cpp:562-580`):

  

   - Keys derived from 48-byte random data in handshake

   - Additional encryption with datacenter secret

   - Keys stored in `AES_KEY` structures

  

3. **Protocol Types**:

   - **EF** (`ProtocolTypeEF`): Original protocol, 0xef marker

   - **EE** (`ProtocolTypeEE`): Extended protocol, 0xee marker

   - **DD** (`ProtocolTypeDD`): Datacenter protocol, 0xdd marker

   - **TLS** (`ProtocolTypeTLS`): TLS obfuscation, 0xee + TLS handshake

  

---

  

### 3.5 Layer 5: Socket Layer (C++)

  

**Location**: `TMessagesProj/jni/tgnet/ConnectionSocket.cpp`

  

The `ConnectionSocket` class handles actual network I/O operations.

  

#### 3.5.1 Buffer Management: `writeBuffer()`

  

**File**: `ConnectionSocket.cpp:1176-1179`

  

```cpp

void ConnectionSocket::writeBuffer(NativeByteBuffer *buffer) {

    // Append to outgoing byte stream

    outgoingByteStream->append(buffer);

  

    // Adjust epoll events to enable write

    adjustWriteOp();

}

```

  

**ByteStream Implementation** (`ByteStream.cpp:20-86`):

  

```cpp

class ByteStream {

    std::vector<NativeByteBuffer *> buffersQueue;

  

    void append(NativeByteBuffer *buffer) {

        if (buffer == nullptr) return;

        buffersQueue.push_back(buffer);

    }

  

    bool hasData() {

        for (uint32_t a = 0; a < buffersQueue.size(); a++) {

            if (buffersQueue[a]->hasRemaining()) {

                return true;

            }

        }

        return false;

    }

  

    void get(NativeByteBuffer *dst) {

        // Copy data from queue to destination buffer

        for (uint32_t a = 0; a < buffersQueue.size(); a++) {

            buffer = buffersQueue[a];

            if (buffer->remaining() > dst->remaining()) {

                dst->writeBytes(buffer->bytes(), buffer->position(),

                               dst->remaining());

                break;

            }

            dst->writeBytes(buffer->bytes(), buffer->position(),

                           buffer->remaining());

            if (!dst->hasRemaining()) break;

        }

    }

  

    void discard(uint32_t count) {

        // Remove sent data from queue

        while (count > 0) {

            if (buffersQueue.empty()) break;

            buffer = buffersQueue[0];

            uint32_t remaining = buffer->remaining();

            if (count < remaining) {

                buffer->position(buffer->position() + count);

                break;

            }

            buffer->reuse();

            buffersQueue.erase(buffersQueue.begin());

            count -= remaining;

        }

    }

};

```

  

#### 3.5.2 Epoll Event Handling: `onEvent()`

  

**File**: `ConnectionSocket.cpp:1100-1167`

  

```cpp

void ConnectionSocket::onEvent(uint32_t events) {

    if (events & EPOLLOUT) {

        // Socket is writable

        if (outgoingByteStream->hasData()) {

            NativeByteBuffer *buffer =

                BuffersStorage::getInstance().getFreeBuffer(READ_BUFFER_SIZE);

            outgoingByteStream->get(buffer);

            buffer->flip();

  

            uint32_t remaining = buffer->remaining();

  

            if (remaining) {

                ssize_t sentLength;

  

                // Handle TLS obfuscation (lines 1105-1136)

                if (tlsState != 0) {

                    if (remaining > 2878) {

                        remaining = 2878;  // TLS record size limit

                    }

                    size_t headersSize = 0;

  

                    if (tlsState == 1) {

                        // Write TLS change cipher spec

                        static std::string header1 =

                            std::string("\x14\x03\x03\x00\x01\x01", 6);

                        std::memcpy(tempBuffer->bytes, header1.data(),

                                   header1.size());

                        headersSize += header1.size();

                        tlsState = 2;

                    }

  

                    // Write TLS application data header

                    static std::string header2 = std::string("\x17\x03\x03", 3);

                    std::memcpy(tempBuffer->bytes + headersSize,

                               header2.data(), header2.size());

                    headersSize += header2.size();

  

                    // Write length

                    tempBuffer->bytes[headersSize] =

                        static_cast<uint8_t>((remaining >> 8) & 0xff);

                    tempBuffer->bytes[headersSize + 1] =

                        static_cast<uint8_t>(remaining & 0xff);

                    headersSize += 2;

  

                    // Copy payload

                    std::memcpy(tempBuffer->bytes + headersSize,

                               buffer->bytes(), remaining);

  

                    // Send with TLS framing

                    sentLength = send(socketFd, tempBuffer->bytes,

                                    headersSize + remaining, 0);

  

                    if (sentLength >= headersSize) {

                        delegate->onBytesSent((int32_t) sentLength, ...);

                        outgoingByteStream->discard((uint32_t)

                                                   (sentLength - headersSize));

                        adjustWriteOp();

                    } else {

                        closeSocket(1, -1);

                    }

                } else {

                    // Direct send (lines 1138-1148)

                    sentLength = send(socketFd, buffer->bytes(), remaining, 0);

  

                    if (sentLength > 0) {

                        // Update statistics

                        if (delegate != nullptr) {

                            delegate->onBytesSent((int32_t) sentLength,

                                                 currentNetworkType,

                                                 instanceNum);

                        }

  

                        // Discard sent data

                        outgoingByteStream->discard((uint32_t) sentLength);

  

                        // Adjust epoll events

                        adjustWriteOp();

                    } else if (sentLength < 0) {

                        // Error: Close connection

                        closeSocket(1, -1);

                    }

                }

            }

        }

    }

  

    // Handle other events (EPOLLHUP, EPOLLRDHUP, EPOLLERR)

    if (events & EPOLLHUP) {

        closeSocket(1, -1);

        return;

    } else if (events & EPOLLRDHUP) {

        closeSocket(1, -1);

        return;

    }

    if (events & EPOLLERR) {

        // Log error

        return;

    }

}

```

  

**Epoll Configuration** (`ConnectionSocket.cpp:1181-1195`):

  

```cpp

void ConnectionSocket::adjustWriteOp() {

    if (!waitingForHostResolve.empty()) {

        adjustWriteOpAfterResolve = true;

        return;

    }

  

    // Configure epoll events

    eventMask.events = EPOLLIN | EPOLLRDHUP | EPOLLERR | EPOLLET;  // Edge-triggered

  

    // Add EPOLLOUT if data to send or not yet connected

    if (proxyAuthState == 0 &&

        (outgoingByteStream->hasData() || !onConnectedSent) ||

        proxyAuthState == 1 || proxyAuthState == 3 ||

        proxyAuthState == 5 || proxyAuthState == 10) {

        eventMask.events |= EPOLLOUT;

    }

  

    eventMask.data.ptr = eventObject;

  

    // Update epoll registration

    if (epoll_ctl(ConnectionsManager::getInstance(instanceNum).epolFd,

                  EPOLL_CTL_MOD, socketFd, &eventMask) != 0) {

        closeSocket(1, -1);

    }

}

```

  

**Key Properties**:

  

- **Non-blocking I/O**: Socket set to `O_NONBLOCK` (`ConnectionSocket.cpp:722`)

- **Edge-triggered mode**: `EPOLLET` flag for efficient event handling

- **Partial writes**: Handled via `ByteStream.discard()`

- **Error handling**: Connection closed on send errors

  

#### 3.5.3 Socket Send: System Call

  

**File**: `ConnectionSocket.cpp:1138`

  

The final system call:

  

```cpp

ssize_t sentLength = send(socketFd, buffer->bytes(), remaining, 0);

```

  

**Parameters**:

  

- `socketFd`: File descriptor (created via `socket()`)

- `buffer->bytes()`: Encrypted data buffer

- `remaining`: Number of bytes to send

- `0`: Flags (none)

  

**Return Value**:

  

- `> 0`: Bytes sent (may be less than requested)

- `0`: Connection closed

- `< 0`: Error (errno set)

  

---

  

## 4. Supporting Systems

  

### 4.1 Callback System

  

The callback system enables asynchronous response handling across the Java-C++ boundary.

  

#### 4.1.1 Callback Types

  

**File**: `Defines.h:50-56`

  

```cpp

// Completion callback: Invoked when request completes

typedef std::function<void(TLObject *response, TL_error *error,

                          int32_t networkType, int64_t responseTime,

                          int64_t msgId, int32_t dcId)> onCompleteFunc;

  

// Quick ACK callback: Invoked when server acknowledges receipt

typedef std::function<void()> onQuickAckFunc;

  

// Write callback: Invoked when data written to socket

typedef std::function<void()> onWriteToSocketFunc;

  

// Clear callback: Invoked when request is cleared

typedef std::function<void()> onRequestClearFunc;

```

  

#### 4.1.2 Callback Storage

  

**File**: `Request.h:59-65`

  

Callbacks are stored in the `Request` object:

  

```cpp

class Request {

    onCompleteFunc onCompleteRequestCallback;   // Stored at creation

    onQuickAckFunc onQuickAckCallback;          // Stored at creation

    onWriteToSocketFunc onWriteToSocketCallback;

    onRequestClearFunc onRequestClearCallback;  // Called on destruction

};

```

  

#### 4.1.3 Callback Invocation

  

**File**: `Request.cpp:55-60`, `ConnectionsManager.cpp:1234-1250`

  

```cpp

// Completion callback (Request.cpp:55-60)

void Request::onComplete(TLObject *result, TL_error *error,

                        int32_t networkType, int64_t responseTime,

                        int64_t requestMsgId, int32_t dcId) {

    if (onCompleteRequestCallback != nullptr &&

        (result != nullptr || error != nullptr)) {

        completedSent = true;

        onCompleteRequestCallback(result, error, networkType,

                                 responseTime, requestMsgId, dcId);

    }

}

  

// Response processing (ConnectionsManager.cpp:1227-1250)

else if (typeInfo == typeid(TL_rpc_result)) {

    auto response = (TL_rpc_result *) message;

    int64_t resultMid = response->req_msg_id;

  

    // Find request by message ID

    Request *request = nullptr;

    for (auto iter = runningRequests.begin();

         iter != runningRequests.end(); iter++) {

        if ((*iter)->respondsToMessageId(resultMid)) {

            request = iter->get();

            break;

        }

    }

  

    if (request != nullptr) {

        // Deserialize response

        TLObject *result = request->rawRequest->deserializeResponse(

            response->result.get(), ...);

  

        // Invoke callback

        int32_t dcId = request->datacenterId != DEFAULT_DATACENTER_ID ?

                      request->datacenterId : currentDatacenterId;

        request->onComplete(result, nullptr, connection->currentNetworkType,

                           timeMessage, resultMid, dcId);

  

        // Mark as completed

        request->completed = true;

  

        // Remove from running requests

        runningRequests.erase(iter);

    }

}

```

  

**Thread Safety**:

  

- Callbacks executed on network thread

- Java callbacks require JNI environment (`jniEnv[instanceNum]`)

- Synchronization via mutex for shared state

  

### 4.2 Request Lifecycle

  

Requests progress through multiple states during their lifecycle:

  

```

┌─────────────┐

│  Created    │  sendRequest() called

└──────┬──────┘

       │

       ▼

┌─────────────┐

│   Queued    │  Added to requestsQueue

└──────┬──────┘

       │

       ▼

┌─────────────┐

│  Processing │  processRequestQueue()

│             │  - Wrapped in MTProto

│             │  - Assigned message ID

│             │  - Moved to runningRequests

└──────┬──────┘

       │

       ▼

┌─────────────┐

│   Running   │  In-flight, awaiting response

└──────┬──────┘

       │

       ├─────────────────┐

       │                 │

       ▼                 ▼

┌─────────────┐   ┌─────────────┐

│  Completed  │   │  Cancelled  │

│  (Success)  │   │  (Error)    │

└─────────────┘   └─────────────┘

```

  

**State Transitions**:

  

1. **Created** (`ConnectionsManager.cpp:1917-2200`):

  

   - Request object created with unique token

   - Callbacks registered

   - Added to queue

  

2. **Queued** (`ConnectionsManager.h:217`):

  

   - Stored in `requestsQueue` or `waitingLoginRequests`

   - Awaiting processing

  

3. **Processing** (`ConnectionsManager.cpp:2434-2800`):

  

   - Wrapped in MTProto layer

   - Message ID and sequence assigned

   - Moved to `runningRequests`

  

4. **Running** (`ConnectionsManager.h:218`):

  

   - In-flight on network

   - Awaiting server response

   - Tracked by message ID

  

5. **Completed** (`ConnectionsManager.cpp:1234-1250`):

  

   - Response received and matched

   - Callback invoked

   - Removed from `runningRequests`

  

6. **Cancelled** (`ConnectionsManager.cpp:2069-2168`):

   - Explicit cancellation or error

   - Callback invoked with error

   - Removed from queue/requests

  

### 4.3 Message ID Tracking

  

Message IDs are used to match responses to requests.

  

#### 4.3.1 Message ID Generation

  

**File**: `ConnectionsManager.cpp:131` (approximate)

  

```cpp

int64_t ConnectionsManager::generateMessageId() {

    int64_t time = (int64_t) (getCurrentTimeMonotonicMillis() / 1000);

    int64_t messageId = ((time + timeDifference) << 32) |

                       ((lastOutgoingMessageId++) & 0xFFFFFFFF);

    return messageId;

}

```

  

**Structure**:

  

- Upper 32 bits: Server time (seconds since epoch)

- Lower 32 bits: Monotonic sequence counter

- Time difference: Offset to server time (`timeDifference`)

  

#### 4.3.2 Response Matching

  

**File**: `Request.cpp:41-43`, `ConnectionsManager.cpp:1032-1039`

  

```cpp

// Check if request responds to message ID (Request.cpp:41-43)

bool Request::respondsToMessageId(int64_t id) {

    return messageId == id ||

           std::find(respondsToMessageIds.begin(),

                    respondsToMessageIds.end(), id) !=

           respondsToMessageIds.end();

}

  

// Find request by message ID (ConnectionsManager.cpp:1032-1039)

TLObject *ConnectionsManager::getRequestWithMessageId(int64_t messageId) {

    for (auto & iter : runningRequests) {

        Request *request = iter.get();

        if (request->messageId == messageId) {

            return request->rawRequest;

        }

    }

    return nullptr;

}

```

  

**Matching Process** (`ConnectionsManager.cpp:1227-1250`):

  

1. Server sends `TL_rpc_result` with `req_msg_id`

2. Search `runningRequests` for matching message ID

3. Deserialize response using request's deserializer

4. Invoke completion callback

5. Remove from `runningRequests`

  

### 4.4 Connection Session Management

  

**Location**: `TMessagesProj/jni/tgnet/ConnectionSession.h`

  

Sessions track message processing state per connection:

  

```cpp

class ConnectionSession {

    int64_t sessionId;                        // Unique session ID

    uint32_t nextSeqNo = 0;                   // Next sequence number

    int64_t minProcessedMessageId = 0;        // Minimum processed ID

  

    std::vector<int64_t> processedMessageIds;      // Processed IDs

    std::vector<int64_t> messagesIdsForConfirmation; // Pending ACKs

    std::vector<int64_t> processedSessionChanges;    // Session history

};

```

  

**Key Operations**:

  

1. **Sequence Number Generation** (`ConnectionSession.cpp:24`):

  

   ```cpp

   uint32_t ConnectionSession::generateMessageSeqNo(bool increment) {

       if (increment) {

           nextSeqNo += 2;  // Odd for client, even for server

       }

       return nextSeqNo;

   }

   ```

  

2. **Message ID Processing** (`ConnectionSession.cpp:25-26`):

  

   ```cpp

   int32_t ConnectionSession::isMessageIdProcessed(int64_t messageId) {

       // Check if message ID already processed

       // Returns: 0 = not processed, 1 = processed, 2 = duplicate

   }

  

   void ConnectionSession::addProcessedMessageId(int64_t messageId) {

       processedMessageIds.push_back(messageId);

       // Maintain sliding window

   }

   ```

  

3. **Session Recreation** (`ConnectionSession.cpp:20-21`):

   ```cpp

   void ConnectionSession::recreateSession() {

       genereateNewSessionId();

       processedMessageIds.clear();

       messagesIdsForConfirmation.clear();

   }

   ```

  

**Session Lifecycle**:

  

- Created on connection establishment

- Recreated on session conflicts

- Tracked per connection type and datacenter

  

---

  

## 5. Data Flow Analysis

  

### 5.1 Complete Request-Response Cycle

  

```

┌─────────────────────────────────────────────────────────────────┐

│ 1. Java: sendMessage()                                         │

│    - Create TLRPC.Message                                      │

│    - Build TL_messages_sendMessage                             │

│    - Call performSendMessageRequest()                          │

│    File: SendMessagesHelper.java:3816-4781                     │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ 2. Java: ConnectionsManager.sendRequest() (JNI)                │

│    - Register callbacks                                         │

│    - Schedule task on network thread                           │

│    File: SendMessagesHelper.java:6952                          │

└───────────────────────────┬─────────────────────────────────────┘

                             │ JNI Bridge

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ 3. C++: scheduleTask()                                         │

│    - Add to pendingTasks queue                                 │

│    - Wakeup event loop                                         │

│    File: ConnectionsManager.cpp:309-312                        │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ 4. C++: checkPendingTasks()                                    │

│    - Execute task on network thread                            │

│    - Create Request object                                     │

│    - Add to requestsQueue                                      │

│    File: ConnectionsManager.cpp:187-198                        │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ 5. C++: processRequestQueue()                                  │

│    - Wrap in MTProto layer                                     │

│    - Generate message ID                                       │

│    - Move to runningRequests                                   │

│    File: ConnectionsManager.cpp:2434-2800                      │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ 6. C++: sendMessagesToConnection()                             │

│    - Batch messages (3KB limit)                                │

│    - Create transport data                                     │

│    - Call connection->sendData()                               │

│    File: ConnectionsManager.cpp:2285-2335                      │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ 7. C++: Connection.sendData()                                 │

│    - Add protocol header                                       │

│    - Encrypt with AES-256-CTR                                  │

│    - Call writeBuffer()                                        │

│    File: Connection.cpp:430-622                                │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ 8. C++: ConnectionSocket.writeBuffer()                         │

│    - Append to ByteStream                                      │

│    - Adjust epoll events                                       │

│    File: ConnectionSocket.cpp:1176-1179                        │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ 9. C++: ConnectionSocket.onEvent(EPOLLOUT)                     │

│    - Get data from ByteStream                                   │

│    - Call send() syscall                                        │

│    File: ConnectionSocket.cpp:1138                             │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

                    ┌─────────────────┐

                    │  Network Stack  │

                    │  (Linux Kernel) │

                    └─────────────────┘

                             │

                             ▼

                    [Network Transmission]

                             │

                             ▼

                    [Server Processing]

                             │

                             ▼

                    [Response Received]

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ 10. C++: Connection.onReceivedData()                           │

│     - Decrypt response                                         │

│     - Parse MTProto message                                    │

│     - Call processServerResponse()                             │

│     File: Connection.cpp:71-200 (approximate)                  │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ 11. C++: ConnectionsManager.processServerResponse()           │

│     - Match message ID to request                              │

│     - Deserialize response                                     │

│     - Invoke Request.onComplete()                              │

│     File: ConnectionsManager.cpp:1085-1300                     │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ 12. C++: Request.onComplete()                                  │

│     - Invoke onCompleteRequestCallback                         │

│     - Execute on JNI thread                                    │

│     File: Request.cpp:55-60                                    │

└───────────────────────────┬─────────────────────────────────────┘

                             │ JNI Bridge

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│ 13. Java: Callback Execution                                   │

│     - Process response                                         │

│     - Update UI                                                │

│     - Handle errors                                            │

│     File: SendMessagesHelper.java:6952 (callback)             │

└─────────────────────────────────────────────────────────────────┘

```

  

### 5.2 Message Structure Evolution

  

```

┌─────────────────────────────────────────────────────────────┐

│ Stage 1: Java Object                                         │

│   TLRPC.TL_messages_sendMessage                             │

│   - message: "Hello"                                         │

│   - peer: InputPeerUser                                      │

│   - random_id: 0x1234567890abcdef                           │

│   Size: ~100 bytes (Java object)                            │

└───────────────────────────┬─────────────────────────────────┘

                             │ Serialization

                             ▼

┌─────────────────────────────────────────────────────────────┐

│ Stage 2: Serialized Bytes                                   │

│   NativeByteBuffer                                          │

│   [Constructor ID: 4 bytes]                                │

│   [Fields: variable length]                                 │

│   Size: ~120 bytes                                          │

└───────────────────────────┬─────────────────────────────────┘

                             │ MTProto Wrapping

                             ▼

┌─────────────────────────────────────────────────────────────┐

│ Stage 3: MTProto Message                                    │

│   TL_message                                                │

│   - msg_id: 0x5a1b2c3d4e5f6789 (64-bit)                    │

│   - seqno: 43 (32-bit, odd)                                 │

│   - body: [Serialized request]                              │

│   Size: ~140 bytes                                          │

└───────────────────────────┬─────────────────────────────────┘

                             │ Protocol Header

                             ▼

┌─────────────────────────────────────────────────────────────┐

│ Stage 4: Protocol Packet                                    │

│   [Packet Length: 4 bytes]                                  │

│   [ACK Flag: 1 bit]                                         │

│   [Body: MTProto message]                                   │

│   Size: ~144 bytes                                          │

└───────────────────────────┬─────────────────────────────────┘

                             │ Encryption

                             ▼

┌─────────────────────────────────────────────────────────────┐

│ Stage 5: Encrypted Packet                                   │

│   [Encrypted Header: 4 bytes]                               │

│   [Encrypted Body: ~140 bytes]                              │

│   [Optional Padding: 0-16 bytes]                            │

│   Total: ~144-160 bytes                                      │

└───────────────────────────┬─────────────────────────────────┘

                             │ Socket Write

                             ▼

┌─────────────────────────────────────────────────────────────┐

│ Stage 6: Network Transmission                              │

│   TCP/IP Packet                                             │

│   [IP Header: 20 bytes]                                     │

│   [TCP Header: 20 bytes]                                    │

│   [Encrypted Data: ~144 bytes]                              │

│   Total: ~184 bytes                                         │

└─────────────────────────────────────────────────────────────┘

```

  

---

  

## 6. Performance Characteristics

  

### 6.1 Batching Strategy

  

**File**: `ConnectionsManager.cpp:2293-2300`

  

Messages are batched to reduce network overhead:

  

- **Batch Size**: Up to 3KB (`currentSize >= 3 * 1024`)

- **Benefits**: Reduced packet count, improved throughput

- **Trade-off**: Slight latency increase for batching

  

### 6.2 Connection Pooling

  

**File**: `Datacenter.h:63-70`

  

Multiple connections per datacenter:

  

- **Generic**: 1 connection (`getGenericConnection()`)

- **Media**: 1 connection (`getGenericMediaConnection()`)

- **Download**: 2 connections (`DOWNLOAD_CONNECTIONS_COUNT`)

- **Upload**: 4 connections (`UPLOAD_CONNECTIONS_COUNT`)

- **Proxy**: 4 connections (`PROXY_CONNECTIONS_COUNT`)

  

**Benefits**: Parallel request processing, improved throughput

  

### 6.3 Buffer Management

  

**File**: `BuffersStorage.cpp` (implied)

  

- **Buffer Pooling**: Reused `NativeByteBuffer` objects

- **Memory Efficiency**: Reduced allocations

- **Thread Safety**: Per-thread or synchronized access

  

### 6.4 Epoll Efficiency

  

**File**: `ConnectionSocket.cpp:1186`

  

- **Edge-Triggered Mode**: `EPOLLET` for efficient event handling

- **Single Thread**: All I/O on network thread

- **Minimal Syscalls**: Batch operations where possible

  

---

  

## 7. Error Handling & Resilience

  

### 7.1 Network Errors

  

**File**: `ConnectionSocket.cpp:1138-1141`

  

```cpp

if (sentLength < 0) {

    closeSocket(1, -1);  // Close connection on error

    return;

}

```

  

**Recovery**:

  

- Connection automatically reconnected

- Requests retried from queue

- State preserved across reconnection

  

### 7.2 Server Errors

  

**File**: `ConnectionsManager.cpp:1234-1250`

  

Server errors handled via `TL_error`:

  

```cpp

if (typeInfo == typeid(TL_error)) {

    auto error = (TL_error *) message;

  

    // Handle specific error codes

    if (error->code == 303) {

        // MIGRATE_DC: Switch datacenter

    } else if (error->code == 400) {

        // BAD_REQUEST: Invalid request

    } else if (error->code == 429) {

        // FLOOD_WAIT: Rate limiting

    }

  

    // Invoke callback with error

    request->onComplete(nullptr, error, ...);

}

```

  

### 7.3 Timeout Handling

  

**File**: `ConnectionSocket.cpp:1207-1218`

  

```cpp

bool ConnectionSocket::checkTimeout(int64_t now) {

    if (timeout != 0 && (now - lastEventTime) > (int64_t) timeout * 1000) {

        if (!onConnectedSent || hasPendingRequests()) {

            closeSocket(2, 0);  // Timeout

            return true;

        }

    }

    return false;

}

```

  

**Timeout Values**:

  

- Initial: 12 seconds (`ConnectionSocket.h:61`)

- Adjusted based on connection quality

- Reduced after successful data transfer

  

---

  

## 8. Security Considerations

  

### 8.1 Encryption

  

- **Algorithm**: AES-256-CTR

- **Key Derivation**: From random handshake data

- **Key Rotation**: Per-connection keys

- **Forward Secrecy**: Perfect Forward Secrecy (PFS) enabled

  

### 8.2 Protocol Obfuscation

  

- **Multiple Protocols**: EF/EE/DD/TLS variants

- **Random Padding**: Variable-length padding

- **TLS Handshake**: Additional obfuscation layer

  

### 8.3 Message Authentication

  

- **Message IDs**: Time-based, prevent replay

- **Sequence Numbers**: Ensure ordering

- **Server Salts**: Prevent session hijacking

  

---

  

## 9. Conclusion

  

The Telegram Android send message flow demonstrates a sophisticated multi-layer architecture with:

  

1. **Clear Separation of Concerns**: Each layer has distinct responsibilities

2. **Asynchronous Design**: Callbacks enable non-blocking operations

3. **Efficient Resource Usage**: Batching, pooling, buffer reuse

4. **Robust Error Handling**: Comprehensive error recovery

5. **Security**: End-to-end encryption via MTProto

  

The system successfully balances performance, security, and maintainability across Java and C++ codebases.

  

---

  

## 10. References

  

### 10.1 Source Files

  

**Java Layer**:

  

- `TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java:3816-4785`

- `TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java:6931-7100`

- `TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java:6952`

  

**C++ Network Layer**:

  

- `TMessagesProj/jni/tgnet/ConnectionsManager.cpp:345-362` (ThreadProc)

- `TMessagesProj/jni/tgnet/ConnectionsManager.cpp:309-312` (scheduleTask)

- `TMessagesProj/jni/tgnet/ConnectionsManager.cpp:187-198` (checkPendingTasks)

- `TMessagesProj/jni/tgnet/ConnectionsManager.cpp:2434-2800` (processRequestQueue)

- `TMessagesProj/jni/tgnet/ConnectionsManager.cpp:2285-2335` (sendMessagesToConnection)

- `TMessagesProj/jni/tgnet/ConnectionsManager.cpp:1085-1300` (processServerResponse)

  

**C++ Connection Layer**:

  

- `TMessagesProj/jni/tgnet/Connection.cpp:430-622` (sendData)

- `TMessagesProj/jni/tgnet/ConnectionSession.h:24-41` (Session management)

  

**C++ Socket Layer**:

  

- `TMessagesProj/jni/tgnet/ConnectionSocket.cpp:1176-1179` (writeBuffer)

- `TMessagesProj/jni/tgnet/ConnectionSocket.cpp:1100-1167` (onEvent)

- `TMessagesProj/jni/tgnet/ConnectionSocket.cpp:1138` (send syscall)

- `TMessagesProj/jni/tgnet/ByteStream.cpp:20-86` (Buffer management)

  

**Header Files**:

  

- `TMessagesProj/jni/tgnet/Defines.h:50-66` (Callback types, NetworkMessage)

- `TMessagesProj/jni/tgnet/Request.h:24-81` (Request structure)

- `TMessagesProj/jni/tgnet/ConnectionSession.h:16-42` (Session structure)

- `TMessagesProj/jni/tgnet/Datacenter.h:28-154` (Datacenter structure)

  

### 10.2 Related Documentation

  

- MTProto Protocol Specification

- Telegram API Documentation

- OpenSSL AES-CTR Documentation

- Linux epoll(7) Manual

- POSIX send(2) Manual

  

---

  

_Document generated from TMessagesProj source code analysis_

_Last Updated: 2024_
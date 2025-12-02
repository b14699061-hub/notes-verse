# Telegram Send Message Flow: Deep Dive

  

## Overview

  

This document provides a comprehensive analysis of the send message flow in Telegram Android (TMessagesProj), tracing the journey from user interaction in the Java application layer all the way down to the actual socket send operation in the native C++ network layer.

  

The flow spans multiple architectural layers:

  

1. **UI Layer** (Java) - User interaction and message composition

2. **Application Layer** (Java) - Message preparation and request construction

3. **Network Manager** (C++/JNI) - Request queuing and connection management

4. **Connection Layer** (C++) - Protocol wrapping and encryption

5. **Socket Layer** (C++) - Actual network transmission

  

---

  

## High-Level Flow Diagram

  

```

┌─────────────────────────────────────────────────────────────────┐

│                        UI Layer (Java)                         │

│  ChatActivity → User types message → Send button clicked       │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│                   Application Layer (Java)                      │

│  SendMessagesHelper.sendMessage()                              │

│    ├─ Create TLRPC.Message object                              │

│    ├─ Build TL_messages_sendMessage request                    │

│    └─ Call performSendMessageRequest()                         │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│                    JNI Bridge Layer                            │

│  ConnectionsManager.sendRequest()                               │

│    ├─ Create Request object                                   │

│    ├─ Serialize TLObject to bytes                             │

│    └─ Queue request                                            │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│              Network Manager (C++ - Native Thread)             │

│  ConnectionsManager.processRequestQueue()                      │

│    ├─ Wrap in MTProto layer                                    │

│    ├─ Generate message ID and sequence                        │

│    ├─ Encrypt with AES-256-CTR                                 │

│    └─ Call sendMessagesToConnection()                          │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│                   Connection Layer (C++)                        │

│  Connection.sendData()                                         │

│    ├─ Add protocol header (EF/EE/DD/TLS)                      │

│    ├─ Encrypt packet with AES-CTR                              │

│    └─ Call ConnectionSocket.writeBuffer()                     │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

┌─────────────────────────────────────────────────────────────────┐

│                    Socket Layer (C++)                           │

│  ConnectionSocket.writeBuffer()                                 │

│    ├─ Append to ByteStream queue                               │

│    ├─ Adjust epoll events (EPOLLOUT)                           │

│    └─ On EPOLLOUT event: send() syscall                        │

└───────────────────────────┬─────────────────────────────────────┘

                             │

                             ▼

                    ┌─────────────────┐

                    │  Network Stack  │

                    │  (Linux Kernel) │

                    └─────────────────┘

```

  

---

  

## Layer-by-Layer Breakdown

  

### 1. UI Layer - User Interaction

  

**Location**: `TMessagesProj/src/main/java/org/telegram/ui/ChatActivity.java`

  

The user types a message and clicks send. While the exact UI entry point varies, the flow typically involves:

  

- User input captured in `ChatActivity`

- Message text and metadata collected

- `SendMessagesHelper.sendMessage()` called with message parameters

  

**Key Components**:

  

- `ChatActivity` - Main chat interface

- Message composition UI components

- Input validation and formatting

  

---

  

### 2. Application Layer - Message Preparation

  

**Location**: `TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java`

  

#### Entry Point: `sendMessage(SendMessageParams)`

  

**File**: `SendMessagesHelper.java:3816`

  

This is the main entry point for sending messages. It handles:

  

```java

public void sendMessage(SendMessageParams sendMessageParams) {

    // Extract parameters

    String message = sendMessageParams.message;

    long peer = sendMessageParams.peer;

    // ... other parameters

  

    // Validate and prepare message

    // Create TLRPC.Message object

    // Build appropriate request (TL_messages_sendMessage, TL_messages_sendMedia, etc.)

  

    // Call performSendMessageRequest()

}

```

  

**Key Steps**:

  

1. **Message Object Creation** (lines 3888-4646)

  

   - Creates `TLRPC.Message` object with unique ID

   - Sets message flags, peer information, reply headers

   - Handles different message types (text, media, location, etc.)

   - Stores message in local database

  

2. **Request Construction** (lines 4656-4785)

  

   - For text messages: Creates `TLRPC.TL_messages_sendMessage`

   - For media: Creates `TLRPC.TL_messages_sendMedia`

   - Sets request parameters:

     - `peer` - Target chat/user

     - `message` - Message text

     - `random_id` - Unique identifier

     - `reply_to` - Reply information

     - `entities` - Text formatting entities

     - `schedule_date` - For scheduled messages

  

3. **Request Execution** (line 4781)

   ```java

   performSendMessageRequest(reqSend, newMsgObj, null, null, parentObject, params, scheduleDate != 0);

   ```

  

#### Request Submission: `performSendMessageRequest()`

  

**File**: `SendMessagesHelper.java:6931`

  

This method submits the request to the network layer:

  

```java

protected void performSendMessageRequest(final TLObject req, final MessageObject msgObj, ...) {

    // Store message in sending queue

    putToSendingMessages(newMsgObj, scheduled);

  

    // Submit to ConnectionsManager

    newMsgObj.reqId = getConnectionsManager().sendRequest(req, (response, error) -> {

        // Handle response callback

        if (error == null) {

            // Process successful response

            processSentMessage(newMsgObj.id);

        } else {

            // Handle error

        }

    });

}

```

  

**Key Code Reference**: `SendMessagesHelper.java:6952`

  

```java

newMsgObj.reqId = getConnectionsManager().sendRequest(req, (response, error) -> {

    // Response handling

});

```

  

---

  

### 3. Network Manager - Request Processing

  

**Location**: `TMessagesProj/jni/tgnet/ConnectionsManager.cpp`

  

The `ConnectionsManager` runs on a dedicated native thread and manages all network operations.

  

#### Request Submission: `sendRequest()`

  

**File**: `ConnectionsManager.cpp` (multiple overloads)

  

The Java layer calls `ConnectionsManager.sendRequest()` via JNI:

  

```cpp

int32_t sendRequest(TLObject *object,

                   onCompleteFunc onComplete,

                   onQuickAckFunc onQuickAck,

                   onRequestClearFunc onClear,

                   uint32_t flags,

                   uint32_t datacenterId,

                   ConnectionType connectionType,

                   bool immediate)

```

  

**Internal Flow** (`sendRequestInternal`):

  

1. **Request Creation**:

  

   - Creates `Request` object with unique token

   - Stores callbacks and metadata

   - Determines target datacenter

  

2. **Serialization**:

  

   - Serializes `TLObject` to byte array using TL schema

   - Creates `NetworkMessage` wrapper

  

3. **Queue Management**:

   - Adds to `requestsQueue` or `waitingLoginRequests`

   - Calls `processRequestQueue()` to process

  

#### Request Processing: `processRequestQueue()`

  

**File**: `ConnectionsManager.cpp:2434`

  

This method processes queued requests:

  

1. **Request Selection**:

  

   - Selects requests from queue based on priority

   - Groups requests by datacenter and connection type

  

2. **Message Wrapping**:

  

   - Calls `wrapInLayer()` to wrap in MTProto layer

   - Generates message ID: `generateMessageId()`

   - Generates sequence number

   - Applies server salt for encryption

  

3. **Connection Selection**:

  

   - Gets appropriate `Connection` for datacenter

   - Ensures connection is established

  

4. **Message Transmission**:

   ```cpp

   sendMessagesToConnection(messages, connection, reportAck);

   ```

  

#### Message Transmission: `sendMessagesToConnection()`

  

**File**: `ConnectionsManager.cpp:2285`

  

This method batches and sends messages:

  

```cpp

void ConnectionsManager::sendMessagesToConnection(

    std::vector<std::unique_ptr<NetworkMessage>> &messages,

    Connection *connection,

    bool reportAck) {

  

    // Batch messages (up to 3KB)

    for (uint32_t a = 0; a < count; a++) {

        currentSize += networkMessage->message->bytes;

  

        if (currentSize >= 3 * 1024 || a == count - 1) {

            // Create transport data

            NativeByteBuffer *transportData =

                datacenter->createRequestsData(currentMessages, ...);

  

            // Send via connection

            connection->sendData(transportData, reportAck, true);

        }

    }

}

```

  

**Key Code Reference**: `ConnectionsManager.cpp:2326`

  

```cpp

connection->sendData(transportData, reportAck, true);

```

  

---

  

### 4. Connection Layer - Protocol Wrapping & Encryption

  

**Location**: `TMessagesProj/jni/tgnet/Connection.cpp`

  

The `Connection` class handles MTProto protocol wrapping and encryption.

  

#### Data Transmission: `sendData()`

  

**File**: `Connection.cpp:430`

  

This is where the actual protocol wrapping and encryption happens:

  

```cpp

void Connection::sendData(NativeByteBuffer *buff, bool reportAck, bool encrypted) {

    // 1. Check connection state

    if (connectionState == TcpConnectionStageIdle) {

        connect();

    }

  

    // 2. Determine protocol type (EF/EE/DD/TLS)

    ProtocolType currentProtocolType = ProtocolTypeEE; // or EF, DD, TLS

  

    // 3. Calculate packet length and padding

    uint32_t packetLength = buff->limit() / 4; // For EF protocol

    uint32_t additinalPacketSize = 0; // Random padding

  

    // 4. Handle first packet (handshake)

    if (!firstPacketSent) {

        // Generate 64-byte random header

        // Encrypt key exchange

        // Set up AES keys

        firstPacketSent = true;

    }

  

    // 5. Write protocol header

    if (currentProtocolType == ProtocolTypeEF) {

        if (packetLength < 0x7f) {

            buffer->writeByte((uint8_t) packetLength);

        } else {

            buffer->writeInt32((packetLength << 8) + 0x7f);

        }

    } else {

        buffer->writeInt32(packetLength);

    }

  

    // 6. Encrypt header

    AES_ctr128_encrypt(bytes, bytes, headerSize, &encryptKey, ...);

  

    // 7. Encrypt payload

    buff->rewind();

    AES_ctr128_encrypt(buff->bytes(), buff->bytes(), buff->limit(),

                      &encryptKey, encryptIv, encryptCount, &encryptNum);

  

    // 8. Write to socket

    writeBuffer(buffer);  // Header

    writeBuffer(buff);     // Payload

    if (buffer2 != nullptr) {

        writeBuffer(buffer2);  // Padding

    }

}

```

  

**Key Encryption Details**:

  

1. **AES-256-CTR Encryption**:

  

   - Uses OpenSSL's AES-CTR mode

   - Separate keys for encrypt/decrypt

   - IV (Initialization Vector) managed per connection

  

2. **Protocol Types**:

  

   - **EF**: Original protocol (0xef header)

   - **EE**: Extended protocol (0xee header)

   - **DD**: Datacenter protocol (0xdd header)

   - **TLS**: TLS obfuscation (0xee + TLS handshake)

  

3. **First Packet Handshake**:

   - 64-byte random header

   - Key exchange encrypted with secret

   - Datacenter ID embedded

  

**Key Code Reference**: `Connection.cpp:614-621`

  

```cpp

writeBuffer(buffer);  // Encrypted header

buff->rewind();

AES_ctr128_encrypt(buff->bytes(), buff->bytes(), buff->limit(),

                  &encryptKey, encryptIv, encryptCount, &encryptNum);

writeBuffer(buff);     // Encrypted payload

```

  

---

  

### 5. Socket Layer - Network Transmission

  

**Location**: `TMessagesProj/jni/tgnet/ConnectionSocket.cpp`

  

The `ConnectionSocket` class handles the actual socket operations.

  

#### Buffer Writing: `writeBuffer()`

  

**File**: `ConnectionSocket.cpp:1176`

  

```cpp

void ConnectionSocket::writeBuffer(NativeByteBuffer *buffer) {

    // Append to outgoing byte stream

    outgoingByteStream->append(buffer);

  

    // Adjust epoll events to enable write

    adjustWriteOp();

}

```

  

**ByteStream Management**:

  

The `ByteStream` class (`ByteStream.cpp`) maintains a queue of buffers:

  

- Buffers are appended to `buffersQueue`

- Data is sent when socket becomes writable

- Buffers are discarded after successful send

  

#### Epoll Event Handling: `onEvent()`

  

**File**: `ConnectionSocket.cpp` (around line 1100)

  

When the socket becomes writable (EPOLLOUT event):

  

```cpp

void ConnectionSocket::onEvent(uint32_t events) {

    if (events & EPOLLOUT) {

        // Socket is writable

        if (outgoingByteStream->hasData()) {

            NativeByteBuffer *buffer = ...;

            outgoingByteStream->get(buffer);

  

            uint32_t remaining = buffer->remaining();

  

            if (tlsState != 0) {

                // TLS obfuscation mode

                // Add TLS headers

                // Send with TLS framing

                sentLength = send(socketFd, tempBuffer->bytes,

                                 headersSize + remaining, 0);

            } else {

                // Direct send

                sentLength = send(socketFd, buffer->bytes(), remaining, 0);

            }

  

            if (sentLength > 0) {

                // Update statistics

                delegate->onBytesSent(sentLength, ...);

  

                // Discard sent data

                outgoingByteStream->discard(sentLength);

  

                // Adjust epoll events

                adjustWriteOp();

            }

        }

    }

}

```

  

#### Actual Socket Send

  

**File**: `ConnectionSocket.cpp:1138` (non-TLS) and `1126` (TLS)

  

The final system call:

  

```cpp

// Non-TLS mode

ssize_t sentLength = send(socketFd, buffer->bytes(), remaining, 0);

  

// TLS mode

ssize_t sentLength = send(socketFd, tempBuffer->bytes,

                         headersSize + remaining, 0);

```

  

**Key Details**:

  

1. **Non-blocking Socket**:

  

   - Socket is set to `O_NONBLOCK` mode

   - `send()` may return partial writes

   - Epoll is used to detect when socket is writable

  

2. **Epoll Integration**:

  

   - Socket registered with `epoll_ctl()`

   - `EPOLLOUT` event triggers write

   - Events adjusted via `adjustWriteOp()`

  

3. **Error Handling**:

   - `send()` errors trigger connection close

   - Partial sends handled via `ByteStream.discard()`

   - Connection state tracked

  

**Key Code Reference**: `ConnectionSocket.cpp:1138`

  

```cpp

if ((sentLength = send(socketFd, buffer->bytes(), remaining, 0)) < 0) {

    closeSocket(1, -1);

    return;

} else {

    delegate->onBytesSent((int32_t) sentLength, ...);

    outgoingByteStream->discard((uint32_t) sentLength);

    adjustWriteOp();

}

```

  

---

  

## Data Flow Visualization

  

### Message Structure Evolution

  

```

┌─────────────────────────────────────────────────────────────┐

│ 1. Java Layer: TLRPC.TL_messages_sendMessage                │

│    ├─ message: "Hello"                                      │

│    ├─ peer: InputPeerUser                                   │

│    ├─ random_id: 123456789                                   │

│    └─ flags: 0                                               │

└───────────────────────────┬─────────────────────────────────┘

                             │ Serialization

                             ▼

┌─────────────────────────────────────────────────────────────┐

│ 2. Serialized: NativeByteBuffer                             │

│    [Constructor ID][Fields...]                              │

│    Size: ~100 bytes                                          │

└───────────────────────────┬─────────────────────────────────┘

                             │ MTProto Wrapping

                             ▼

┌─────────────────────────────────────────────────────────────┐

│ 3. MTProto Layer: NetworkMessage                            │

│    ├─ Message ID: 0x5a1b2c3d4e5f6789                       │

│    ├─ Sequence: 42                                          │

│    ├─ Body: [Serialized request]                            │

│    └─ Size: ~120 bytes                                       │

└───────────────────────────┬─────────────────────────────────┘

                             │ Protocol Header

                             ▼

┌─────────────────────────────────────────────────────────────┐

│ 4. Protocol Header (EE mode)                                 │

│    [Packet Length: 4 bytes]                                  │

│    [Encrypted with AES-CTR]                                  │

└───────────────────────────┬─────────────────────────────────┘

                             │ Encryption

                             ▼

┌─────────────────────────────────────────────────────────────┐

│ 5. Encrypted Packet                                          │

│    [Encrypted Header: 4 bytes]                               │

│    [Encrypted Body: ~120 bytes]                             │

│    [Optional Padding: 0-16 bytes]                            │

│    Total: ~140 bytes                                         │

└───────────────────────────┬─────────────────────────────────┘

                             │ Socket Write

                             ▼

┌─────────────────────────────────────────────────────────────┐

│ 6. Network Transmission                                      │

│    send(socketFd, encryptedData, size, 0)                    │

│    → TCP/IP Stack → Internet                                 │

└─────────────────────────────────────────────────────────────┘

```

  

---

  

## Detailed Code Walkthrough

  

### Step 1: Java - Message Creation

  

**File**: `SendMessagesHelper.java:4723-4781`

  

```java

// Create send message request

final TLRPC.TL_messages_sendMessage reqSend = new TLRPC.TL_messages_sendMessage();

  

reqSend.message = message;                    // Message text

reqSend.clear_draft = retryMessageObject == null;

reqSend.silent = newMsg.silent;              // Notification settings

reqSend.peer = sendToPeer;                    // Target chat/user

reqSend.random_id = newMsg.random_id;         // Unique identifier

  

// Set reply information

if (newMsg.reply_to instanceof TLRPC.TL_messageReplyHeader) {

    reqSend.reply_to = createReplyInput((TLRPC.TL_messageReplyHeader) newMsg.reply_to);

    reqSend.flags |= 1;

}

  

// Set entities (formatting)

if (entities != null && !entities.isEmpty()) {

    reqSend.entities = entities;

    reqSend.flags |= 8;

}

  

// Submit request

performSendMessageRequest(reqSend, newMsgObj, null, null, parentObject, params, scheduleDate != 0);

```

  

### Step 2: Java - Request Submission

  

**File**: `SendMessagesHelper.java:6952`

  

```java

// Submit to network layer

newMsgObj.reqId = getConnectionsManager().sendRequest(

    req,                                    // TLObject request

    (response, error) -> {                  // Completion callback

        if (error == null) {

            // Success: Process response

            processSentMessage(newMsgObj.id);

        } else {

            // Error: Handle failure

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

  

### Step 3: C++ - Request Queuing

  

**File**: `ConnectionsManager.cpp` (sendRequestInternal)

  

```cpp

// Create request object

std::unique_ptr<Request> request = std::make_unique<Request>(

    instanceNum,

    ++lastRequestToken,                     // Unique token

    connectionType,

    flags,

    datacenterId,

    onComplete,

    onQuickAck,

    nullptr,

    onClear

);

  

// Serialize TLObject

request->rawRequest = object;

request->rpcRequest = std::unique_ptr<TLObject>(object->deserializeResponse(...));

  

// Add to queue

requestsQueue.push_back(std::move(request));

  

// Process queue

processRequestQueue(connectionType, datacenterId);

```

  

### Step 4: C++ - Message Wrapping

  

**File**: `ConnectionsManager.cpp:2285-2335`

  

```cpp

void ConnectionsManager::sendMessagesToConnection(

    std::vector<std::unique_ptr<NetworkMessage>> &messages,

    Connection *connection,

    bool reportAck) {

  

    uint32_t currentSize = 0;

  

    // Batch messages (up to 3KB)

    for (uint32_t a = 0; a < count; a++) {

        NetworkMessage *networkMessage = messages[a].get();

        currentMessages.push_back(std::move(messages[a]));

        currentSize += networkMessage->message->bytes;

  

        // Send batch when size limit reached

        if (currentSize >= 3 * 1024 || a == count - 1) {

            // Create transport data with MTProto wrapping

            NativeByteBuffer *transportData =

                datacenter->createRequestsData(

                    currentMessages,

                    reportAck ? &quickAckId : nullptr,

                    connection,

                    false

                );

  

            // Send via connection

            connection->sendData(transportData, reportAck, true);

  

            currentSize = 0;

            currentMessages.clear();

        }

    }

}

```

  

### Step 5: C++ - Encryption & Protocol

  

**File**: `Connection.cpp:430-622`

  

```cpp

void Connection::sendData(NativeByteBuffer *buff, bool reportAck, bool encrypted) {

    // Determine protocol type

    ProtocolType currentProtocolType = ProtocolTypeEE;

  

    // Calculate packet length

    uint32_t packetLength = buff->limit() / 4;

  

    // Handle first packet (handshake)

    if (!firstPacketSent) {

        // Generate 64-byte random header

        RAND_bytes(bytes, 64);

  

        // Set protocol marker

        bytes[56] = bytes[57] = bytes[58] = bytes[59] = 0xee;

  

        // Encrypt header with secret

        AES_ctr128_encrypt(bytes, temp, 64, &encryptKey, ...);

  

        firstPacketSent = true;

    }

  

    // Write protocol header

    if (currentProtocolType == ProtocolTypeEF) {

        if (packetLength < 0x7f) {

            buffer->writeByte((uint8_t) packetLength);

        } else {

            buffer->writeInt32((packetLength << 8) + 0x7f);

        }

    } else {

        buffer->writeInt32(packetLength);

    }

  

    // Encrypt header

    AES_ctr128_encrypt(bytes, bytes, headerSize, &encryptKey, ...);

  

    // Encrypt payload

    AES_ctr128_encrypt(buff->bytes(), buff->bytes(), buff->limit(),

                      &encryptKey, encryptIv, encryptCount, &encryptNum);

  

    // Write to socket

    writeBuffer(buffer);  // Encrypted header

    writeBuffer(buff);    // Encrypted payload

}

```

  

### Step 6: C++ - Socket Transmission

  

**File**: `ConnectionSocket.cpp:1138-1148`

  

```cpp

void ConnectionSocket::onEvent(uint32_t events) {

    if (events & EPOLLOUT) {

        // Socket is writable

        if (outgoingByteStream->hasData()) {

            NativeByteBuffer *buffer = ...;

            outgoingByteStream->get(buffer);

  

            uint32_t remaining = buffer->remaining();

  

            // Perform actual send

            ssize_t sentLength = send(socketFd, buffer->bytes(), remaining, 0);

  

            if (sentLength > 0) {

                // Update statistics

                delegate->onBytesSent((int32_t) sentLength, ...);

  

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

```

  

## Key Components Reference

  

### Java Classes

  

| Class                           | File                      | Purpose                    |

| ------------------------------- | ------------------------- | -------------------------- |

| `SendMessagesHelper`            | `SendMessagesHelper.java` | Main message sending logic |

| `ConnectionsManager` (JNI)      | JNI wrapper               | Bridge to native layer     |

| `TLRPC.TL_messages_sendMessage` | Generated                 | Request schema             |

| `MessageObject`                 | `MessageObject.java`      | Message wrapper            |

  

### C++ Classes

  

| Class                | File                     | Purpose                          |

| -------------------- | ------------------------ | -------------------------------- |

| `ConnectionsManager` | `ConnectionsManager.cpp` | Network request management       |

| `Connection`         | `Connection.cpp`         | Protocol wrapping & encryption   |

| `ConnectionSocket`   | `ConnectionSocket.cpp`   | Socket operations                |

| `Request`            | `Request.h/cpp`          | Request metadata                 |

| `ByteStream`         | `ByteStream.cpp`         | Buffer queue management          |

| `Datacenter`         | `Datacenter.cpp`         | Datacenter connection management |

  

---

  

## Encryption & Security

  

### MTProto Encryption

  

1. **AES-256-CTR**:

  

   - Symmetric encryption for all data

   - Counter mode for parallel encryption

   - Separate keys for each direction

  

2. **Key Exchange**:

  

   - First packet contains encrypted key material

   - Uses datacenter secret for additional obfuscation

   - Keys derived from random data in handshake

  

3. **Protocol Obfuscation**:

   - Multiple protocol types (EF/EE/DD/TLS)

   - Random padding to obscure packet sizes

   - TLS-like handshake for additional obfuscation

  

### Security Features

  

- **Message IDs**: Monotonically increasing, time-based

- **Sequence Numbers**: Ensure message ordering

- **Server Salts**: Prevent replay attacks

- **Connection Tokens**: Unique per connection

  

---

  

## Threading Model

  

```

┌─────────────────────────────────────────────────────────────┐

│ UI Thread (Java)                                           │

│  ├─ User interaction                                        │

│  ├─ SendMessagesHelper.sendMessage()                        │

│  └─ JNI call to ConnectionsManager                          │

└───────────────────────────┬─────────────────────────────────┘

                             │ JNI Bridge

                             ▼

┌─────────────────────────────────────────────────────────────┐

│ Network Thread (C++ - pthread)                              │

│  ├─ ConnectionsManager::ThreadProc()                       │

│  ├─ epoll_wait() - Event loop                               │

│  ├─ processRequestQueue()                                   │

│  ├─ Connection::sendData()                                  │

│  └─ ConnectionSocket::onEvent()                             │

└─────────────────────────────────────────────────────────────┘

```

  

**Key Points**:

  

- Network operations run on dedicated native thread

- Epoll-based event loop for I/O

- Thread-safe communication via message queues

- Callbacks executed on appropriate threads

  

---

  

## Error Handling & Retry Logic

  

### Error Scenarios

  

1. **Network Errors**:

  

   - Connection drops trigger reconnection

   - Failed sends retry automatically

   - Timeout handling via `checkTimeout()`

  

2. **Server Errors**:

  

   - Flood wait errors handled with backoff

   - File reference errors trigger re-upload

   - Integrity check failures cause session recreation

  

3. **Application Errors**:

   - Message marked as send error

   - User notified via `NotificationCenter`

   - Retry available via `retrySendMessage()`

  

### Retry Mechanism

  

**File**: `SendMessagesHelper.java:1626`

  

```java

public boolean retrySendMessage(MessageObject messageObject, ...) {

    // Reconstruct message parameters

    // Call sendMessage() again

    // Handle encrypted chat special cases

}

```

  

---

  

## Performance Optimizations

  

1. **Message Batching**:

  

   - Multiple requests combined in single packet (up to 3KB)

   - Reduces network overhead

  

2. **Connection Pooling**:

  

   - Multiple connections per datacenter

   - Separate connections for media/generic requests

  

3. **Buffer Reuse**:

  

   - `BuffersStorage` for efficient memory management

   - NativeByteBuffer pooling

  

4. **Epoll Efficiency**:

   - Edge-triggered mode (EPOLLET)

   - Single thread handles all connections

   - Minimal syscall overhead

  

---

  

## Debugging & Logging

  

### Key Log Points

  

1. **Java Layer**:

  

   ```java

   FileLog.d("send message user_id = " + ...);

   ```

  

2. **C++ Layer**:

   ```cpp

   DEBUG_D("connection(%p) send data", this);

   DEBUG_D("connection(%p) send failed", this);

   ```

  

### Log Locations

  

- `SendMessagesHelper.java:4649` - Message send start

- `Connection.cpp:430` - Data send entry

- `ConnectionSocket.cpp:1138` - Socket send

- `ConnectionsManager.cpp:2285` - Message transmission

  

---

  

## Conclusion

  

The send message flow in Telegram Android is a sophisticated multi-layer system:

  

1. **Java Application Layer** prepares and validates messages

2. **JNI Bridge** connects Java to native code

3. **Network Manager** queues and processes requests

4. **Connection Layer** handles protocol and encryption

5. **Socket Layer** performs actual network transmission

  

Each layer has specific responsibilities, ensuring:

  

- **Reliability**: Error handling and retry logic

- **Security**: End-to-end encryption via MTProto

- **Performance**: Efficient batching and connection management

- **Maintainability**: Clear separation of concerns

  

The architecture demonstrates a well-designed system that balances performance, security, and maintainability across multiple programming languages and system layers.

  

---

  

## References

  

### Key Files

  

- `TMessagesProj/src/main/java/org/telegram/messenger/SendMessagesHelper.java`

- `TMessagesProj/jni/tgnet/ConnectionsManager.cpp`

- `TMessagesProj/jni/tgnet/Connection.cpp`

- `TMessagesProj/jni/tgnet/ConnectionSocket.cpp`

- `TMessagesProj/jni/tgnet/ByteStream.cpp`

- `TMessagesProj/jni/tgnet/Request.h`

  

### Related Documentation

  

- MTProto Protocol Specification

- Telegram API Documentation

- OpenSSL AES-CTR Documentation

- Linux epoll(7) Manual

  

---

  

_Document generated from TMessagesProj source code analysis_
# RPC Result Byte Buffer Flow: C++ to Java Deserialization

  

## Table of Contents

1. [Overview](#overview)

2. [Memory Layouts](#memory-layouts)

3. [Complete RPC Receive Flow](#complete-rpc-receive-flow)

4. [Byte-by-Byte Representation](#byte-by-byte-representation)

5. [Object Memory Structures](#object-memory-structures)

6. [Zero-Copy Mechanism](#zero-copy-mechanism)

  

---

  

## Overview

  

This document provides an exhaustive analysis of how RPC result byte buffers flow from network reception in C++ through to Java deserialization, with precise memory layouts and byte-level details.

  

**Key Principle**: The system uses **DirectByteBuffer** for zero-copy memory sharing between C++ and Java. Both sides access the **same physical memory addresses** without data copying.

  

---

  

## Memory Layouts

  

### 1. NativeByteBuffer (C++ Object)

  

**Location**: `TMessagesProj/jni/tgnet/NativeByteBuffer.h`

  

```cpp

class NativeByteBuffer {

private:

uint8_t *buffer = nullptr; // Raw byte pointer (8 bytes on 64-bit)

bool calculateSizeOnly = false; // 1 byte

bool sliced = false; // 1 byte

uint32_t _position = 0; // 4 bytes

uint32_t _limit = 0; // 4 bytes

uint32_t _capacity = 0; // 4 bytes

bool bufferOwner = true; // 1 byte

#ifdef ANDROID

jobject javaByteBuffer = nullptr; // 8 bytes (pointer to Java object)

#endif

};

```

  

**Memory Layout (64-bit system, with padding)**:

```

Offset Size Field

------ ---- -----

+0x00 8 buffer (uint8_t*) → Points to actual byte data

+0x08 1 calculateSizeOnly

+0x09 1 sliced

+0x0A 6 [padding]

+0x10 4 _position

+0x14 4 _limit

+0x18 4 _capacity

+0x1C 1 bufferOwner

+0x1D 7 [padding]

+0x24 8 javaByteBuffer (jobject) → Points to Java ByteBuffer object

Total: 44 bytes (with alignment)

```

  

**Key Point**: The `buffer` pointer points to memory that is **shared** with Java's `DirectByteBuffer`.

  

---

  

### 2. Java ByteBuffer (DirectByteBuffer)

  

**Location**: `java.nio.DirectByteBuffer` (JVM internal)

  

**Memory Layout**:

```

Native Memory (off-heap):

┌─────────────────────────────────────────┐

│ DirectByteBuffer Internal Structure │

│ (Managed by JVM) │

├─────────────────────────────────────────┤

│ +0x00: address (long) → 0x7f8a1c000000 │ Points to raw bytes

│ +0x08: capacity (int) │

│ +0x0C: limit (int) │

│ +0x10: position (int) │

│ +0x14: mark (int) │

│ +0x18: order (ByteOrder) │

│ ... (other fields) │

└─────────────────────────────────────────┘

│

│ address points to:

▼

┌─────────────────────────────────────────┐

│ Raw Byte Data (off-heap) │

│ Address: 0x7f8a1c000000 │

├─────────────────────────────────────────┤

│ [0x78][0x56][0x34][0x12]... │ ← Actual RPC data

│ [0xAB][0xCD][0xEF][0x01]... │

│ ... │

└─────────────────────────────────────────┘

```

  

**Key Point**: `DirectByteBuffer` stores a **native pointer** (`address`) that points directly to off-heap memory. This memory is accessible from both C++ and Java.

  

---

  

### 3. Java NativeByteBuffer Wrapper

  

**Location**: `TMessagesProj/src/main/java/org/telegram/tgnet/NativeByteBuffer.java`

  

```java

public class NativeByteBuffer extends AbstractSerializedData {

protected long address; // 8 bytes: C++ NativeByteBuffer* pointer

public ByteBuffer buffer; // 8 bytes: Reference to DirectByteBuffer

private boolean justCalc; // 1 byte

private int len; // 4 bytes

public boolean reused = true; // 1 byte

}

```

  

**Memory Layout**:

```

Java Heap Object:

┌─────────────────────────────────────────┐

│ NativeByteBuffer (Java object) │

├─────────────────────────────────────────┤

│ +0x00: address (long) = 0x7f8a1c000000 │ C++ pointer

│ +0x08: buffer (ByteBuffer ref) │ → Points to DirectByteBuffer

│ +0x10: justCalc (boolean) │

│ +0x11: len (int) │

│ +0x15: reused (boolean) │

│ ... (inherited from AbstractSerializedData)

└─────────────────────────────────────────┘

```

  

---

  

### 4. TL_rpc_result (C++ Object)

  

**Location**: `TMessagesProj/jni/tgnet/MTProtoScheme.h`

  

```cpp

class TL_rpc_result : public TLObject {

public:

static const uint32_t constructor = 0xf35c6d01;

int64_t req_msg_id; // 8 bytes

std::unique_ptr<TLObject> result; // 8 bytes (smart pointer)

};

```

  

**Memory Layout**:

```

┌─────────────────────────────────────────┐

│ TL_rpc_result object (heap) │

├─────────────────────────────────────────┤

│ +0x00: vtable pointer (8 bytes) │

│ +0x08: req_msg_id (int64_t, 8 bytes) │

│ +0x10: result (unique_ptr, 8 bytes) │ → Points to TLObject*

│ └─> Points to TL_api_response │

└─────────────────────────────────────────┘

```

  

---

  

### 5. TL_api_response (C++ Object)

  

**Location**: `TMessagesProj/jni/tgnet/MTProtoScheme.h`

  

```cpp

class TL_api_response : public TLObject {

public:

std::unique_ptr<NativeByteBuffer> response; // 8 bytes

};

```

  

**Memory Layout**:

```

┌─────────────────────────────────────────┐

│ TL_api_response object (heap)           │

├─────────────────────────────────────────┤

│ +0x00: vtable pointer (8 bytes)         │

│ +0x08: response (unique_ptr, 8 bytes)   │ → Points to NativeByteBuffer*

│ └─> Points to NativeByteBuffer          │

│ └─> buffer points to raw data           │

└─────────────────────────────────────────┘

```

  

---

  

## Complete RPC Receive Flow

  

### Phase 1: Network Reception & Decryption

  

**Location**: `Connection::onReceivedData()` (`Connection.cpp:71`)

  

```

1. Encrypted data arrives from network socket

┌─────────────────────────────────────┐

│ Encrypted Packet (TCP stream)       │

│ [encrypted bytes...]                │

└─────────────────────────────────────┘

│
▼

2. NativeByteBuffer allocated (from BuffersStorage)

┌─────────────────────────────────────┐

│ NativeByteBuffer* buffer            │

│ buffer->buffer = 0x7f8a1c000000     │

│ buffer->_position = 0               │

│ buffer->_limit = packet_size        │

└─────────────────────────────────────┘

│

▼

3. AES-CTR decryption (in-place)

AES_ctr128_encrypt(

buffer->bytes(), // Input: encrypted

buffer->bytes(), // Output: decrypted (same buffer!)

buffer->limit(), // Length

&decryptKey, // AES key

decryptIv, // Initialization vector

decryptCount, // Counter

&decryptNum // Num bytes processed

);

Memory after decryption:

┌─────────────────────────────────────┐

│ Address: 0x7f8a1c000000 │

│ [0x00][0x00][0x00][0x00] ← keyId │

│ [0x00][0x00][0x00][0x00] │

│ [0x12][0x34][0x56][0x78] ← salt │

│ [0x12][0x34][0x56][0x78] │

│ [0xAB][0xCD][0xEF][0x01] ← msgId │

│ [0xAB][0xCD][0xEF][0x01] │

│ [0x00][0x00][0x00][0x05] ← seqNo │

│ [0x00][0x00][0x01][0x00] ← length │

│ [0xf3][0x5c][0x6d][0x01] ← TL_rpc_result constructor

│ [0x12][0x34][0x56][0x78] ← req_msg_id

│ [0x12][0x34][0x56][0x78] │

│ [0xCC][0xCC][0xCC][0xCC] ← result data...

│ ... │

└─────────────────────────────────────┘

```

  

**Byte Layout of Decrypted MTProto Message**:

```

Offset Size Field Value (Example)

------ ---- ----- -----------------

+0x00 8 auth_key_id (int64) 0x0000000000000000

+0x08 8 message_salt (int64) 0x1234567812345678

+0x10 8 session_id (int64) 0xABCDEF01ABCDEF01

+0x18 8 message_id (int64) 0xABCDEF01ABCDEF01

+0x20 4 message_seq_no (int32) 0x00000005

+0x24 4 message_length (uint32) 0x00000100 (256 bytes)

+0x28 4 constructor (uint32) 0xf35c6d01 (TL_rpc_result)

+0x2C 8 req_msg_id (int64) 0x1234567812345678

+0x34 4 result_constructor 0xCCCCCCCC (or error)

+0x38 N result_data... [actual RPC response bytes]

```

  

---

  

### Phase 2: Packet Parsing

  

**Location**: `Connection::onReceivedData()` (`Connection.cpp:149-254`)

  

```

1. Parse packet length header

ProtocolTypeEF:

- If first byte < 0x7F: length = byte * 4

- If first byte == 0x7F: length = (next 4 bytes >> 8) * 4

Other protocols:

- Read 4 bytes as uint32 (little-endian)

  

2. Extract complete message

buffer->limit(buffer->position() + currentPacketLength);

3. Call ConnectionsManager::onConnectionDataReceived()

onConnectionDataReceived(this, buffer, currentPacketLength);

```

  

---

  

### Phase 3: Message Decryption & Deserialization

  

**Location**: `ConnectionsManager::onConnectionDataReceived()` (`ConnectionsManager.cpp:824`)

  

```

1. Read auth_key_id (8 bytes)

int64_t keyId = data->readInt64(&error);

2. Decrypt message body (if keyId != 0)

datacenter->decryptServerResponse(

keyId,

data->bytes() + mark + 8, // Source: encrypted data

data->bytes() + mark + 24, // Dest: decrypted (in-place)

length - 24, // Length

connection

);

Memory transformation:

BEFORE (encrypted):

┌─────────────────────────────────────┐

│ [keyId][encrypted_body...] │

└─────────────────────────────────────┘

AFTER (decrypted):

┌─────────────────────────────────────┐

│ [keyId][salt][session][msgId] │

│ [seqNo][length][constructor] │

│ [req_msg_id][result_data...] │

└─────────────────────────────────────┘

  

3. Read message header

data->position(mark + 24);

int64_t messageServerSalt = data->readInt64(&error);

int64_t messageSessionId = data->readInt64(&error);

int64_t messageId = data->readInt64(&error);

int32_t messageSeqNo = data->readInt32(&error);

uint32_t messageLength = data->readUint32(&error);

  

4. Deserialize TL_rpc_result

object = TLdeserialize(nullptr, messageLength, data);

This calls:

- data->readUint32() → constructor (0xf35c6d01)

- TL_rpc_result::readParamsEx()

```

  

---

  

### Phase 4: TL_rpc_result Deserialization

  

**Location**: `TL_rpc_result::readParamsEx()` (`MTProtoScheme.cpp:561`)

  

```cpp

void TL_rpc_result::readParamsEx(NativeByteBuffer *stream, uint32_t bytes,

int32_t instanceNum, bool &error) {

// Read req_msg_id (8 bytes)

req_msg_id = stream->readInt64(&error);

// Get the original request to know what type to deserialize

ConnectionsManager &connectionsManager = ConnectionsManager::getInstance(instanceNum);

TLObject *request = connectionsManager.getRequestWithMessageId(req_msg_id);

// Deserialize the result (bytes - 12 because we already read constructor + req_msg_id)

TLObject *object = connectionsManager.TLdeserialize(

request,

bytes - 12, // Remaining bytes after constructor (4) + req_msg_id (8)

stream

);

if (object != nullptr) {

result = std::unique_ptr<TLObject>(object);

} else {

error = true;

}

}

```

  

**Memory State After Deserialization**:

```

┌─────────────────────────────────────────┐

│ TL_rpc_result object                    │

├─────────────────────────────────────────┤

│ req_msg_id = 0x1234567812345678 │

│ result = unique_ptr<TL_api_response> │

│ └─> TL_api_response* │

│ └─> response = unique_ptr<NativeByteBuffer>

│ └─> NativeByteBuffer* │

│ └─> buffer = 0x7f8a1c000234

│ (points to result data in original buffer)

└─────────────────────────────────────────┘

```

  

**Critical Detail**: `TL_api_response::readParamsEx()` creates a **sliced** `NativeByteBuffer` that points to a **subset** of the original buffer:

  

```cpp

void TL_api_response::readParamsEx(NativeByteBuffer *stream, uint32_t bytes,

bool &error) {

// Create new NativeByteBuffer pointing to existing memory

// stream->position() - 4 because constructor was already read

response = std::unique_ptr<NativeByteBuffer>(

new NativeByteBuffer(

stream->bytes() + stream->position() - 4, // Start of result data

bytes // Length

)

);

stream->skip((uint32_t) (bytes - 4));

}

```

  

**Memory Layout After Slicing**:

```

Original buffer (0x7f8a1c000000):

┌─────────────────────────────────────────┐

│ [header...]                             │

│ [0xf3][0x5c][0x6d][0x01] ← constructor  │

│ [0x12][0x34][0x56][0x78] ← req_msg_id   │

│ [0x12][0x34][0x56][0x78]                │

│ [0xCC][0xCC][0xCC][0xCC] ← result start │ ← TL_api_response.response points here

│ [0xDD][0xDD][0xDD][0xDD]                │

│ ...                                     │

└─────────────────────────────────────────┘

▲

│

│ TL_api_response.response->buffer points here

│ (sliced buffer, no copy!)

```

  

---

  

### Phase 5: Request Callback Invocation

  

**Location**: `ConnectionsManager::processServerResponse()` (`ConnectionsManager.cpp:1227-1360`)

  

```

1. Find matching Request object

for (auto iter = runningRequests.begin(); iter != runningRequests.end(); iter++) {

Request *request = iter->get();

if (request->respondsToMessageId(resultMid)) {

// Found matching request

}

}

  

2. Check if request has callback

if (request->onCompleteRequestCallback != nullptr) {

TLObject *result = response->result.get();

// result is TL_api_response*

// result->response is NativeByteBuffer* (sliced buffer)

}

  

3. Handle gzip compression (if needed)

if (typeid(*result) == typeid(TL_gzip_packed)) {

unpacked_data = decompressGZip(...);

// Creates new buffer with decompressed data

}

  

4. Invoke callback

request->onCompleteRequestCallback(

result, // TL_api_response* (contains NativeByteBuffer*)

error, // TL_error* or nullptr

networkType,

responseTime,

requestMsgId,

dcId

);

```

  

---

  

### Phase 6: JNI Callback to Java

  

**Location**: `TgNetWrapper.cpp:108-125` (lambda callback)

  

```cpp

ConnectionsManager::getInstance(instanceNum).sendRequest(request,

([instanceNum, token](TLObject *response, TL_error *error, ...) {

TL_api_response *resp = (TL_api_response *) response;

jlong ptr = 0;

jint errorCode = 0;

jstring errorText = nullptr;

if (resp != nullptr) {

// Extract NativeByteBuffer* pointer

ptr = (jlong) resp->response.get(); // ← C++ pointer as jlong

} else if (error != nullptr) {

errorCode = error->code;

errorText = jniEnv[instanceNum]->NewStringUTF(error->text.c_str());

}

// Call Java method with pointer

jniEnv[instanceNum]->CallStaticVoidMethod(

jclass_ConnectionsManager,

jclass_ConnectionsManager_onRequestComplete,

instanceNum,

token,

ptr, // ← NativeByteBuffer* as jlong (0x7f8a1c000234)

errorCode,

errorText,

networkType,

responseTime,

msgId,

dcId

);

}),

...

);

```

  

**Memory State**:

```

C++ Side:

┌─────────────────────────────────────────┐

│ TL_api_response* resp │

│ └─> response = unique_ptr<NativeByteBuffer>

│ └─> NativeByteBuffer* at 0x7f8a1c000234

│ └─> buffer = 0x7f8a1c000234 (sliced)

│ └─> Points to result data

└─────────────────────────────────────────┘

│

│ ptr = (jlong)0x7f8a1c000234

▼

Java Side receives:

┌─────────────────────────────────────────┐

│ onRequestComplete( │

│ instanceNum, │

│ token, │

│ 0x7f8a1c000234L, ← C++ pointer │

│ errorCode, │

│ errorText, │

│ ... │

│ ) │

└─────────────────────────────────────────┘

```

  

---

  

### Phase 7: Java Receives Pointer

  

**Location**: `ConnectionsManager.java:426` (onRequestComplete)

  

```java

public static void onRequestComplete(int instanceNum, int token, long address,

int errorCode, String errorText,

int networkType, long responseTime,

long msgId, int dcId) {

try {

// address = 0x7f8a1c000234L (C++ NativeByteBuffer*)

if (address != 0) {

// Wrap the C++ pointer

NativeByteBuffer buff = NativeByteBuffer.wrap(address);

// buff.address = 0x7f8a1c000234L

// buff.buffer = DirectByteBuffer (from native_getJavaByteBuffer)

// Deserialize the response

int constructor = buff.readInt32(true);

TLObject response = TLClassStore.Instance().TLdeserialize(

buff,

constructor,

true

);

// Invoke user callback

RequestCallbacks callbacks = requestCallbacks.get(token);

if (callbacks != null && callbacks.onComplete != null) {

callbacks.onComplete.run(response, null);

}

}

} catch (Exception e) {

FileLog.e(e);

}

}

```

  

---

  

### Phase 8: Java NativeByteBuffer.wrap()

  

**Location**: `NativeByteBuffer.java:26`

  

```java

public static NativeByteBuffer wrap(long address) {

if (address != 0) {

LinkedList<NativeByteBuffer> queue = addressWrappers.get();

NativeByteBuffer result = queue.poll();

if (result == null) {

result = new NativeByteBuffer(0, true);

}

// Store C++ pointer

result.address = address; // 0x7f8a1c000234L

// Get Java ByteBuffer from C++ NativeByteBuffer

result.buffer = native_getJavaByteBuffer(address);

// ↑ JNI call: returns DirectByteBuffer

// Sync position/limit from C++

result.buffer.limit(native_limit(address));

int position = native_position(address);

if (position <= result.buffer.limit()) {

result.buffer.position(position);

}

result.buffer.order(ByteOrder.LITTLE_ENDIAN);

result.reused = false;

return result;

}

return null;

}

```

  

**JNI Implementation** (`TgNetWrapper.cpp:64`):

```cpp

jobject getJavaByteBuffer(JNIEnv *env, jclass c, jlong address) {

NativeByteBuffer *buffer = (NativeByteBuffer *) (intptr_t) address;

if (buffer == nullptr) {

return nullptr;

}

return buffer->getJavaByteBuffer(); // Returns DirectByteBuffer

}

```

  

**NativeByteBuffer::getJavaByteBuffer()** (`NativeByteBuffer.cpp:691`):

```cpp

jobject NativeByteBuffer::getJavaByteBuffer() {

if (javaByteBuffer == nullptr && javaVm != nullptr) {

JNIEnv *env = 0;

javaVm->GetEnv((void **) &env, JNI_VERSION_1_6);

// Create DirectByteBuffer wrapping existing C++ memory

javaByteBuffer = env->NewDirectByteBuffer(buffer, _capacity);

// ↑ buffer = 0x7f8a1c000234 (C++ pointer)

// _capacity = size of sliced buffer

// Make global reference

jobject globalRef = env->NewGlobalRef(javaByteBuffer);

env->DeleteLocalRef(javaByteBuffer);

javaByteBuffer = globalRef;

}

return javaByteBuffer;

}

```

  

**Memory State After wrap()**:

```

Java Heap:

┌─────────────────────────────────────────┐

│ NativeByteBuffer (Java object) │

├─────────────────────────────────────────┤

│ address = 0x7f8a1c000234L │ ← C++ pointer

│ buffer = DirectByteBuffer (reference) │

│ └─> DirectByteBuffer (JVM object) │

│ ├─ address = 0x7f8a1c000234 │ ← Points to same memory!

│ ├─ capacity = result_size │

│ ├─ limit = result_size │

│ └─ position = 0 │

└─────────────────────────────────────────┘

│

│ DirectByteBuffer.address points to:

▼

Native Memory (shared):

┌─────────────────────────────────────────┐

│ Address: 0x7f8a1c000234 │

│ [0xCC][0xCC][0xCC][0xCC] ← result data │

│ [0xDD][0xDD][0xDD][0xDD] │

│ ... │

└─────────────────────────────────────────┘

▲

│

│ C++ NativeByteBuffer.buffer also points here

│ (same memory, zero-copy!)

```

  

---

  

### Phase 9: Java Deserialization

  

**Location**: `TLClassStore.java:53` and `TLObject` subclasses

  

```java

// In onRequestComplete():

int constructor = buff.readInt32(true);

// Reads from DirectByteBuffer at position 0

// Returns: 0xCCCCCCCC (result constructor)

  

TLObject response = TLClassStore.Instance().TLdeserialize(

buff,

constructor,

true

);

  

// TLClassStore.TLdeserialize():

public TLObject TLdeserialize(NativeByteBuffer stream, int constructor,

boolean exception) {

Class objClass = classStore.get(constructor);

if (objClass != null) {

// Instantiate Java object

response = (TLObject) objClass.newInstance();

// Deserialize fields from buffer

response.readParams(stream, exception);

// ↑ Reads from DirectByteBuffer

// Which reads from native memory at 0x7f8a1c000234

return response;

}

return null;

}

```

  

**Example: Deserializing a User object**:

```java

// In TLRPC.TL_user.readParams():

public void readParams(AbstractSerializedData stream, boolean exception) {

flags = stream.readInt32(exception); // Reads 4 bytes

id = stream.readInt64(exception); // Reads 8 bytes

access_hash = stream.readInt64(exception); // Reads 8 bytes

first_name = stream.readString(exception); // Reads string

last_name = stream.readString(exception); // Reads string

// ... more fields

}

```

  

**Byte-by-Byte Reading**:

```

NativeByteBuffer.readInt32():

1. buffer.getInt() → DirectByteBuffer.getInt()

2. DirectByteBuffer reads from native memory:

- Reads 4 bytes at address 0x7f8a1c000234

- Returns int (little-endian)

Memory read:

┌─────────────────────────────────────────┐

│ 0x7f8a1c000234: [0x78][0x56][0x34][0x12]│

│ │

│ Returns: 0x12345678 (little-endian) │

└─────────────────────────────────────────┘

```

  

---

  

## Byte-by-Byte Representation

  

### Complete RPC Response Packet

  

**Example: RPC result for `users.getUsers` returning a User object**

  

```

Network Packet (encrypted, before decryption):

┌─────────────────────────────────────────┐

│ [AES-CTR encrypted bytes...] │

│ Length: variable │

└─────────────────────────────────────────┘

  

After AES-CTR decryption:

┌─────────────────────────────────────────┐

│ Offset Bytes Description │

├─────────────────────────────────────────┤

│ 0x00 8 auth_key_id │

│ [0x00][0x00][0x00][0x00] │

│ [0x00][0x00][0x00][0x00] │

│ 0x08 8 message_salt │

│ [0x12][0x34][0x56][0x78] │

│ [0x12][0x34][0x56][0x78] │

│ 0x10 8 session_id │

│ [0xAB][0xCD][0xEF][0x01] │

│ [0xAB][0xCD][0xEF][0x01] │

│ 0x18 8 message_id │

│ [0xAB][0xCD][0xEF][0x01] │

│ [0xAB][0xCD][0xEF][0x01] │

│ 0x20 4 message_seq_no │

│ [0x00][0x00][0x00][0x05] │

│ 0x24 4 message_length │

│ [0x00][0x00][0x01][0x00] (256) │

│ 0x28 4 TL_rpc_result constructor│

│ [0xf3][0x5c][0x6d][0x01] │

│ 0x2C 8 req_msg_id │

│ [0x12][0x34][0x56][0x78] │

│ [0x12][0x34][0x56][0x78] │

│ 0x34 4 TL_api_response constructor│

│ [0xCC][0xCC][0xCC][0xCC] │

│ 0x38 4 User constructor │

│ [0xd3][0xbc][0x4b][0x7a] │

│ 0x3C 4 flags │

│ [0x00][0x00][0x00][0x01] │

│ 0x40 8 id │

│ [0x78][0x56][0x34][0x12] │

│ [0x78][0x56][0x34][0x12] │

│ 0x48 8 access_hash │

│ [0xAB][0xCD][0xEF][0x01] │

│ [0xAB][0xCD][0xEF][0x01] │

│ 0x50 1 first_name length │

│ [0x05] │

│ 0x51 5 first_name "John" │

│ [0x4A][0x6F][0x68][0x6E][0x00] │

│ 0x56 3 padding (to 4-byte) │

│ [0x00][0x00][0x00] │

│ ... ... more fields... │

└─────────────────────────────────────────┘

```

  

**After Slicing (TL_api_response.response)**:

```

NativeByteBuffer* at 0x7f8a1c000234:

┌─────────────────────────────────────────┐

│ [0xCC][0xCC][0xCC][0xCC] ← constructor │

│ [0xd3][0xbc][0x4b][0x7a] ← User │

│ [0x00][0x00][0x00][0x01] ← flags │

│ [0x78][0x56][0x34][0x12] ← id (low) │

│ [0x78][0x56][0x34][0x12] ← id (high) │

│ ... │

└─────────────────────────────────────────┘

```

  

---

  

## Object Memory Structures

  

### Complete Memory Map (64-bit System)

  

```

┌─────────────────────────────────────────────────────────────┐

│ C++ Heap │

├─────────────────────────────────────────────────────────────┤

│ │

│ TL_rpc_result* at 0x7f8a1c000100 │

│ ┌─────────────────────────────────────────────────────────┐│

│ │ vtable*: 0x7f8a1c001000 ││

│ │ req_msg_id: 0x1234567812345678 ││

│ │ result: unique_ptr → 0x7f8a1c000200 ││

│ └─────────────────────────────────────────────────────────┘│

│ │

│ TL_api_response* at 0x7f8a1c000200 │

│ ┌─────────────────────────────────────────────────────────┐│

│ │ vtable*: 0x7f8a1c001100 ││

│ │ response: unique_ptr → 0x7f8a1c000234 ││

│ └─────────────────────────────────────────────────────────┘│

│ │

│ NativeByteBuffer* at 0x7f8a1c000234 │

│ ┌─────────────────────────────────────────────────────────┐│

│ │ buffer: 0x7f8a1c000234 (sliced, points to itself!) ││

│ │ _position: 0 ││

│ │ _limit: 200 ││

│ │ _capacity: 200 ││

│ │ javaByteBuffer: 0x7f8a1c000300 (jobject) ││

│ └─────────────────────────────────────────────────────────┘│

│ │

└─────────────────────────────────────────────────────────────┘

│

│ NativeByteBuffer.buffer points to:

▼

┌─────────────────────────────────────────────────────────────┐

│ Native Memory (off-heap, shared) │

├─────────────────────────────────────────────────────────────┤

│ Address: 0x7f8a1c000234 │

│ ┌─────────────────────────────────────────────────────────┐│

│ │ [0xCC][0xCC][0xCC][0xCC] ← TL_api_response constructor ││

│ │ [0xd3][0xbc][0x4b][0x7a] ← User constructor ││

│ │ [0x00][0x00][0x00][0x01] ← flags ││

│ │ [0x78][0x56][0x34][0x12] ← id (low 32 bits) ││

│ │ [0x78][0x56][0x34][0x12] ← id (high 32 bits) ││

│ │ [0xAB][0xCD][0xEF][0x01] ← access_hash (low) ││

│ │ [0xAB][0xCD][0xEF][0x01] ← access_hash (high) ││

│ │ [0x05] ← first_name length ││

│ │ [0x4A][0x6F][0x68][0x6E] ← "John" ││

│ │ [0x00][0x00][0x00] ← padding ││

│ │ ... ← more fields ││

│ └─────────────────────────────────────────────────────────┘│

└─────────────────────────────────────────────────────────────┘

▲

│

│ DirectByteBuffer.address points here

│

┌─────────────────────────────────────────────────────────────┐

│ Java Heap │

├─────────────────────────────────────────────────────────────┤

│ │

│ DirectByteBuffer (JVM internal) at 0x7f8a1c000300 │

│ ┌─────────────────────────────────────────────────────────┐│

│ │ address: 0x7f8a1c000234 ← Points to native memory! ││

│ │ capacity: 200 ││

│ │ limit: 200 ││

│ │ position: 0 ││

│ │ order: LITTLE_ENDIAN ││

│ └─────────────────────────────────────────────────────────┘│

│ │

│ NativeByteBuffer (Java) at 0x7f8a1c000400 │

│ ┌─────────────────────────────────────────────────────────┐│

│ │ address: 0x7f8a1c000234L ← C++ pointer ││

│ │ buffer: → DirectByteBuffer (reference) ││

│ │ reused: false ││

│ └─────────────────────────────────────────────────────────┘│

│ │

│ TLRPC.TL_user (Java) at 0x7f8a1c000500 │

│ ┌─────────────────────────────────────────────────────────┐│

│ │ id: 0x1234567812345678L ││

│ │ access_hash: 0xABCDEF01ABCDEF01L ││

│ │ first_name: "John" (String reference) ││

│ │ last_name: null ││

│ │ ... ││

│ └─────────────────────────────────────────────────────────┘│

│ │

└─────────────────────────────────────────────────────────────┘

```

  

---

  

## Zero-Copy Mechanism

  

### How Zero-Copy Works

  

1. **Single Memory Allocation**:

- `DirectByteBuffer.allocateDirect(size)` allocates native memory

- C++ gets pointer via `GetDirectBufferAddress()`

- **Same physical memory** accessed by both

  

2. **Pointer Passing**:

- C++ passes `NativeByteBuffer*` as `jlong` (pointer address)

- Java receives pointer and calls `native_getJavaByteBuffer()`

- Returns existing `DirectByteBuffer` or creates new one wrapping same memory

  

3. **No Data Copying**:

- C++ writes: `buffer[0] = 0x78;` → writes to native memory

- Java reads: `buffer.get()` → reads from **same** native memory

- **Zero bytes copied**

  

### Memory Access Pattern

  

```

C++ Side:

┌─────────────────────────────────────────┐

│ uint8_t *buffer = 0x7f8a1c000234; │

│ buffer[0] = 0x78; // Write │

│ uint8_t val = buffer[0]; // Read │

└─────────────────────────────────────────┘

│

│ Direct memory access

▼

┌─────────────────────────────────────────┐

│ Native Memory: 0x7f8a1c000234 │

│ [0x78] ← Written by C++, read by Java │

└─────────────────────────────────────────┘

▲

│ Direct memory access

│

Java Side:

┌─────────────────────────────────────────┐

│ DirectByteBuffer buffer; │

│ buffer.address = 0x7f8a1c000234; │

│ buffer.put((byte)0x78); // Write │

│ byte val = buffer.get(); // Read │

└─────────────────────────────────────────┘

```

  

### Performance Benefits

  

- **No memcpy()**: Data never copied between C++ and Java

- **Low Latency**: Direct memory access, no serialization overhead

- **Memory Efficient**: Single allocation, shared between languages

- **Cache Friendly**: Same memory location accessed by both sides

  

---

  

## Summary

  

The RPC result byte buffer flow uses a sophisticated zero-copy mechanism:

  

1. **Network → C++**: Encrypted data decrypted in-place into `NativeByteBuffer`

2. **C++ Deserialization**: `TL_rpc_result` → `TL_api_response` → `NativeByteBuffer*` (sliced)

3. **Pointer Passing**: C++ `NativeByteBuffer*` passed as `jlong` to Java

4. **Java Wrapping**: Java creates `DirectByteBuffer` wrapping **same memory**

5. **Java Deserialization**: Reads directly from native memory via `DirectByteBuffer`

  

**Key Insight**: The entire flow maintains **one physical memory allocation** that is accessed by both C++ and Java through different abstractions (`uint8_t*` vs `DirectByteBuffer`), achieving true zero-copy data transfer.
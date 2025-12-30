
## How C++ Byte Buffers Convert to Java Objects - The Bits & Bytes

### The Core Mechanism: DirectByteBuffer (Zero-Copy)

The project uses Java's `DirectByteBuffer` to share memory between C++ and Java without copying.

### Step-by-Step Flow:

#### 1. **C++ Side - Buffer Creation** (`NativeByteBuffer.cpp:19-36`)

```cpp
NativeByteBuffer::NativeByteBuffer(uint32_t size) {
    // On Android, allocate a Java DirectByteBuffer
    javaByteBuffer = env->CallStaticObjectMethod(
        jclass_ByteBuffer, 
        jclass_ByteBuffer_allocateDirect,  // ByteBuffer.allocateDirect(size)
        size
    );
    
    // Get the DIRECT POINTER to the Java ByteBuffer's memory
    buffer = (uint8_t *) env->GetDirectBufferAddress(javaByteBuffer);
    // ↑ This is the KEY: buffer points to the SAME memory Java sees!
}
```

What happens:
- `ByteBuffer.allocateDirect(size)` allocates native memory (off-heap)
- `GetDirectBufferAddress()` returns a C++ pointer to that same memory
- No copy: C++ and Java share the same bytes

#### 2. **C++ Writes Data Directly** (`NativeByteBuffer.cpp:165-178`)

```cpp
void NativeByteBuffer::writeInt32(int32_t x, bool *error) {
    // Write directly to the shared memory
    buffer[_position++] = (uint8_t) x;           // Byte 0: LSB
    buffer[_position++] = (uint8_t) (x >> 8);    // Byte 1
    buffer[_position++] = (uint8_t) (x >> 16);   // Byte 2
    buffer[_position++] = (uint8_t) (x >> 24);   // Byte 3: MSB
}
```

Byte layout (little-endian):
- `int32_t x = 0x12345678`
- Memory: `[0x78, 0x56, 0x34, 0x12]`

#### 3. **C++ Passes Pointer to Java** (`TgNetWrapper.cpp:340-343`)

```cpp
void onUnparsedMessageReceived(int64_t reqMessageId, NativeByteBuffer *buffer, ...) {
    // Pass the C++ pointer as a jlong (64-bit integer)
    jniEnv[instanceNum]->CallStaticVoidMethod(
        jclass_ConnectionsManager, 
        jclass_ConnectionsManager_onUnparsedMessageReceived, 
        (jlong) (intptr_t) buffer,  // ← Pointer address passed as long
        instanceNum, 
        reqMessageId
    );
}
```

What's passed:
- The C++ object pointer (e.g., `0x7f8a1c000000`) as a `jlong`
- No data is copied

#### 4. **Java Receives Pointer** (`ConnectionsManager.java:757-759`)

```java
public static void onUnparsedMessageReceived(long address, ...) {
    // Wrap the pointer address
    NativeByteBuffer buff = NativeByteBuffer.wrap(address);
    // address is the C++ pointer: 0x7f8a1c000000
}
```

#### 5. **Java Gets the ByteBuffer Object** (`NativeByteBuffer.java:26-42`)

```java
public static NativeByteBuffer wrap(long address) {
    result.address = address;  // Store the C++ pointer
    
    // Call native method to get the Java ByteBuffer object
    result.buffer = native_getJavaByteBuffer(address);
    // ↑ This returns the Java ByteBuffer that wraps the SAME memory
    
    // Sync position/limit from C++ side
    result.buffer.limit(native_limit(address));
    result.buffer.position(native_position(address));
    
    result.buffer.order(ByteOrder.LITTLE_ENDIAN);  // Match C++ endianness
    return result;
}
```

#### 6. **JNI Bridge Returns ByteBuffer** (`TgNetWrapper.cpp:64-70`)

```cpp
jobject getJavaByteBuffer(JNIEnv *env, jclass c, jlong address) {
    NativeByteBuffer *buffer = (NativeByteBuffer *) (intptr_t) address;
    return buffer->getJavaByteBuffer();  // Returns the Java ByteBuffer
}
```

#### 7. **C++ Returns Existing or Creates New ByteBuffer** (`NativeByteBuffer.cpp:691-709`)

```cpp
jobject NativeByteBuffer::getJavaByteBuffer() {
    if (javaByteBuffer == nullptr && javaVm != nullptr) {
        // If we have C++ memory but no Java ByteBuffer yet, create one
        javaByteBuffer = env->NewDirectByteBuffer(buffer, _capacity);
        // ↑ Creates Java ByteBuffer wrapping the EXISTING C++ memory
    }
    return javaByteBuffer;  // Return the Java object
}
```

Two scenarios:
- If created from Java: `javaByteBuffer` already exists (from `allocateDirect`)
- If created from C++ memory: `NewDirectByteBuffer()` wraps the existing memory

### The Memory Layout:

```
┌─────────────────────────────────────────┐
│  Native Memory (off-heap)               │
│  Address: 0x7f8a1c000000                 │
│  ┌───────────────────────────────────┐  │
│  │ [0x78][0x56][0x34][0x12]...      │  │ ← Raw bytes
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
         ▲                    ▲
         │                    │
    C++ pointer          Java ByteBuffer
    (uint8_t*)          (DirectByteBuffer)
    buffer              javaByteBuffer
```

### Key Points:

1. Zero-copy: C++ and Java access the same memory
2. DirectByteBuffer: Allocated off-heap, accessible from native code
3. Pointer passing: C++ passes a pointer address (`jlong`), not data
4. Endianness: Both sides use little-endian
5. Synchronization: Position/limit are synced via JNI calls

### Example: Reading an int32

**C++ writes:**
```cpp
buffer[0] = 0x78;  // Memory address 0x7f8a1c000000
buffer[1] = 0x56;  // Memory address 0x7f8a1c000001
buffer[2] = 0x34;  // Memory address 0x7f8a1c000002
buffer[3] = 0x12;  // Memory address 0x7f8a1c000003
```

**Java reads:**
```java
int value = buffer.getInt();  // Reads from SAME memory addresses
// Result: 0x12345678
```


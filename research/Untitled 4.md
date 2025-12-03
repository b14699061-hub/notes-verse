# Buffer Flow Analysis: `Connection::onReceivedData()`

## 🔍 Overview
This function handles incoming encrypted network data, manages packet fragmentation/reassembly, and processes complete packets. Let me map out the complete buffer handling flow:

---

## 📊 Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    ENTRY: onReceivedData()                  │
│                   Input: NativeByteBuffer *buffer           │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 1: DECRYPT INCOMING DATA                              │
│  • AES_ctr128_encrypt(buffer->bytes(), ...)                 │
│  • Decrypts buffer in-place                                 │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 2: HANDLE FRAGMENTED DATA (restOfTheData != nullptr)  │
└──────────────────────────┬──────────────────────────────────┘
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
    ┌─────────────────────┐  ┌──────────────────────┐
    │ Case A:             │  │ Case B:              │
    │ lastPacketLength==0 │  │ lastPacketLength > 0 │
    └─────────┬───────────┘  └──────────┬───────────┘
              │                         │
              ▼                         ▼
    [Merge with previous    [Complete partial packet]
     partial header data]              │
              │                         │
              ▼                         ▼
    Does it fit in         Append up to lastPacketLength
    existing buffer?                    │
         │    │                         ▼
      YES│    │NO              Is packet complete?
         │    │                    │         │
         │    │                  YES│        │NO
         │    │                    │         │
         │    │                    ▼         ▼
         │    │          Set parseLaterBuffer  Return
         │    │          (if data remains)    (wait for more)
         │    │                    │
         │    ▼                    │
         │  Allocate new          │
         │  larger buffer         │
         │  Copy both             │
         │  Reuse old             │
         └────┴────────────────────┘
                │
                ▼
        buffer = merged buffer
                │
                └────────────────────────────┐
                                             │
                                             ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 3: MAIN PACKET PARSING LOOP                           │
│  while (buffer->hasRemaining())                             │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 3A: READ PACKET LENGTH                                │
│  Protocol-specific length extraction:                       │
│                                                              │
│  ProtocolTypeEF:                                            │
│    • First byte != 0x7f → length = byte * 4                 │
│    • First byte == 0x7f → read 4 bytes, length = int32 * 4  │
│    • Check for Quick ACK (bit 7 set)                        │
│                                                              │
│  Other protocols:                                           │
│    • Read 4 bytes (uint32)                                  │
│    • Check for Quick ACK (bit 31 set)                       │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 3B: VALIDATE PACKET LENGTH                            │
│  • Check alignment (must be multiple of 4)                  │
│  • Check max size (≤ 2MB)                                   │
│  • Invalid? → reconnect() and break                         │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 3C: CHECK DATA AVAILABILITY                           │
└──────────────────────────┬──────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
  ┌──────────┐      ┌──────────┐      ┌──────────────┐
  │ buffer < │      │ buffer = │      │ buffer >     │
  │ packet   │      │ packet   │      │ packet       │
  └────┬─────┘      └────┬─────┘      └────┬─────────┘
       │                 │                  │
       ▼                 │                  │
 FRAGMENTED              │                  │
 Save to restOfTheData   │                  │
       │                 │                  │
       ▼                 │                  │
 Allocate/reuse          │                  │
 restOfTheData buffer    │                  │
       │                 │                  │
 Write remaining bytes   │                  │
       │                 │                  │
 Set lastPacketLength    │                  │
       │                 │                  │
 BREAK (wait for more)   │                  │
       │                 │                  │
       └─────────────────┴──────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 3D: PROCESS COMPLETE PACKET                           │
│  • Set buffer limit to current position + packetLength      │
│  • Call onConnectionDataReceived(buffer, packetLength)      │
│  • Check if connection was reset (generation changed)       │
│  • Restore buffer limit                                     │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 3E: CLEANUP restOfTheData                             │
│  • If packet complete → mark for reuse (reuseLater)         │
│  • If partial → compact buffer for next iteration           │
│  • If parseLaterBuffer exists → switch to it                │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
                    Loop back to 3A
                    (if data remains)
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  STEP 4: FINAL CLEANUP                                      │
│  • Reuse any marked buffers (reuseLater->reuse())           │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔑 Key Buffer Variables

| Variable | Purpose |
|----------|---------|
| **`buffer`** | Input buffer (current incoming data) |
| **`restOfTheData`** | Persistent buffer holding incomplete packet data from previous calls |
| **`parseLaterBuffer`** | Temporary pointer to remaining data after completing a packet from `restOfTheData` |
| **`reuseLater`** | Marks buffers for memory recycling at function end |
| **`lastPacketLength`** | Expected total length of incomplete packet being assembled |

---

## 🎯 Critical Buffer Handling Scenarios

### **Scenario 1: First Call with Partial Header**
```
Input: [0x7f, 0xAB] (only 2 bytes, need 4 for length)
Action: Copy to restOfTheData, set lastPacketLength=0, return
```

### **Scenario 2: Continuation with Header Completion**
```
Previous: restOfTheData=[0x7f, 0xAB] (lastPacketLength=0)
Input: [0xCD, 0xEF, ...payload...]
Action: Merge buffers, parse length, continue processing
```

### **Scenario 3: Partial Payload**
```
Parsed length: 1024 bytes
Available: 512 bytes
Action: Save to restOfTheData, set lastPacketLength=1024, return
```

### **Scenario 4: Completion + Extra Data**
```
restOfTheData: 512 bytes (expecting 1024 total)
Input: 600 bytes
Action: 
  - Complete packet (use first 512 bytes)
  - Set parseLaterBuffer to remaining 88 bytes
  - Process complete packet
  - Loop with parseLaterBuffer as new input
```

### **Scenario 5: Buffer Resize**
```
restOfTheData capacity: 100 bytes
Need to append: 200 bytes
Action:
  - Allocate new 300-byte buffer
  - Copy restOfTheData content
  - Append new data
  - Reuse old buffer
```

---

## ⚠️ Edge Cases & Safety Checks

1. **Insufficient Header Data**
   - Less than 4 bytes remaining
   - Creates/reuses `restOfTheData` buffer
   - Breaks loop, waits for next call

2. **Invalid Packet Length**
   - Not multiple of 4 (for certain protocols)
   - Exceeds 2MB
   - Triggers `reconnect()` and aborts processing

3. **Quick ACK Packets**
   - Special control packets (bit 7/31 set)
   - Processed immediately without buffering
   - Continue to next packet in loop

4. **Connection Reset During Processing**
   - Checks `generation` counter before/after packet processing
   - Breaks loop if connection was reset

5. **Buffer Exhaustion**
   - Uses `BuffersStorage` pool for efficient allocation
   - Reuses buffers via `reuse()` method

---

## 🔄 Buffer Lifecycle

```
┌─────────────────────┐
│ BuffersStorage Pool │
└──────────┬──────────┘
           │
           ▼ getFreeBuffer()
    ┌─────────────┐
    │ restOfTheData│◄──── Persistent across calls
    └──────┬──────┘
           │
           ▼ Packet complete
    ┌─────────────┐
    │ reuseLater  │
    └──────┬──────┘
           │
           ▼ reuse()
┌─────────────────────┐
│ BuffersStorage Pool │
└─────────────────────┘
```

---

## 💡 Key Insights

1. **Zero-Copy Optimization**: Where possible, the function reuses existing buffers by adjusting positions/limits rather than copying data

2. **Stateful Processing**: `restOfTheData` maintains state between function calls to handle TCP stream fragmentation

3. **Memory Efficiency**: Buffer pooling via `BuffersStorage` prevents excessive allocations

4. **Protocol Flexibility**: Handles multiple protocol types (EF, DD, TLS) with different framing schemes

5. **Defensive Programming**: Extensive validation prevents buffer overruns and invalid state transitions

This design efficiently handles the inherent fragmentation of TCP streams while maintaining low memory overhead and CPU usage.
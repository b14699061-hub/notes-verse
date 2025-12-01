# NotificationCenter `listen` & ConnectionsManager – Flow and Patterns

  

## Overview

  

This note explains, from a software engineer’s perspective:

  

- How `NotificationCenter.listen(...)` works in `NotificationCenter.java`

- How `ConnectionsManager.java` *publishes* events via `NotificationCenter`

- The overall flow, mechanism, and design patterns behind Telegram’s in‑process event bus

  

Although `ConnectionsManager` does **not** directly call `NotificationCenter.listen(...)`, it **emits** events that UI components **consume** using `listen(...)`, `addObserver(...)`, or helper wrappers like `listenEmojiLoading(...)`. Together, these form a decoupled pub/sub system.

  

---

  

## 1. NotificationCenter as an Event Bus

  

### 1.1 Core Concepts

  

- `NotificationCenter` is a **per‑account event bus** plus a **global event bus**.

- Events are represented as **integer IDs** (e.g. `didUpdateConnectionState`, `emojiLoaded`, etc.).

- Observers implement the `NotificationCenterDelegate` interface and are registered/unregistered per event ID.

- `postNotificationName(...)` broadcasts an event (with optional arguments) to all observers.

  

Key parts:

  

```java

// NotificationCenter.java

public class NotificationCenter {

private final SparseArray<ArrayList<NotificationCenterDelegate>> observers = new SparseArray<>();

...

public interface NotificationCenterDelegate {

void didReceivedNotification(int id, int account, Object... args);

}

  

@UiThread

public static NotificationCenter getInstance(int num) { ... } // per-account bus

  

@UiThread

public static NotificationCenter getGlobalInstance() { ... } // global bus

}

```

  

This is essentially an **Observer pattern** implementation, specialized for a high‑cardinality set of event types.

  

---

  

## 2. `NotificationCenter.listen(...)` – View‑Scoped Observation

  

### 2.1 Implementation

  

`listen(...)` is a **convenience API** that:

  

- Wraps a raw `NotificationCenterDelegate` and a `View`

- Automatically **adds** the observer when the `View` is attached

- Automatically **removes** the observer when the `View` is detached

- Returns a `Runnable` that can be called to manually stop listening

  

```java

// NotificationCenter.java

public Runnable listen(View view, final int id, final Utilities.Callback<Object[]> callback) {

if (view == null || callback == null) {

return () -> {};

}

final NotificationCenterDelegate delegate = (_id, account, args) -> {

if (_id == id) {

callback.run(args);

}

};

final View.OnAttachStateChangeListener viewListener = new View.OnAttachStateChangeListener() {

@Override

public void onViewAttachedToWindow(View view) {

addObserver(delegate, id);

}

@Override

public void onViewDetachedFromWindow(View view) {

removeObserver(delegate, id);

}

};

view.addOnAttachStateChangeListener(viewListener);

  

return () -> {

view.removeOnAttachStateChangeListener(viewListener);

removeObserver(delegate, id);

};

}

```

  

### 2.2 Behavior & Intent

  

- The **delegate** filters events on `_id == id`, then forwards `args` to the provided `callback`.

- `View.OnAttachStateChangeListener` ensures observers are:

- Registered **only** while the view is part of the view hierarchy.

- Automatically cleaned up when the view is detached (avoiding leaks).

- The returned `Runnable` gives you **manual control** to unsubscribe early (e.g. when a dialog closes).

  

This is a lightweight **lifecycle‑aware observer** wrapper:

  

> *Tie the lifespan of your event subscription to the lifespan of a UI view, without repeating boilerplate add/remove calls.*

  

---

  

## 3. NotificationCenter Dispatch – How Events Travel

  

### 3.1 Adding / Removing Observers

  

Raw observer management:

  

```java

public void addObserver(NotificationCenterDelegate observer, int id) {

if (BuildVars.DEBUG_VERSION) {

if (Thread.currentThread() != ApplicationLoader.applicationHandler.getLooper().getThread()) {

throw new RuntimeException("addObserver allowed only from MAIN thread");

}

}

if (broadcasting != 0) {

// defer modifications while iterating observers

...

return;

}

ArrayList<NotificationCenterDelegate> objects = observers.get(id);

if (objects == null) {

observers.put(id, (objects = createArrayForId(id)));

}

if (!objects.contains(observer)) {

objects.add(observer);

}

}

  

public void removeObserver(NotificationCenterDelegate observer, int id) {

// Similar logic, with deferral if broadcasting != 0

}

```

  

Design aspects:

  

- **Main‑thread only**: registration and removal must be on the UI thread (in debug mode it will crash otherwise).

- **Broadcast‑safe**: while broadcasting (`broadcasting > 0`), add/remove operations are queued in `addAfterBroadcast` / `removeAfterBroadcast` and applied after the broadcast finishes.

- Some busy event IDs get a `UniqArrayList` wrapper for faster `contains()` and deduplication.

  

### 3.2 Posting Events

  

High‑level API:

  

```java

public void postNotificationName(final int id, Object... args) {

boolean allowDuringAnimation = ...; // special cases

...

if (shouldDebounce(id, args) && BuildVars.DEBUG_VERSION) {

postNotificationDebounced(id, args);

} else {

postNotificationNameInternal(id, allowDuringAnimation, args);

}

...

}

```

  

Core dispatch:

  

```java

@UiThread

public void postNotificationNameInternal(int id, boolean allowDuringAnimation, Object... args) {

if (BuildVars.DEBUG_VERSION) {

if (Thread.currentThread() != ApplicationLoader.applicationHandler.getLooper().getThread()) {

throw new RuntimeException("postNotificationName allowed only from MAIN thread");

}

}

if (!allowDuringAnimation && isAnimationInProgress()) {

delayedPosts.add(new DelayedPost(id, args));

return;

}

if (!postponeCallbackList.isEmpty()) {

for (int i = 0; i < postponeCallbackList.size(); i++) {

if (postponeCallbackList.get(i).needPostpone(id, currentAccount, args)) {

delayedPosts.add(new DelayedPost(id, args));

return;

}

}

}

broadcasting++;

ArrayList<NotificationCenterDelegate> objects = observers.get(id);

if (objects != null && !objects.isEmpty()) {

for (int a = 0; a < objects.size(); a++) {

NotificationCenterDelegate obj = objects.get(a);

obj.didReceivedNotification(id, currentAccount, args);

}

}

broadcasting--;

if (broadcasting == 0) {

// apply deferred add/remove

...

}

}

```

  

Key properties:

  

- **Main‑thread dispatch** ensures UI safety.

- **Animation / heavy‑operation aware**:

- Events can be delayed during complex transitions (`isAnimationInProgress()`).

- Only whitelisted IDs may pass through.

- **Postpone callbacks** allow higher‑level managers to veto / delay certain notifications.

  

From a design pattern POV:

  

- This is an **Observer / Publish–Subscribe** system with:

- Main‑thread safety constraints

- Back‑pressure and debouncing support

- Lifecycle‑aware helpers (`listen`, `listenOnce`, etc.)

  

---

  

## 4. ConnectionsManager – Using NotificationCenter as Publisher

  

Although `ConnectionsManager` lives in `org.telegram.tgnet` (network layer) and never calls `NotificationCenter.listen(...)`, it **publishes** state changes that the UI listens to via `NotificationCenter`.

  

### 4.1 Connection State Changes

  

```java

// ConnectionsManager.java

public static void onConnectionStateChanged(final int state, final int currentAccount) {

AndroidUtilities.runOnUIThread(() -> {

getInstance(currentAccount).connectionState = state;

AccountInstance.getInstance(currentAccount)

.getNotificationCenter()

.postNotificationName(NotificationCenter.didUpdateConnectionState);

});

}

```

  

Flow:

  

1. Native code detects a connection state change and calls `ConnectionsManager.onConnectionStateChanged(...)`.

2. This method:

- Switches to the **UI thread** via `AndroidUtilities.runOnUIThread`.

- Updates the in‑memory `connectionState` field.

- Fires the `NotificationCenter.didUpdateConnectionState` event on the **per‑account** bus.

3. Any part of the UI that has:

  

```java

NotificationCenter.getInstance(account)

.addObserver(this, NotificationCenter.didUpdateConnectionState);

```

  

will receive `didReceivedNotification(id, account, args)` when the state changes.

  

From a pattern POV:

  

- `ConnectionsManager` plays the role of **publisher** in the Observer pattern.

- UI controllers and components are **subscribers**.

- There is **no direct dependency** from `ConnectionsManager` to specific UI classes.

  

### 4.2 Global Alerts (Proxy Errors)

  

```java

// ConnectionsManager.java

public static void onProxyError() {

AndroidUtilities.runOnUIThread(() ->

NotificationCenter.getGlobalInstance()

.postNotificationName(NotificationCenter.needShowAlert, 3)

);

}

```

  

Here:

  

- The network layer reports a proxy error as a **global event** (`needShowAlert`).

- Any global listener (e.g., an activity or a global dialog manager) can subscribe:

  

```java

NotificationCenter.getGlobalInstance()

.addObserver(this, NotificationCenter.needShowAlert);

```

  

- This is analogous to a **domain event**: UI chooses how to interpret and present it.

  

### 4.3 Example: Premium Flood Wait

  

```java

// ConnectionsManager.java

if (isPremiumFloodWait && delegate != null) {

delegate.onPremiumFloodWait(instanceNum, request->requestToken, isUpload);

}

...

public static void onPremiumFloodWait(int currentAccount, int requestToken, boolean isUpload) {

NotificationCenter.getInstance(currentAccount)

.postNotificationName(NotificationCenter.premiumFloodWaitReceived);

}

```

  

UI components can subscribe to `premiumFloodWaitReceived` to show specialized dialogs or banners.

  

---

  

## 5. Compare: `NotificationCenter.listen` vs `ConnectionsManager.listen`

  

It’s easy to confuse the two because they share the name `listen`, but they serve different layers and follow different patterns.

  

### 5.1 `NotificationCenter.listen(View, id, callback)`

  

- **Layer:** UI utility

- **Purpose:** Attach a **view‑scoped** observer to a `NotificationCenter` event.

- **Pattern:** Observer + lifecycle binding

- **Key behavior:**

- Registers on `onViewAttachedToWindow`

- Unregisters on `onViewDetachedFromWindow`

- Returns a `Runnable` to unsubscribe programmatically.

  

### 5.2 `ConnectionsManager.listen(requestToken, ...)`

  

```java

// ConnectionsManager.java

private void listen(int requestToken,

RequestDelegateInternal onComplete,

QuickAckDelegate onQuickAck,

WriteToSocketDelegate onWriteToSocket) {

requestCallbacks.put(requestToken, new RequestCallbacks(onComplete, onQuickAck, onWriteToSocket));

}

```

  

- **Layer:** Network / RPC layer (Java side of MTProto)

- **Purpose:** Register **per‑request callbacks** (completion, quick‑ack, write notifications) keyed by `requestToken`.

- **Pattern:** Callback registry / promise‑like dispatch

- **Flow:**

- When Java issues a request, `sendRequestInternal` allocates a `requestToken` and calls `listen(...)` with lambdas.

- When native code completes the request, it calls back into Java (`onRequestComplete`), which looks up `requestCallbacks.get(requestToken)` and executes the stored lambda.

- Finally, it removes the entry from `requestCallbacks` to avoid leaks.

  

Conceptually:

  

- `NotificationCenter.listen` → **multi‑cast** subscriptions to **named events**, primarily for UI.

- `ConnectionsManager.listen` → **single‑cast** callback for **one RPC request**, primarily for networking.

  

Both are callback registries, but:

  

- One is **event‑bus / Observer pattern**.

- The other is **RPC callback mapping / Future‑like**.

  

---

  

## 6. Design Patterns & Rationale

  

### 6.1 Observer / Publish–Subscribe

  

`NotificationCenter` is a classic **Observer / Pub–Sub** implementation with:

  

- **Loose coupling**: publishers (e.g., `ConnectionsManager`, `MessagesController`) never reference concrete subscribers.

- **Many‑to‑many** relationships: multiple observers can subscribe to the same event, and one observer can subscribe to many event IDs.

- **Context separation**: per‑account vs. global instances.

  

Pros:

  

- Reduces direct dependencies between layers (network, domain, UI).

- Makes it easy to plug in new features that react to existing events.

  

Cons / Trade‑offs:

  

- Event IDs are integers → less type safety, more reliance on documentation.

- Debugging can be harder because dispatch is indirect.

  

### 6.2 Lifecycle‑Aware Observers

  

`listen(View, ...)` adds a **View lifecycle–aware layer**:

  

- Observers attach automatically when the view is visible.

- Observers detach automatically when the view is detached.

- Greatly reduces the chance of memory leaks or stray notifications to dead UIs.

  

Pattern‑wise, this is similar to **Android LiveData** or **LifecycleOwner‑aware observers**, but implemented manually.

  

### 6.3 Event Bus vs. Direct Callbacks

  

- For **UI and cross‑cutting concerns** (theme changes, connection state, emoji loaded, etc.), Telegram prefers `NotificationCenter`:

- Changes propagate broadly.

- Consumers are loosely coupled.

- For **per‑request responses** (RPC calls), Telegram uses per‑request callbacks in `ConnectionsManager`:

- Each request token has its own callback.

- Results are delivered exactly once, then removed.

  

From a software engineering perspective, this split keeps:

  

- **Global state transitions** (like connection state) on the event bus.

- **Local, one‑off interactions** (like an RPC’s result) on a direct callback channel.

  

---

  

## 7. Mental Model Summary

  

Think of the system as two layered mechanisms:

  

1. **Low‑level, point‑to‑point callbacks** in `ConnectionsManager`:

- For each network request, register a completion/quick‑ack callback.

- This is close to a **Future/Promise** or **callback registry** pattern.

  

2. **High‑level, broadcast events** via `NotificationCenter`:

- `ConnectionsManager` and others publish domain events (e.g., connection state changed, proxy error).

- UI code uses `NotificationCenter.listen`, `addObserver`, or `listenOnce` to react, often scoped to a `View`’s lifecycle.

  

The combination provides:

  

- Efficient, low‑latency RPC handling.

- A flexible, decoupled UI reaction layer.
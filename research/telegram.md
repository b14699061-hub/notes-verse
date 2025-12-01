

## Receive Flow Overview
- `ApplicationLoader.postInitApplication()` wires up every `AccountInstance`: it initializes native libs, spins up `MessagesController`, `ConnectionsManager`, and restores any cached users so message processing can start as soon as the network connects.

```java

//192:276:TMessagesProj/src/main/java/org/telegram/messenger/ApplicationLoader.java

public static void postInitApplication() {
    if (applicationInited || applicationContext == null) {
        return;
    }
    applicationInited = true;
    NativeLoader.initNativeLibs(ApplicationLoader.applicationContext);
    // ...
    for (int a = 0; a < UserConfig.MAX_ACCOUNT_COUNT; a++) {
        UserConfig.getInstance(a).loadConfig();
        MessagesController.getInstance(a);
        if (a == 0) {
            SharedConfig.pushStringStatus = "__FIREBASE_GENERATING_SINCE_" + ConnectionsManager.getInstance(a).getCurrentTime() + "__";
        } else {
            ConnectionsManager.getInstance(a);
        }
        TLRPC.User user = UserConfig.getInstance(a).getCurrentUser();
        if (user != null) {
            MessagesController.getInstance(a).putUser(user, true);
            SendMessagesHelper.getInstance(a).checkUnsentMessages();
        }
    }
    // ...
}
```

## Network Entry Point
- The Java side of `ConnectionsManager` receives raw MTProto payloads from native. When they deserialize to `TLRPC.Updates`, they are forwarded on Telegram’s staging queue to `MessagesController.processUpdates()`, ensuring ordering and avoiding UI-thread work.

```java

// 757:772:TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java

public static void onUnparsedMessageReceived(long address, final int currentAccount, long messageId) {
    // ...
    final TLObject message = TLClassStore.Instance().TLdeserialize(buff, constructor, true);
    if (message instanceof TLRPC.Updates) {
        KeepAliveJob.finishJob();
        Utilities.stageQueue.postRunnable(() ->
            AccountInstance.getInstance(currentAccount).getMessagesController().processUpdates((TLRPC.Updates) message, false));
    } else {
        // ...
    }
}
```

## Per‑Message Logic
- `MessagesController.processUpdates()` handles every update type. For very common `updateShortMessage` / `updateShortChatMessage`, it lazily fetches any missing users/chats, validates `pts` continuity, materializes a full `TL_message`, and synchronizes UI, storage, and notifications. Failures to satisfy `pts` or missing data flip `needGetDiff`, so the controller later asks the server for a full difference.

```java

// 16776:16964:TMessagesProj/src/main/java/org/telegram/messenger/MessagesController.java

public void processUpdates(final TLRPC.Updates updates, boolean fromQueue) {
    // ...
    } else if (updates instanceof TLRPC.TL_updateShortChatMessage || updates instanceof TLRPC.TL_updateShortMessage) {
        // resolve users/chats/bots referenced in the short update
        if (missingData) {
            needGetDiff = true;
        } else {
            if (getMessagesStorage().getLastPtsValue() + updates.pts_count == updates.pts) {
                TLRPC.TL_message message = new TLRPC.TL_message();
                // build peer/from ids, apply entities, ttl, etc.
                message.unread = value < message.id;
                // persist unread state and dialog metadata
                MessageObject obj = new MessageObject(currentAccount, message, isDialogCreated, isDialogCreated);
                ArrayList<MessageObject> objArr = new ArrayList<>();
                objArr.add(obj);
                // push to UI
                AndroidUtilities.runOnUIThread(() -> {
                    // notify typing state changes as needed
                    updateInterfaceWithMessages(targetDialogId, objArr, 0);
                    getNotificationCenter().postNotificationName(NotificationCenter.dialogsNeedReload);
                });
                if (!obj.isOut()) {
                    getMessagesStorage().getStorageQueue().postRunnable(() ->
                        AndroidUtilities.runOnUIThread(() ->
                            getNotificationsController().processNewMessages(objArr, true, false, null)));
                }
                getMessagesStorage().putMessages(arr, false, true, false, 0, 0, 0);
            } else if (getMessagesStorage().getLastPtsValue() != updates.pts) {
                // enqueue out-of-order updates while a diff is fetched
                updatesQueuePts.add(updates);
            }
        }
    }
    // ...
}
```

- UI updates happen instantly on the main thread, while storage and notification work is offloaded to `MessagesStorage`’s queue, preventing frame drops when a burst of updates arrives.
- The global `pts`/`qts` logic allows Telegram to resume correctly even if the device was offline; messages are only marked unread if they advance beyond the tracked read max per dialog.

## State Repair & Difference Fetching
- Whenever `needGetDiff` is raised or when a new session starts, `MessagesController` calls `getDifference()` to reconcile all updates (channels rely on similar logic per channel ID). `ConnectionsManager`’s native callbacks trigger timers (`onUpdate`) and difference requests (`onSessionCreated`) on the staging queue.

```java

//781:787:TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java

public static void onUpdate(final int currentAccount) {
    Utilities.stageQueue.postRunnable(() ->
        AccountInstance.getInstance(currentAccount).getMessagesController().updateTimerProc());
}

public static void onSessionCreated(final int currentAccount) {
    Utilities.stageQueue.postRunnable(() ->
        AccountInstance.getInstance(currentAccount).getMessagesController().getDifference());
}
```

## Push Wakeups & Background Delivery
- Incoming FCM/HMS pushes don’t carry full messages; they wake the app so MTProto can reconnect. `ConnectionsManager.onInternalPushReceived()` simply triggers `KeepAliveJob` to bring up the network and let the normal update path above retrieve pending messages.

```java

//915:917:TMessagesProj/src/main/java/org/telegram/tgnet/ConnectionsManager.java

public static void onInternalPushReceived(final int currentAccount) {
    KeepAliveJob.startJob();
}
```

## Role of TMessagesProj_App
- `TMessagesProj_App` is a thin Android-application module. `ApplicationLoaderImpl` overrides `onGetApplicationId()` so the base module can stay generic while the app build supplies `BuildConfig.APPLICATION_ID`. Google Voice integration helpers just delegate to shared `AndroidUtilities`, so they don’t alter the receive pipeline itself.

```java

//5:10:TMessagesProj_App/src/main/java/org/telegram/messenger/ApplicationLoaderImpl.java

public class ApplicationLoaderImpl extends ApplicationLoader {
    @Override
    protected String onGetApplicationId() {
        return BuildConfig.APPLICATION_ID;
    }
}
```

```java

//16:22:TMessagesProj_App/src/main/java/org/telegram/messenger/GoogleVoiceClientService.java

public class GoogleVoiceClientService extends SearchActionVerificationClientService {
    @Override
    public void performAction(Intent intent, boolean isVerified, Bundle options) {
        AndroidUtilities.googleVoiceClientService_performAction(intent, isVerified, options);
    }
}
```

### Key Takeaways
- Initialization ensures every account has a `ConnectionsManager` + `MessagesController` ready before any updates arrive.
- All incoming MTProto updates funnel through `ConnectionsManager.onUnparsedMessageReceived()` and are serialized on `Utilities.stageQueue` before `MessagesController` mutates state.
- Short updates are expanded into full `TL_message` objects, validated via `pts`, persisted, then pushed to UI/notifications; any gap triggers a `getDifference()` sync.
- Push messages merely wake the network; the reliable receive flow still goes through the MTProto connection, not FCM payloads.
- `TMessagesProj_App` mostly configures IDs/services and leaves the core receive logic untouched.

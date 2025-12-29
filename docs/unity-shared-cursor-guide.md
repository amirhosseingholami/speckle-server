# Speckle Shared Cursor + Follow Guide (Unity)

This guide explains how to add **shared cursors + camera follow** in a Unity 6 app so it interoperates with the Speckle web viewer and other Unity clients.

You already have Speckle servers deployed at:

- `https://dev.geovision.cloud`
- `https://app.geovision.cloud`

Use these servers’ GraphQL endpoints to send and receive viewer activity.

---

## 1) What this feature is in Speckle

Speckle’s web viewer broadcasts **viewer activity** (camera + selection). Other clients subscribe and render the remote user’s “cursor” based on that activity.

- The **cursor is a 3D point**, not a 2D mouse position.
- It is derived from:
  - the user’s **selection point**, or
  - a **fallback point** between camera target and camera position.

The Unity client should implement the **same mutation + subscription** used by the web viewer.

---

## 2) GraphQL: mutation + subscription

### 2.1 Mutation: `broadcastViewerUserActivity`

Send this **every ~5 seconds**, and when camera/selection changes.

```graphql
mutation BroadcastViewerUserActivity(
  $projectId: String!
  $resourceIdString: String!
  $message: ViewerUserActivityMessageInput!
) {
  broadcastViewerUserActivity(
    projectId: $projectId
    resourceIdString: $resourceIdString
    message: $message
  )
}
```

**Variables example:**

```json
{
  "projectId": "PROJECT_ID",
  "resourceIdString": "modelId:versionId",
  "message": {
    "userName": "Alice",
    "userId": "USER_ID",
    "sessionId": "SESSION_UUID",
    "state": {
      "ui": {
        "camera": {
          "position": [0, 0, 10],
          "target": [0, 0, 0]
        },
        "selection": [1.2, 0.0, -3.4]
      }
    },
    "status": "VIEWING"
  }
}
```

### 2.2 Subscription: `viewerUserActivityBroadcasted`

Open a **GraphQL WebSocket** connection and subscribe to receive other users’ activity.

```graphql
subscription OnViewerUserActivityBroadcasted(
  $target: ViewerUpdateTrackingTarget!
  $sessionId: String!
) {
  viewerUserActivityBroadcasted(target: $target, sessionId: $sessionId) {
    userName
    userId
    user {
      id
      name
      avatar
    }
    state
    status
    sessionId
  }
}
```

**Variables example:**

```json
{
  "target": {
    "projectId": "PROJECT_ID",
    "resourceIdString": "modelId:versionId",
    "loadedVersionsOnly": false
  },
  "sessionId": "SESSION_UUID"
}
```

---

## 3) How to build `resourceIdString`

Speckle uses a **comma-delimited list** of `modelId:versionId` pairs.

Examples:

- Single model:
  ```
  modelId:versionId
  ```
- Multiple models:
  ```
  modelId1:versionId1,modelId2:versionId2
  ```

---

## 4) Unity data model (SerializedViewerState)

You only need the **minimal fields** for cursor + follow:

```json
{
  "ui": {
    "camera": {
      "position": [x, y, z],
      "target": [x, y, z]
    },
    "selection": [x, y, z]
  }
}
```

**Notes:**

- `camera.position` and `camera.target` map directly to your Unity world camera.
- `selection` is the 3D point the user has selected (or raycast hit).

---

## 5) Unity tasks (step-by-step)

### 5.1 Authentication

All GraphQL requests need `Authorization: Bearer <token>`.

Use a Speckle personal access token (or OAuth token).

---

### 5.2 Implement GraphQL mutation sender

- Send activity updates every **~5 seconds**.
- Also send **on camera/selection changes**.
- Set `status` to `VIEWING`.

---

### 5.3 Implement GraphQL subscription listener

- Use **GraphQL over WebSocket** (the same token).
- Listen for `viewerUserActivityBroadcasted` events.

---

### 5.4 Render remote cursors

When a message arrives:

1. Parse `state.ui.selection` (preferred).
2. If `selection` is missing, compute fallback point:
   ```csharp
   fallback = Vector3.Lerp(cameraTarget, cameraPosition, 0.2f);
   ```
3. Convert the 3D point to **screen space** and render a cursor + label.
4. Show avatar + username from `user` / `userName` fields.

---

### 5.5 Implement “follow” (camera sync)

When the user clicks another user’s cursor:

- Set local camera **position + target** to the remote `state.ui.camera`.
- Optionally set selection highlight.

---

### 5.6 Handle disconnects / stale users

Web viewer behavior:

- Stale after ~100 seconds
- Remove after ~200 seconds

Unity recommendation:

- Track `lastUpdateTime` per `sessionId`.
- Remove after 2–3 update intervals.

---

## 6) Suggested Unity C# types

```csharp
[Serializable]
public class ViewerActivityMessage {
  public string userName;
  public string userId;
  public string sessionId;
  public string status;
  public UserInfo user;
  public SerializedViewerState state;
}

[Serializable]
public class UserInfo {
  public string id;
  public string name;
  public string avatar;
}

[Serializable]
public class SerializedViewerState {
  public UIState ui;
}

[Serializable]
public class UIState {
  public CameraState camera;
  public float[] selection; // [x,y,z]
}

[Serializable]
public class CameraState {
  public float[] position; // [x,y,z]
  public float[] target;   // [x,y,z]
}
```

---

## 7) Where to inspect the GraphQL schema

Your Unity dev can explore GraphQL schema and test queries here:

- `https://dev.geovision.cloud/graphql`
- `https://app.geovision.cloud/graphql`

Look for:

- `broadcastViewerUserActivity`
- `viewerUserActivityBroadcasted`
- `ViewerUserActivityMessageInput`
- `ViewerUpdateTrackingTarget`

---

## 8) Unity implementation checklist

**Required**

- [ ] GraphQL mutation every ~5s
- [ ] GraphQL subscription over WebSocket
- [ ] Send `SerializedViewerState` (camera + selection)
- [ ] Render cursor in screen space
- [ ] Show avatar + name label
- [ ] Remove stale users

**Optional**

- [ ] Follow / spotlight mode

---

## 9) Quick Q&A for Unity dev

**Q: Should we send 2D mouse position?**  
No. Speckle uses 3D points derived from selection or camera state.

**Q: Can Unity and web clients see each other?**  
Yes, if both use the same project + resourceIdString and the same mutation/subscription.

**Q: What if there is no selection point?**  
Use the camera fallback point:

```csharp
Vector3 fallback = Vector3.Lerp(cameraTarget, cameraPosition, 0.2f);
```

---

## 10) Reference paths in the Speckle repo (for deeper inspection)

These files show the exact web implementation and server GraphQL types:

- `packages/frontend-2/lib/viewer/composables/activity.ts`
- `packages/frontend-2/components/viewer/AnchoredPoints.vue`
- `packages/frontend-2/components/viewer/anchored-point/User.vue`
- `packages/frontend-2/lib/viewer/composables/anchorPoints.ts`
- `packages/frontend-2/lib/viewer/graphql/mutations.ts`
- `packages/frontend-2/lib/viewer/graphql/subscriptions.ts`
- `packages/server/assets/comments/typedefs/viewer.gql`
- `packages/server/assets/comments/typedefs/comments.gql`
- `packages/server/modules/comments/graph/resolvers/comments.ts`
- `packages/shared/src/viewer/helpers/state.ts`

---

If you need help adapting this to Unity-specific camera controls or selection handling, provide details and I can map them exactly.

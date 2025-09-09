# Copilot Instructions for whatsmeow

## Project Overview

WhatsApp Web multidevice API client library in Go. Core architecture revolves around a `Client` struct that manages websocket connections, protocol handling, and event dispatching for WhatsApp's binary XML protocol.

## Essential Architecture Patterns

### 1. Client Initialization & Store Pattern

```go
// Standard initialization pattern - always use sqlstore.Container
container, err := sqlstore.New(ctx, "sqlite3", "file:store.db?_foreign_keys=on", dbLog)
deviceStore, err := container.GetFirstDevice(ctx) // or GetDevice(jid) for specific devices
client := whatsmeow.NewClient(deviceStore, logger)
```

-   **Device Store**: Persistent storage for crypto keys, sessions, app state. Never instantiate directly.
-   **JID vs LID**: Phone numbers use `@s.whatsapp.net`, Local IDs use `@lid`. Always handle both.
-   **Multi-device**: Use `container.GetAllDevices()` for multiple sessions, each needs separate Client instance.

### 2. Event-Driven Architecture

```go
client.AddEventHandler(func(evt interface{}) {
    switch v := evt.(type) {
    case *events.Message:
        // v.Message.GetConversation() for text
        // v.Info.Chat, v.Info.Sender for metadata
    case *events.Receipt:
        // Delivery/read confirmations
    case *events.Connected:
        // Ready to send messages
    }
})
```

-   **Critical**: Events are dispatched in goroutines. Use proper synchronization for shared state.
-   **Message Unwrapping**: Always call `evt.UnwrapRaw()` to handle wrapped messages (ephemeral, view-once, etc.).
-   **Handler Removal**: Never call `RemoveEventHandler` from within a handler - use `go` to avoid deadlock.

### 3. Binary Protocol Handling

```go
// All WhatsApp communication uses binary XML nodes
node := &waBinary.Node{
    Tag: "iq",
    Attrs: waBinary.Attrs{"type": "get", "id": client.GenerateMessageID()},
    Content: []waBinary.Node{...},
}
```

-   **Node Structure**: Tag (string), Attrs (map), Content ([]Node, []byte, or string)
-   **JID Parsing**: Use `types.ParseJID()` - handles validation and server detection
-   **Server Types**: `s.whatsapp.net` (users), `g.us` (groups), `broadcast` (status), `newsletter`, `lid` (hidden users)

### 4. App State Synchronization

```go
// App state manages contact lists, chat settings, labels
cli.FetchAppState(ctx, appstate.WAPatchName, fullSync, onlyIfNotSynced)
patch := appstate.BuildLabelEdit(labelID, name, color, deleted)
cli.SendAppState(patch)
```

-   **Critical**: App state patches are differential updates. Always check current state before modifying.
-   **Types**: `appstate.WAPatchName` constants define what data to sync (contacts, chats, labels, etc.)

### 5. Dangerous Internals Pattern

```go
// Access unexported methods for advanced functionality
dangerous := client.DangerousInternals()
dangerous.SendNode(ctx, node) // Direct protocol access
dangerous.RequestContactLIDSync(ctx) // Force LID mapping updates
```

-   **Generated Code**: `internals.go` is auto-generated from `internals_generate.go`
-   **Use Sparingly**: These bypass safety checks. Prefer public APIs when available.

## Key Conventions

### Message Sending

```go
jid := types.JID{User: "1234567890", Server: types.DefaultUserServer}
msg := &waE2E.Message{Conversation: proto.String("Hello")}
resp, err := client.SendMessage(ctx, jid, msg)
```

### Media Upload Flow

```go
data := []byte{...} // file data
uploaded, err := client.Upload(ctx, data, whatsmeow.MediaImage)
// Use uploaded.URL, uploaded.MediaKey in message
```

### Error Handling Patterns

-   **Connection Errors**: Usually require reconnection via `client.Connect()`
-   **Protocol Errors**: Check for `*whatsmeow.DisconnectedError`, `*socket.CloseError`
-   **Crypto Errors**: Often self-resolving via automatic retries

## Development Workflows

### Testing

-   Use `client_test.go` pattern for examples
-   Mock store with `store.NewNoOpContainer()` for unit tests
-   Integration tests require real WhatsApp QR scanning

### Building

-   `go build` - standard Go build, no special requirements
-   Protocol buffers auto-generated (don't edit proto/ manually)
-   Code generation: `go generate` updates internals.go

### Common Debug Patterns

```go
// Enable debug logging
log := waLog.Stdout("Client", "DEBUG", true)
// Monitor raw protocol
client.Log.Sub("Recv").Debugf("Got node: %v", node)
```

## Integration Gotchas

### Pairing & Authentication

-   QR pairing: `client.GetQRChannel()` → scan → automatic save
-   Pair codes: `client.PairPhone()` for programmatic pairing
-   **State**: Check `client.Store.ID == nil` for unpaired state

### Group vs Individual Messages

-   Groups: `chatJID.Server == types.GroupServer`, sender in `msg.Info.Sender`
-   Individuals: `chatJID.Server == types.DefaultUserServer`, sender equals chat

### Status/Stories vs Regular Messages

-   Status: Use `types.StatusBroadcastJID` as destination
-   Different retention and privacy rules apply

This guide focuses on patterns discoverable from code inspection. Refer to protocol documentation for message format details.

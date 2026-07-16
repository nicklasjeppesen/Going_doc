# WebSocket

- [Introduction](#introduction)
- [Creating a hub](#creating-a-hub)
- [Registering a hub](#registering-a-hub)
- [Middleware](#middleware)
- [Authenticated user](#authenticated-user)
- [Storing clients](#storing-clients)
  - [Hub storage](#hub-storage)
  - [Rooms](#rooms)
- [Receiving messages](#receiving-messages)
- [Sending messages](#sending-messages)
  - [From hub to client](#from-hub-to-client)
  - [To a room](#to-a-room)
  - [From controller to hub](#from-controller-to-hub)
- [App-to-socket messaging](#app-to-socket-messaging)
- [Frontend](#frontend)


## Introduction

Going implements WebSocket support using a hub-based architecture with room management, event routing, and integration with the application layer via an internal channel system.


## Creating a hub

Run the CLI command to generate a hub:

```
art make:hub post
```

This creates a new hub at `myapp/internal/app/http/hubs/post.go`:

```go
package hubs

import (
    "github.com/nicklasjeppesen/going_internal/super/channels"
    socket "github.com/nicklasjeppesen/going_internal/super/socket"
)

type PostHub struct {
    socket.BaseHub
}

// Register event handlers
func (post *PostHub) RegisterRoutes() {
    post.On("sendMessage", post.handleNewMessage)
}

// Handle incoming messages from clients
func (post *PostHub) handleNewMessage(parameters []string, client *socket.Client) error {
    // Handle the message
    return nil
}

// Handle client disconnection
func (post *PostHub) CancelConnection(client *socket.Client) {
}

// Handle messages sent from the application (controllers/jobs)
func (post *PostHub) AppReceiver(message channels.Socket) {

}
```

### Event-based routing

Use `On(event, callback)` to register event handlers. When a client sends a message with a specific action, the corresponding handler is called:

```go
func (post *PostHub) RegisterRoutes() {
    post.On("sendMessage", post.handleNewMessage)
    post.On("typing", post.handleTyping)
    post.On("joinRoom", post.handleJoinRoom)
}
```


## Registering a hub

Go to the socket route file at `myapp/internal/routes/socket.go`:

```go
func Socketrouter(socketRouter socket.Router) socket.Router {
    socketRouter.MapHub("/ws/post", &hubs.PostHub{})
    return socketRouter
}
```


## Middleware

Add optional middleware to your WebSocket routes:

```go
func Socketrouter(socketRouter socket.Router) socket.Router {
    socketRouter.MapHub("/ws/post", &hubs.PostHub{}, middleware.JWTMiddleware)
    return socketRouter
}
```


## Authenticated user

The client object contains the same Auth struct as the Request struct:

```go
func (post *PostHub) handleNewMessage(parameters []string, client *socket.Client) error {
    userId := client.Auth.ID
    // Use userId...
    return nil
}
```


## Storing clients

### Hub storage

Store clients directly in the hub's `Clients` map:

```go
func (message *MessageHub) RegisterUser(contents []string, client *socket.Client) error {
    message.Clients[client.Auth.ID] = client
    return nil
}

func (post *PostHub) CancelConnection(client *socket.Client) {
    delete(post.Clients, client.Auth.ID)
}
```

### Rooms

Going supports rooms for grouping clients:

```go
func (message *MessageHub) RegisterUser(contents []string, client *socket.Client) error {
    // Create a room with a custom ID
    room := message.Rooms.CreateRoomWithCustomUUID("messageRoom", "custom-id-123")
    client.RoomID = "custom-id-123"

    // Or create a room with a random UUID
    // room := message.Rooms.CreateRoom("messageRoom")

    err := message.Rooms.AddClientToRoom(client, room.ID)
    if err != nil {
        return err
    }
    return nil
}
```

Room operations:
- `CreateRoom(name)` - Create a room with a random UUID
- `CreateRoomWithCustomUUID(name, uuid)` - Create a room with a specific ID
- `Getroom(roomId)` - Get a room by ID
- `AddClientToRoom(client, roomId)` - Add a client to a room
- `RemoveClientFromRoom(client, room)` - Remove a client from a room
- `RemoveClientFromRooms(client)` - Remove a client from all rooms


## Receiving messages

The first parameter is an array of strings containing all inputs the user sent. **Always check if the array is nil:**

```go
func (post *PostHub) handleNewMessage(parameters []string, client *socket.Client) error {
    if len(parameters) == 0 {
        return nil
    }
    input := parameters[0]
    fmt.Println("Input from user:", input)
    return nil
}
```


## Sending messages

### From hub to client

Send a message to a specific client:

```go
func (messagehub *MessageHub) SendMessage(contents []string, client *socket.Client) error {
    client.SendMessage("newMessage", "Hello from the server")
    return nil
}
```

### To all clients in a room

```go
func (message *MessageHub) RegisterUser(contents []string, client *socket.Client) error {
    clients := message.Rooms.Getroom("1").Clients
    for _, c := range clients {
        c.SendMessage("newUser", client.GetId())
    }
    return nil
}
```

### To specific clients

Iterate over stored clients and send to specific users:

```go
func (messagehub *MessageHub) SendMessage(contents []string, client *socket.Client) error {
    chat := new(db.Chat).DB(context.Background()).Where("id", chatId).WithUsers().First()
    message := new(db.Message).DB(context.Background()).Where("id", messageId).First()

    for _, chatMember := range chat.Users().Items() {
        if c, ok := messagehub.Clients[fmt.Sprint(chatMember.Id)]; ok {
            c.SendMessage("newMessage", message)
        }
    }
    return nil
}
```

### From controller to hub

Send a message from a controller to a hub using the channels package:

```go
package controller

import "github.com/nicklasjeppesen/going_internal/super/channels"

type PostController struct {
	ControllerBase
}
func (c *PostController) SocketPost() Result {

    // send a message to the hub that handle this URL
	websocketChannel.SendMessageToSocket(
		channels.Socket{
			URL:     "/ws/post",
			Message: "SendMessage",
			Data:    map[string]any{
				// data from server
			},
		})
	return Response.Print("Just send a message to socket")
}
```

This triggers the `AppReceiver` method in your hub that handles the `/ws/post` URL.


## App-to-socket messaging

The `channels` package provides a bridge between application code (controllers, jobs) and WebSocket hubs. When you call `channels.SendMessageToSocket()`, the message is sent through an internal channel and routed to the appropriate hub's `AppReceiver` method.

```go
func (h *PostHub) AppReceiver(message channels.Socket) {
    // message.URL contains the target URL
    // message.Message contains the action name
    // message.Data contains the payload

    // Broadcast to all clients connected to this hub
    for _, client := range h.Clients {
        client.SendMessage(message.Message, message.Data)
    }
}
```

This is useful for sending real-time notifications from background jobs or controllers to connected WebSocket clients.


## Frontend

Going includes a JavaScript WebSocket client. Include it in your page:

```javascript
<script>
const socket = new GOSocket("ws://" + document.location.host + "/ws/post")

socket.onOpen(() => {
    console.log("WebSocket connection opened");
    socket.invoke("RegisterUser");
})

socket.onClose(() => {
    console.log("WebSocket connection closed");
})

socket.on("sendMessage", function(message) {
    console.log("Message from server:", message)
})

socket.on("newNotification", function(data) {
    console.log("Notification:", data)
})

socket.start()

// Send a message to the server
function callServer() {
    socket.invoke("sendMessage", "input from browser")
}
</script>
```

### JavaScript API

- `new GOSocket(url)` - Create a new WebSocket connection
- `socket.start()` - Open the connection
- `socket.onOpen(callback)` - Called when connection opens
- `socket.onClose(callback)` - Called when connection closes
- `socket.on(event, callback)` - Listen for server events
- `socket.invoke(action, ...args)` - Send a message to the server

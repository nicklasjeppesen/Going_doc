# Websocket

- [Introduction](#introduction)
- [Creating a websocket](#creating-a-websocket)
- [use of websocket](#use-of-websocket)
	- [Register a new websocket](#register-a-new-websocket)
	- [Add middleware](#add-middleware)
- [Send a message from the server to the client](#send-a-message-from-the-server-to-the-client)
- [Frontend](#frontend)


<a name="introduction"></a>
## Introduction

This section describe how to start going websocket. going implement web socket by the pattern of using hubs. 

<a name="creating-a-websocket"></a>
## Creating a websocket

Run the command
```
art make:hub post
```
This generate a new hub and place it in the path: `myapp/internal/app/http/hubs/post`

```go
package hubs

import (
	"fmt"

	"github.com/nicklasjeppesen/going_internal/super/channels"
	socket "github.com/nicklasjeppesen/going_internal/super/socket"
)

type PostHub struct {
	socket.BaseHub
}


// Register the routes for the chat hub
func (post *PostHub) RegisterRoutes() {
	post.On("sendMessage", post.handleNewMessage)
}


func (post *PostHub) handleNewMessage(parameters []string, client *socket.Client) error {
	
    // handler for when reciving a message
	return nil
}


// handler when user is cancle a connection
func (post *PostHub) CancelConnection(client *socket.Client) {
}

// Handle message send from the app
func (MessageHub *PostHub) AppReceiver(message channels.Socket) {
    
}

```
This is example on how to hub will look like. 

<a name="use-of-websocket"></a>
## Use of websocket
This section will exlain how to use going websocket, for the server side application

<a name="register-a-new-websocket"></a>
### Register a new websocket
Go to the socket route file here: `myapp/internal/routes/socket.go`, and add register the URL and 

```go 
/*
|--------------------------------------------------------------------------
| Websocket routes
|--------------------------------------------------------------------------
|
| Here is where you can register websocket routes for your application.
*/
func Socketrouter(socketRouter socket.Router) socket.Router {
	socketRouter.MapHub("/ws/post", &hubs.PostHub{})
	return socketRouter
}
```

<a name="add-middleware"></a>
#### Add middleware
You can also add optional middleare to your middleware request, like this: 

```go 
/*
|--------------------------------------------------------------------------
| Websocket routes
|--------------------------------------------------------------------------
|
| Here is where you can register websocket routes for your application.
*/
func Socketrouter(socketRouter socket.Router) socket.Router {
	socketRouter.MapHub("/ws/post", &hubs.PostHub{}, ...middlewares)
	return socketRouter
}
```

### Get authenticate user 
The client object contain the same Auth struct as in the Request struct in struct. 

```go 
func (post *PostHub) handleNewMessage(parameters []string, client *socket.Client) error {
	
    auth := client.Auth
    // Futher actions with Auth. 
	return nil
}
```

### Storing clients
There several options for storing clients that has been registers to your hub. 

#### Hubs Storage
When a clients get registers you can choose to storing this user in hubs array like this: 

```go
// Register a new client to this hubs
func (message *MessageHub) RegisterUser(contents []string, client *socket.Client) error {

	message.Clients[client.Auth.ID] = client
	return nil
}

// handler when user is cancle a connection
func (post *PostHub) CancelConnection(client *socket.Client) {
	delete(post.Clients, client.Auth.ID)
}
```
#### Rooms 
Going support the tradtional feature room for socket. where you can store a clients i a room, for later actions. 

this 

```go

func (message *MessageHub) RegisterUser(contents []string, client *socket.Client) error {

	var customUUID = "43dfggg&dsGv5" // Random id
    var roomName = "messageRoom"
	newRoom := message.Rooms.CreateRoomWithCustomUUID(roomName, customUUID)
	client.RoomID = customUUID
    
    var errors = message.Rooms.AddClientToRoom(client, newRoom.ID)
	if errors != nil {
		fmt.Println("Error adding client to room:", errors)
        return errors
	}
	return nil
}
```
You can also choose to get a random room id by run the following: 
```go
newRoom := message.Rooms.CreateRoom(roomName)
```

### Get Message from incomming message
The first input for a request, is an array of strings, contain a list of al the input the user added to the request. NOTE: the paramter array can be nil, so remember check if it null. 

```go 
func (post *PostHub) handleNewMessage(parameters []string, client *socket.Client) error {
    input_one = parameters[0]
    fmt.Println("Input from user is: ", input_one)
    // Futher actions 
	return nil
}
```

<a name="send-a-message-from-the-server-to-the-client"></a>
##Send a message from the server to the client

There is 2 way you can send a message, from the hub, or in your controller. 

### Send a Message from the server to the client:
```go

 // Function to send a message to other user in a chat 
func (messagehub *MessageHub) SendMessage(contents []string, client *socket.Client) error {

        actionClientListenFor := "sendMessage"
        messageToTheClient := "Hello from the server"
        client.SendMessage(actionClientListenFor, messageToTheClient)	
	return nil
}
```

### Send message to people in a room

``` go 
/*
 * Function to send a message to other user in a chat
 */
func (messagehub *MessageHub) SendMessage(contents []string, client *socket.Client) error {

	var user = new(db.Chat).DB(context.Background()).Where("id", chatId).WithUsers().First()
	var message = new(db.Message).DB(context.Background()).Where("id", messageId).First()

	for _, chatMember := range chat.Users().Items() {
		if client, ok := messagehub.Clients[fmt.Sprint(chatMember.Id)]; ok {
			client.SendMessage(SendMessage, message)
		}
	}

	return nil
}
```
You can also send a message to all clients in a room. 
``` go 
func (message *MessageHub) RegisterUser(contents []string, client *socket.Client) error {

	clients := message.Rooms.Getroom("1").Clients
	for _, _client := range clients {
		_client.SendMessage("NewUser", client.GetId(), "input_two")
	}
	return nil
}
```
You can also send a message to client from your controllers in the app, as the following: 
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
After this, it will be the function ```AppReceiver``` in your hub, that will handle this command.

<a name="frontend"></a>
## Frontend: 
In going project you also receive a javascript file for handling websocket. you should include that into your page, here is an example on how to using it: 

```javascript 
<script> 
 const socket = new GOSocket("ws://" +  document.location.host + "/ws/post")

    socket.onOpen(() => {
        console.log("WebSocket connection opened");
		 socket.invoke("RegisterUser");
	})

    socket.onClose(() => {
        console.log("WebSocket connection Closed");
    })

    socket.on("sendMessage", function(message) {
        console.log("Message from the server")
        console.log(message)
    })

    socket.start()

    // Send message to the server
    function callServer() {
        socket.invoke("sendMessage", "input from browser")
    }
</script>

```
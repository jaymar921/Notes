# Understanding the Real-time Web
This will be my notes from what I learned from my ASP.NET journey. - [Jayharron Abejar](https://jayharronabejar.info)
- I have translated the course to a markdown file, from [Getting Started with ASP.NET Core 2 SignalR](https://app.pluralsight.com/library/courses/aspdotnet-core-signalr-getting-started/table-of-contents) course path at PluralSight
### What is Real-time web?
According to Wikipedia:

The real-time web is a network web using technologies and practices that enable users to receive information as soon as it is published by its authors, rather than requiring that they or their software check a source periodically for updates.
# Server Sent Events
The real-time web is a network web using technologies and practices that enable users to receive information as soon as it is published by its authors, rather than requiring that they or their software check a source periodically for updates. [source](https://en.wikipedia.org/wiki/Server-sent_events)
### Pros and Cons
- Better than [Polling or Long Polling]
- Simple HTTP
- Auto Reconnects
- No Support for older browsers
- Easy Polyfilled
- Maximum HTTP connections issue
- Just text messages [not binary data]
- **One-way connection only**
## Backend code
#### An api deployed somewhere that will handle any order from **Jayharron's** coffee shop
```c#
ASP.NET SERVER
Domain: www.jaycoffee.com/

... Routes below belongs to an ApiController ...
/*
	OrderCoffee(Order order) endpoint,
	Accepts an order from a client then processes the order,
	then return's the ID of the processed order.
*/
[HttpPost]
[Route("Coffee")]
public IActionResult OrderCoffee(Order order){
	// start the process for order

	// lets assume that the order is being processed and we will return
	// the id of the order
	return Accepted(1); // returns an order id of '1'.
} 

// this endpoint will return if there is any updates for the current order
[HttpGet]
[Route("Coffee/{id}")]
public async void GetUpdateForOrder(int orderNo){
	Response.ContentType = "text/event-stream";

	// create the result object for the front-end to fetch
	CheckResult result;

	do{
		// check to see if there is any update from the given update
		result = _orderChecker.GetUpdate(orderNo);

		// let's say that it takes 3 seconds for the server to process the update
		Thread.Sleep(3000);

		// if the result is not new, do again
		if(!result.New) continue;
		
		// note that SSE only support text messages
		// writes the update directly to response
		await HttpContext.Response.WriteAsync(result.Update);
		// flush the buffer to the browser
		await HttpContext.Response.Body.FlushAsync();
	}while(!result.Finished);

	// close the connection when the last update was sent
	Response.Body.Close();
}
```

## Front-end code [javascript - fetch]
#### Let's say a React Front-End application of **Jayharron's** Coffee shop is deployed somewhere and a client is requesting for an order
```js
/*
	In the front-end, a client is preparing for the order, they select
	the product name and the product size in an input field from a form

	The client clicked the submitButton for their order then the 
	**submitBtnClicked** function is called
*/
const submitBtnClicked = () => {
	// prepare the objects to be sent for the api
	const product = '* SOME PRODUCT NAME *';
	const size    = '* SOME PRODUCT SIZE *';

	/*
		Assume that www.jaycoffee.com/api/Coffee is where we will send an order,
		the api returns the ID of the order then we will listen to the updates
		in the www.jaycoffee.com/api/Coffee/<id>
	*/
	fetch('www.jaycoffee.com/api/Coffee',{
		method: 'POST',
		body: {product, size}
	})
	.then( response => response.text()) // assume that the API returned the ID of the product
	.then( text => listen(text));       // listen for the server sent event from the API
}

// listens to the update of an order by using Server Sent Events [SSE]
const listen = (id) => {
	// instantiate an EventSource object
	var eventSource = new EventSource(`www.jaycoffee.com/api/Coffee/${id}`);

	// starts listening to event's by using the onmessage event handler
	eventSource.onmessage = (event) => {
		const dataUpdate = event.data;

		// do something with the dataUpdate
	}
}
```

# Web Sockets
A standardized way to use one TCP socket through which messages can be send from server to a client and vice versa
### Pros and Cons
- Full duplex messaging
- No 6 connection limit
- Multi data-type support (text/binary) which supports streaming audios and videos
- TCP socket upgrade
- WS Protocol
## Web Sockets Handshake: Request
```
GET /chat HTTP/1.1
Host: server.jaycoffee.com
Origin: client.jaycoffee.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: abcdefghijk
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
Sec-WebSocket-Extensions: deflate-stream
```

## Web Sockets Handshake: Response
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: kjihgfedcba
Sec-WebSocket-Protocol: chat
Sec-WebSocket-Extensions: deflate-stream
```
## Web Socket Message Types
- Text
- Binary
- Ping/pong
- Close

### Backend code
#### **Startup.cs** File
```c#
ASP.NET SERVER
Domain: www.jaycoffee.com/

... Configure method belongs to the Startup class ...

// this method gets called by runtime. use this method to configure thewebsocket
public void Configure(IApplicationBuilder app, IHostingEnvironment env){
	if(env.IsDevelopment())
	{
		app.UseDeveloperExceptionPage();
	}

	// configuring a websocket options
	var webSocketOptions = new WebSocketOptions()
	{
		KeepAliveInterval = TimeSpan.FromSeconds(120),
		ReceiveBufferSize = 4 * 1024
	}

	// configure a pipeline to use web sockets
	app.UseWebSockets(webSocketOptions);

	app.UseStaticFiles();
	app.useMvc();
}
```
#### Controller class
```c#
ASP.NET SERVER
Domain: www.jaycoffee.com/

... Routes below belongs to an ApiController ...
/*
	OrderCoffee(Order order) endpoint,
	Accepts an order from a client then processes the order,
	then return's the ID of the processed order.
*/
[HttpPost]
[Route("Coffee")]
public IActionResult OrderCoffee(Order order){
	// start the process for order

	// lets assume that the order is being processed and we will return
	// the id of the order
	return Accepted(1); // returns an order id of '1'.
} 


[HttpGet]
[Route("Coffee/{orderNo}")]
public async void GetUpdateForOrder(int orderNo){
	// getting the httpContext from the injected httpContextAccessor Object
	var context = _httpContextAccessor.HttpContext;

	// check if the incoming http request is a websocket handshake request
	if(context.WebSockets.IsWebSocketRequest)
	{
		/* 
			accept the websocket request, causing the client to receive a reply 
			and the socket is now upgraded,

			the upgrade returns a webSocket object
		*/
		var webSocket = await context.WebSockets.AcceptWebSocketAsync();

		// then sending the events to the socket by calling the method SendEvents
		await SendEvents(webSocket, orderNo);

		// closing the connection specifying normal closure, [Done sending data]
		await webSocket.CloseAsync(WebSocketCloseStatus.NormalClosure, "Done", CancellationToken.None);
	}else
	{
		context.Response.StatusCode = 400; // bad request
	}
}

private async Task SendEvents(WebSocket webSocket, int orderNo){
	CheckResult result;

	do{
		// get an update for the current order
		result = orderChecker.GetUpdate(orderNo);
		Thread.Sleep(2000);

		// check to see if the result is new, otherwise, check again
		if(!result.New) continue;

		// transforming the update result to a json string
		var jsonMessage = $"\"{result.Update}\"";

		// call the SendAsync on the websocket object that expects to pass the buffer
		await webSocket.SendAsync(
			buffer: new ArraySegment<byte>(array: Encoding.ASCII.GetBytes(jsonMessage),
			offset: 0,
			count: jsonMesage.Length),
			messageType: WebSocketMessageType.Text, // speficying that the message is a text type
			endOfMessage: true, // the segment is the last part of the message
			cancellationToken: CancellationToken.None);
	} while(!result.Finished);
}


```

### Front-End
```js
/*
	In the front-end, a client is preparing for the order, they select
	the product name and the product size in an input field from a form

	The client clicked the submitButton for their order then the 
	**submitBtnClicked** function is called
*/
const submitBtnClicked = () => {
	// prepare the objects to be sent for the api
	const product = '* SOME PRODUCT NAME *';
	const size    = '* SOME PRODUCT SIZE *';

	/*
		Assume that www.jaycoffee.com/api/Coffee is where we will send an order,
		the api returns the ID of the order then we will listen to the updates
		in the www.jaycoffee.com/api/Coffee/<id>
	*/
	fetch('www.jaycoffee.com/api/Coffee',{
		method: 'POST',
		body: {product, size}
	})
	.then( response => response.text()) // assume that the API returned the ID of the product
	.then( text => listen(text));       // listen for the server sent event from the API
}

// listens to the update of an order by using Web Socket [WS]
const listen = (id) => {
	// instantiate an WebSocket object
	const socket = new WebSocket(`ws://jaycoffee.com/api/Coffee/${id}`);

	// starts listening to socket event's by using the onmessage event handler
	socket.onmessage = (event) => {
		const dataUpdate = JSON.parse(event.data);

		// do something with the dataUpdate
	}
}
```

# ASP.NET Core SignalR
SignalR is an open source framework that wraps the complexity of real-time web transports
- Cross platform
- Lightweight
- Fast

SignalR consists of 2 parts
- Server side
- Client side
### SignalR Hubs
A hub protocol is a format used to serialize parameters to and deserialize paraments from. Default protocol is **JSON**
### Formats
```json
JSON (38 bytes)

{
	"Product": "Caramel Macchiato",
	"Size": "Vente"
}

MessagePack (30 bytes)

DF 00 00 00 02 A7 50 72 6F 64 75 63 74 B1 43 61 72 61 6D 65 
6C 20 4D 61 63 63 68 69 61 74 6F A4 53 69 7A 65 A5 56 65 6E 
74 65
```

# Working with ASP.NET Core SignalR
- Implementing a Hub
- Configuring SignalR
- Using Hubs in your application
- Authentication and Authorization
- Creating a browser client
- creating a .NET Client
- Configuring MessagePack

## Implementing a Hub
```c#
public class CoffeeHub : Hub
{
	private readonly OrderChecker _orderChecker;

	public CoffeeHub(OrderChecker orderChecker){
		_orderChecker = orderChecker;
	}

	public async Task GetUpdateForOrder(int orderId){
		CheckResult result;
		do
		{
			result = _orderChecker.GetUpdate(orderId);
			Thread.Sleep(1000);
			
			// if there is an update
			if(result.New)
				// send a notification updates to the client
				await Clients.Caller.SendAsync("ReceiveOrderUpdate", result.Update);
		}while(!result.Finished);

		await Clients.Caller.SendAsync("Finished");
	}
}
```

## Configuring SignalR
```c#
/*
 Inside the Startup.cs class
*/
public void ConfigureServices(IServiceCollection services)
{
	.. SERVICES ..

	// call the AddSignalR dependency in the services
	services.AddSignalR();
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
	if(env.IsDevelopment())
	{
		app.UseDeveloperExceptionPage();
	}

	app.UseStaticFiles();

	// adding the UseSignalR pipeline and configure the hub routing url
	app.UseSignalR(routes => routes.MapHub<CoffeeHub>("/coffeehub"));

	app.UseMvc();
}

```

## Using Hubs in your application
```c#
[Route("[controller]")]
public class CoffeeController : Controller
{
	private readonly IHubContext<CoffeeHub> coffeeHub;

	public CoffeeController(IHubContext<CoffeeHub> coffeeHub)
	{
		this.coffeeHub = coffeeHub;
	}


	[HttpPost]
	public async Task<IActionResult> OrderCoffee([FromBody] Order oder)
	{
		// notify all clients that a new order has been processed
		await coffeeHub.Clients.All.SendAsync("NewOrder", order);
		// save the order somewhere and get the order id
		return Accepted(1);
	}
}
```

Context, is a property in a hub that gives you user information
```c#
/*
	Inside the CoffeeHub class
*/

public override async Task OnConnectedAsync()
{
	// get the connection id of a client that caused a triggering of a hub
	var clientConnectionId = Context.ConnectionId;

	/*
		The connection Id can be used in the Client's property in the Hub class offers
	*/

	// you can send a specific client a notification
	await Clients.Client(clientConnectionId).SendAsync("NewOrder", /*some order here*/);

	// you can also send a notification to all client's except a client that you specified
	await Clients.AllExcept(clientConnectionId).SendAsync("NewOrder", /*some order here*/);

	// you can also group a clients in a hub
	await Groups.AddToGroupAsync(clientConnectionId, "NameOfGroup");

	// otherwise, you can also remove a client in a group
	await Groups.RemoveFromGroupAsync(clientConnectionId, "NameOfGroup");

	// call all clients in a group
	await Clients.Group("NameOfGroup").SendAsync("NewOrder", /*some order here*/);
}
```

## Authentication and Authorization
Protecting a Hub against an anonymous access works the same way as protecting a controllers in asp.net core, by simply putting **[Authorize]** in top of the Hub class
```c#
[Authorize]
public class CoffeeHub : Hub
{ /* ... */ }
```

### Implementing a Browser Client
The client side of a SignalR app needs a SignalR library
```cmd
> npm install @microsoft/signalr
```
You'll need the signalr.js and reference it to your application
```html
<script src="signalr.js"></script>
```
### Front-End
Vanilla JS
```js
// connection object for signalR
const connection = new signalR.HubConnectionBuilder()
								  .withUrl("/coffeehub")
								  .build();
/*
	In the front-end, a client is preparing for the order, they select
	the product name and the product size in an input field from a form

	The client clicked the submitButton for their order then the 
	**submitBtnClicked** function is called
*/
const submitBtnClicked = () => {
	// prepare the objects to be sent for the api
	const product = '* SOME PRODUCT NAME *';
	const size    = '* SOME PRODUCT SIZE *';

	/*
		Assume that www.jaycoffee.com/api/Coffee is where we will send an order,
		the api returns the ID of the order then we will listen to the updates
		in the www.jaycoffee.com/api/Coffee/<id>
	*/
	fetch('www.jaycoffee.com/api/Coffee',{
		method: 'POST',
		body: JSON.stringify({product, size}),
		headers: {
			'content-type': 'application/json'
		}
	})
	.then( response => response.text()) // assume that the API returned the ID of the product
	.then( id => connection.invoke("GetForOrderUpdate", id)); // invoke the signalR connection
}

/*
	Setting up connection for SignalR
*/
function setupConnection(){
	connection.on("RecieveOrderUpdate", (update) => {
		// update an element in the html
		document.getElementById("example").innerHTML = update;
	})

	connection.on("NewOrder", (order) => {
		// update an element in the html
		document.getElementById("example").innerHTML = "Someone has ordered an " + order.product;
	})

	connection.on("finished", () => {
		// do something when a request is done
		connection.stop();
	})

	/*
		It's a good idea to first create the handlers of the
		various function call before starting a connection.
	*/
	connection.start()
		.catch( err => console.error(err.toString()));
}

setupConnection();
```
Example NodeJS
```js
const signalR = require("@microsoft/signalr");

let connection = new signalR.HubConnectionBuilder()
    .withUrl("/chat")
    .build();

connection.on("send", data => {
    console.log(data);
});

connection.start()
    .then(() => connection.invoke("send", "Hello"));
```

## Implementing a .NET Client
The .NET client needs a NuGet package
```
> Microsoft.AspNetCore.SignalR.Client
```
The Api is almost the same as the client library for JavaScript
```c#
class Program
{
	static void Main(string[] args)
	{
		Console.WriteLine("Press a key to start listening..");
		Console.ReadKey();

		var connection = new HubConnectionBuilder()
							.WithUrl("http://jaycoffee.com/coffeehub")
							.Build();


		connection.On<Order>("NewOrder", (order) => {
			Console.WriteLine($"Somebody ordered an {order.Product}");
		})

		connection.StartAsync().GetAwaiter().GetResult();

		Console.WriteLine("Listening. Press a key to quit");
		Console.ReadKey();
	}
}
```
## Adding MessagePack
Adding a MessagePack protocol needs a NuGet package
```
> Microsoft.AspNetCore.SignalR.Protocols.MessagePack
```
In the Startup class, add the messagepack protocol in the configure service

```c#
services.AddSignalR().AddMessagePackProtocol();
```


##### [Simulate console in web/html page with websocket server.](https://github.com/suntabu/WebsocketConsoleTool.git)

Idea from [AndroidConsoleServer (I call it ACS)](https://github.com/suntabu/AndroidConsoleServer), but ACS uses jquery for looply fetch server data, not convenient for server commands developing. With websocket, server could send information to web page by itself.

Three important parts consist this console server, [**Commands Parser**](https://github.com/suntabu/WebsocketConsoleTool/tree/master/server/websocket/src/main/java/org/suntabu/commands) & [**Websockets**](https://github.com/suntabu/WebsocketConsoleTool/blob/master/server/websocket/src/main/java/org/suntabu/Server/SimpleWebSocketServer.java) & [**Html**](https://github.com/suntabu/WebsocketConsoleTool/tree/master/html)

###### Commands parser
Using [java annotation](https://en.wikipedia.org/wiki/Java_annotation) for mark the command methods:
``` java

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Command {

    public String value();

    public String description();
}
```

Then we could add new command by just add a new annotation method:
``` java

@Command(value = "hello",description = "say hello")
private void hello(String[] args){
    // say hi to web client side
    ConsoleManager.append("\t\tHi  ^ ^");
}

```


###### Websockets

Just handle the message sent from web page by parsing the command text:
``` java
@Override
public void onMessage(WebSocketFrame message, SimpleWebSocketServer simpleWebSocket) {
    String msg = message.getTextPayload();
    String result = console.console_run(msg);
    simpleWebSocket.send(result);
}

```

###### Html
Rendering the console layout and handling the server message sent via websockets:

``` javascript
function startWebSocket() {
    if (window.WebSocket) {
        updateConsole("This browser supports WebSocket!!")
        var wsUri = "ws://" + document.domain + ":8083/";
        websocket = new WebSocket(wsUri);
        websocket.onopen = function (evt) {
            onOpen(evt)
        };
        websocket.onclose = function (evt) {
            onClose(evt)
        };
        websocket.onmessage = function (evt) {
            onMessage(evt)
        };
        websocket.onerror = function (evt) {
            onError(evt)
        };
    } else {
        updateConsole("This browser does not supports WebSocket!!")
    }
}
function onOpen(evt) {
    updateConsole("connection built!!")
}
function onClose(evt) {
    updateConsole("connection lost!!")
}
function onMessage(evt) {
    console.log(evt.data);
    updateConsole(evt.data);
//            commandIndex = index;
//            $("#input").val(String(data));
}
function onError(evt) {
}

```



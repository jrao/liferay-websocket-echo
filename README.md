# Liferay Websocket Echo Portlet Example

## What is this?

This is an example of how to use the Liferay Websocket Whiteboard

This example will be executed in the following environment:

* Liferay Portal 7.0.0 GA3

## Creating a Liferay Porltet

### New MVC Portlet
We are going to use [Blade](https://dev.liferay.com/develop/tutorials/-/knowledge_base/7-0/blade-cli) to create a `Echo Portlet`:

```sh
blade create -t mvcportlet -p com.liferay.websocket.example.echo -c EchoPortlet echo-portlet
```

### Add websocket dependency
We will need to add the dependencies in the [build file](build.gradle), as shown below:

```gradle
dependencies {
...
    compileOnly group: "javax.websocket", name: "javax.websocket-api", version: "1.1"
...
}
```

### Create a Echo WebSocket Endpoint
We will create a [endpoint](src/main/java/com/liferay/websocket/example/echo/endpoint/EchoWebSocketEndpoint.java) that returns to the client the same message that receives.

```java
public class EchoWebSocketEndpoint extends Endpoint {

	@Override
	public void onOpen(final Session session, EndpointConfig endpointConfig) {
		session.addMessageHandler(
			new MessageHandler.Whole<String>() {

				@Override
				public void onMessage(String text) {
					try {
						RemoteEndpoint.Basic remoteEndpoint =
							session.getBasicRemote();

						remoteEndpoint.sendText(text);
					}
					catch (IOException ioe) {
						throw new RuntimeException(ioe);
					}
				}

			});
	}

}
```

To register this endpoint in the path `\o\echo` we will use declarative services as follow:

```java
@Component(
	immediate = true,
	property = {"org.osgi.http.websocket.endpoint.path=/o/echo"},
	service = Endpoint.class
)
```

### Create a Echo WebSocket Client

We are going to use the [Websocket API](https://www.w3.org/TR/2011/WD-websockets-20110419/) to create a JavaScript client for the echo Endpoint.

In our [view.jsp](src/resources/META-INF/view.jsp) file we will add a new 'button' that open the Websocket connection, 
when the button is clicked we just need to create a *new WebSocket* with the websocket URL:

```javascript
	websocket = new WebSocket( wsUri );
```

We will add another 'button'that closes the Websocket connection:

```javascript
	websocket.close();
```

Finally we will allow our client to send messages:

```javascript
	websocket.send(msg);
```

Here you can see a full example of the Websocket Clint

```jsp
<%@ page
		import="com.liferay.websocket.example.echo.endpoint.EchoWebSocketEndpoint" %>

<%@ include file="/init.jsp" %>

<div id="content">
	<div id="left_col">
		<aui:input label="url" name="urlInputText" readonly="true" type="text" value='<%= "ws://" + request.getServerName() + ":" + request.getServerPort() + EchoWebSocketEndpoint.ECHO_WEBSOCKET_PATH %>' />
		</br>

		<aui:button name="connect" onClick="initWebSocket();" value="connect" />
		<aui:button name="disconnect" onClick="stopWebSocket();" value="disconnect" />
		<aui:button name="state" onClick="checkSocket();" value="state" />
		<br />
		<aui:input name="message" onkeydown="ifevent(.keyCode==13)sendMessage();" type="text" />
		<aui:button name="send" onClick="sendMessage();" value="send" />
	</div>

	<div id="right_col">
		<aui:input name="debugTextArea" readonly="true" type="textarea" />
	</div>
</div>

<script type="text/javascript">
	var debugTextArea = document.getElementById('<portlet:namespace />debugTextArea');

	var wsUri = document.getElementById('<portlet:namespace />urlInputText').value;

	function debug(message) {
		debugTextArea.value += message + "\n\n";
		debugTextArea.scrollTop = debugTextArea.scrollHeight;
	}

	function sendMessage() {
		var msg = document.getElementById('<portlet:namespace />message').value;
		if ( websocket != null ) {
			document.getElementById('<portlet:namespace />message').value = "";
			websocket.send(msg);
			debug("Message sent: " + msg);
		}
		else {
			debug("Can't sent message, the connection is not open");
		}
	}

	var websocket = null;

	function initWebSocket() {
		try {
			if (typeof MozWebSocket == 'function')
				WebSocket = MozWebSocket;
			if ( websocket && websocket.readyState == 1 )
				websocket.close();
			websocket = new WebSocket( wsUri );
			websocket.onopen = function(evt) {
				debug("CONNECTED");
			};
			websocket.onclose = function(evt) {
				debug("DISCONNECTED");
			};
			websocket.onmessage = function(evt) {
				debug( "Message received: " + evt.data );
			};
			websocket.onerror = function(evt) {
				debug('ERROR: ' + evt.data);
			};
		}
		catch (exception) {
			debug('ERROR: ' + exception);
		}
	}

	function stopWebSocket() {
		if (websocket) {
			websocket.close();
		}
	}

	function checkSocket() {
		if (websocket != null) {
			var stateStr;
			switch (websocket.readyState) {
				case 0: {
					stateStr = "CONNECTING";
					break;
				}
				case 1: {
					stateStr = "OPEN";
					break;
				}
				case 2: {
					stateStr = "CLOSING";
					break;
				}
				case 3: {
					stateStr = "CLOSED";
					break;
				}
				default: {
					stateStr = "UNKNOW";
					break;
				}
			}
			debug("WebSocket state = " + websocket.readyState + " ( " + stateStr + " )");
		} else {
			debug("WebSocket is null");
		}
	}
</script>
```

## Prepare the Liferay Environment

### Register a websocket ServletContext

We will need to register a javax.servlet.ServletContext service with the property websocket.active set to true.

```java
@Component(immediate = true)
public class Configurator {

	@Activate
	public void activate(BundleContext bundleContext) {
		java.util.Dictionary<String, Object> servletContextProps =
			new Hashtable<>();

		servletContextProps.put("websocket.active", Boolean.TRUE);

		bundleContext.registerService(
			ServletContext.class, _servletContext, servletContextProps);
	}

	@Reference(target = "(original.bean=true)")
	private void setServletContext(ServletContext servletContext) {
		_servletContext = servletContext;
	}

	private ServletContext _servletContext;

}
```

### Prepare a Liferay Container

We will download a Liferay Portal Bundle from (SourceForge)[http://downloads.sourceforge.net/project/lportal/Liferay%20Portal/7.0.2%20GA3/liferay-ce-portal-tomcat-7.0-ga3-20160804222206210.zip?r=&ts=1476801340&use_mirror=kent]

We will need export the *javax.websocket and javax.websocket.server* packages. We can do that overriding the property *module.framework.system.packages.extra* in portal-ext.properties

```properties
module.framework.system.packages.extra=\
        com.ibm.crypto.provider,\
        com.ibm.db2.jcc,\
        com.microsoft.sqlserver.jdbc,\
        com.mysql.jdbc,\
        com.p6spy.engine.spy,\
        com.sun.security.auth.module,\
        com.sybase.jdbc4.jdbc,\
        oracle.jdbc,\
        org.postgresql,\
        org.apache.naming.java,\
        org.hsqldb.jdbc,\
        org.mariadb.jdbc,\
        sun.misc,\
        sun.net.util,\
        sun.security.provider,\
        javax.websocket;version="1.1.0",\
        javax.websocket.server;version="1.1.0"
```

Also we should install in *osgi/modules* the library com.liferay:com.liferay.com.liferay.websocket.whiteboard:1.0.0.
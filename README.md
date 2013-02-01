# RESTful PTK Integration

## Some Simple Interactions

Create a new conversation by POSTing an XMLHttpRequest using Javascript like the code below. First, we'll introduce a Javascript function to wrap the nuance of the XMLHttpRequest object; this will keep the code that deals with the mechanics of the request separate from the more interesting code that is the focus of our work. The remaining two Javascript functions send the request, and processes the result. Additionally, we'll declare a global scope `var interactionLocation` that will hold the resource URL of the created interaction.

	var interactionLocation;

	function ajaxRestRequest(url,method,callback) {
		var xhr = new XMLHttpRequest();
		xhr.open(method, url, true);
		xhr.setRequestHeader("Content-type","application/json");
		xhr.setRequestHeader("Accept", "application/json");
		xhr.onreadystatechange = function () {
			if ((xhr.readyState == 4)) {
				callback(xhr);
			}
		}
		return xhr;
	}

	function createConversation(url,segment,callback) {
		ajaxRestRequest(url,"POST",callback).send(JSON.stringify({"segment":segment}));
	}

	function onConversationCreated(xhr) {
		interactionLocation = xhr.getResponseHeader("Location");
	}

The following invocation would *kickoff* the process.

	createConversation("http://localhost:8000/conversations","PlatformToolKit",onConversationCreated);

The interactionLocation (global) holds the URL needed to interact with the conversation we just created.

To update the conversation, we again use an XMLHttpRequest. This time we will PUT the request, passing additional *context* data to the conversation. Our callback function shows how to *access* the data returned from the PTK; later we will see an example of how to use this data.

	function onConversationUpdated(xhr) {
		jsonPUT = eval('(' + xhr.responseText + ')');
	}

	function updateConversationWithUser(user,callback) {
		ajaxRestRequest(interactionLocation,"PUT",callback).send(JSON.stringify({"context":{"user":user}}));
	}

To create a callback, we need to PUT again. Instead of updating the context, we will update the contactUri. This attribute can contain both the callback number and an opitonal appointment time (for scheduled callbacks). Here is an example of the simpler ASAP callback. This assumes all data validation has occurred and that phoneNumber is a well formatted number.

	function onCallbackCreated(xhr) {
		jsonPUT = eval('(' + xhr.responseText + ')');
	}

	function updateConversationToAsapCallback(phoneNumber,callback) {
		ajaxRestRequest(interactionLocaiton,"PUT",onCallbackCreated).send(JSON.stringify({"contactUri":"voice://" + phoneNumber}));
	}

To better support appointments, we should refactor the code

	function updateConversationToAsapCallback(phoneNumber,callback) {
		updateConversationToCallback("voice://" + phoneNumber,callback);
	}

	function updateConversationToCallback(contactUri,callback) {
		ajaxRestRequest(interactionLocation,"PUT",onCallbackCreated).send(JSON.stringify({"contactUri":contactUri}));
	}

Now, we can just create another one-liner to handle scheduled callbacks

	function updateConversationToScheduledCallback(phoneNumber,appointmentTime,callback) {
		udpateConversationToCallback("voice://"+phoneNumber+"?appt="+appointmentTime,callback);
	}

## Putting It All Together

Lets create a `<div>` to house our callback widget. We'll class it as a widget, for styling, and give it an id for scripting.

	<div class="widget" id="cb_widget">

	</div>

Initially, the widget should not be visible, since we haven't yet gotten in touch with Virtual Hold. So, our CSS will look like

	div.widget {
		display: none;
	}

The styling information can be embedded inside  a `<style>` tag, or linked from our page to an external file as

	<link rel="stylesheet" type="text/css" href="callback.css"></link>

Now, we should have a webpage that displays nothing. This is much the same state we were in before we added any code. But, we've set the stage for great things to come...

Now we can add our createConversation function from above. But, we'll include a couple of new tricks. First, we'll source in the jquery library to simplify control of our web page. Second, we'll have the code actually *do* something (make our widget `<div>` appear) when we get back a reply.

	<html>
	<head>
	<title>My Simple Callback</title>
	<link rel="stylesheet" type="text/css" href="callback.css"></link>
	<script language="javascript" src="http://code.jquery.com/jquery-1.8.3.min.js"></script>
	</head>
	<body>
	<div class="widget" id="cb_widget">
		<button id="cb_button">Call me back now</button>
	</div>
	<script language="javascript">
	var interactionLocation;

	function ajaxRestRequest(url,method,callback) {
		var xhr = new XMLHttpRequest();
		xhr.open(method, url, true);
		xhr.setRequestHeader("Content-type","application/json");
		xhr.setRequestHeader("Accept", "application/json");
		xhr.onreadystatechange = function () {
			if ((xhr.readyState == 4)) {
				callback(xhr);
			}
		}
		return xhr;
	}

	$(document).ready(function() {
		createConversation("http://localhost:8000/conversations","PlatformToolKit",onConversationCreated);
	});

	function createConversation(url,segment,callback) {
		ajaxRestRequest(url,"POST",callback).send(JSON.stringify({"segment":segment}));
	}

	function onConversationCreated(xhr) {
		interactionLocation = xhr.getResponseHeader("Location");
		$('#cb_widget').show();
	}

	</script>
	</body>
	</html>

Now we can add a handler for the button click and see if we can get a callback. We'll also make our widget disappear, since our conversation will only be good for this one callback. Resonable implementations of updateConversationToAsapCallback and onCallbackCreated exist in the previous section.

	$('#cb_button').click(function() {
		updateConversationToAsapCallback("5551212",onCallbackCreated);
		$('#cb_widget').hide();
	});

Finally, here's our example of how to make use of the conversation state returned by the PTK methods. We can make our UI more informative by leveraging this information. We take the JSON response from the PTK, and move that into a local variable, using eval(). Then we access the originalWaitTime attribute of the conversation, use string functions to grab the minutes, convert that into an integer; finally we update the text of the callback button with a message that we will get a callback in *less than* the EWT plus one minute.

	function onConversationCreated(xhr) {
		interactionLocation = xhr.getResponseHeader("Location");
		json = eval('(' + xhr.responseText + ')');
		ewt = parseInt(json.originalWaitTime.split(':')[0]);
		$('#cb_button').text("Call me back in less than "+(ewt+1)+" minute(s)");
		$('#cb_widget').show();
	}

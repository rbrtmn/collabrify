# Installation and quick start #

Making your webapp Collabrified with Collabrify is a breeze.
Just drop this script tag in you html file:
```
	<script scr='http://collabrify-client-js.appspot.com/static/Collabrify.js'></script>
```
Now you can instantiate a Collabrify Client:
```
	var client = new CollabrifyClient({
		application_id: Long.fromString('1234567890'),
		user_id: 'example@gmail.com'
	});
```
Now start a collabrified session:
```
	var client = new CollabrifyClient({
		application_id: Long.fromString('1234567890'),
		user_id: 'example@gmail.com'
	});
	client.createSession({
		name: 'your_session_name',
		password: 'password',
		tags: ['your_sesson_tag1','your_sesson_tag2']
	});
```
On another webpage, lets search for the session and join in:
```
	client = new CollabrifyClient({
		application_id: Long.fromString('1234567890'),
		user_id: 'example@gmail.com'
	});
	client.listSessions(['your_sesson_tags']);
	.then(function (sessions) {
		client.joinSession({session: sessions[0], password: 'password'});
	});
	.then(function (session) {});
	.catch(function (e) {"catch any errors here"})
```
And, BOOM! your App is now collabrified. Send messages between clients like so:
```
	client.broadcast({some: 'data', any: 'data'})
```
And recieve the data on the other clients
```
	client.on('event', function (data) {
		// USE THE DATA HERE
	});
```


---


# Full API Documentation #

## Initialize ##

```
	var client = new CollabrifyClient({
		application_id: Long.fromString('1234567890'),
		user_id: 'example@gmail.com'
	});
```

## CollabrifyClient#createSession(Object) ##
```
	client.createSession({
		name: 'name of the session',
		password: 'password',
		tags: ['your_sesson_tag1','your_sesson_tag2']
	});
```
Returns a promise that get resolved when a sessions is created on server.

## CollabrifyClient#listSession(Array, boolean) ##
```
	client.listSession(['an', 'array', 'of', 'tags']);
```
Returns a promise that gets resolved when a list of sessions objects is available.

The second (optional, defaults to false) boolean argument determines whether a filter or exact match query is performed (false = filter, true = exact match).
With filter, a session will be included if their tag list contains all of the tags contained in the array.
With exact match, a session will be included if and only if their tag list matches the specified tag list.

Example:<br>
Session 1's tag: ['apple', 'orange', 'banana']<br>
listSession(['apple']) will include Session 1.<br>
listSession(['apple'], true) will not include Session 1.<br>

<h2>CollabrifyClient#joinSession(Object)</h2>
<pre><code>	client.joinSession({session: session, password: 'password'});<br>
</code></pre>
Returns a promise that gets resolved when the session is joined.<br>
<br>
<h2>CollabrifyClient#broadcast()</h2>
<pre><code>	client.broadcast({any: 'javascript', object: 1});<br>
</code></pre>
Returns a promise that gets resolved when broadcast is done. If a ArrayBuffer object is passed, it is passed along as raw data.<br>
<br>
<h2>CollabrifyClient#leaveSession()</h2>
<pre><code>	client.leaveSession();<br>
</code></pre>
Returns a promise that gets resolved when session has been left.<br>
<br>
<h2>CollabrifyClient#endSession()</h2>
<pre><code>	client.endSession();<br>
</code></pre>
Returns a promise that gets resolved when session has been left. Can only be called by owner.<br>
<br>
<h2>CollabrifyClient#preventFurtherJoins()</h2>
<pre><code>	client.preventFurtherJoins();<br>
</code></pre>
Returns a promise that gets resolved when a confirmation from the server is recieved. Can only be called by owner.<br>
<br>
<h2>CollabrifyClient#pauseEvents()</h2>
<pre><code>	client.pauseEvents();<br>
</code></pre>
Pauses incoming events.<br>
<br>
<h2>CollabrifyClient#resumeEvents()</h2>
<pre><code>	client.resumeEvents();<br>
</code></pre>
Resumes incoming events.<br>
<br>
<h2>CollabrifyClient#version</h2>
<pre><code>        console.log CollabrifyClient.version<br>
</code></pre>
Returns the client version<br>
<hr />
<h1>Events</h1>

<h2>event(event)</h2>
<h3>event#data()</h3>
Returns JSON parsed data<br>
<h2>event#rawData()</h2>
Returns raw bytes<br>
<h2>user_joined(user)</h2>
<h2>user_left(user)</h2>
<h2>sesson_ended</h2>
<h2>notifications_start</h2>
<h2>notifications_error</h2>
<h2>notifications_close</h2>
<hr />
<h1>Properties</h1>

<h3>CollabrifyClient#session</h3>
This object contains information about the active session. undefined otherwise.<br>
<br>
<h3>CollabrifyClient#participant</h3>
This object contains information about the current user if a session is active, undefined otherwise.<br>
<br>
<h3>CollabrifyClient#submission_registration_id</h3>
The current submission_registration_id.<br>
<hr />
<h1>Errors</h1>
All errors except for notification errors are handled through promises.<br>
When a broadcast call fails, the catch handler passes an Array of events (that failed) that have a resend() method that can be used to try to send the event again.<br>
<hr />
<h1>Compatibility</h1>

Should work on<br>
<h3>Android >= 4.0</h3>
<h3>iOS safari >= 6.0</h3>
<h3>IE 11 (Desktop and Windows Phone)</h3>
<h3>Chrome >= 7</h3>
<h3>Opera 11.6</h3>

Use of a Promise pollyfill is recommended. Take a look at <a href='https://github.com/then/promise'>https://github.com/then/promise</a>
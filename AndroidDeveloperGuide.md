**This document is meant for Collabrify Client version 3.00.**

# Installation #
  1. Download the latest Collabrify client zip.
  1. Extract the contents of the archive in your project's libs/ folder
  1. Add the following permissions in your manifest file:
```
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
```
  1. You will need to restart eclipse to enable javadoc/content assist in the editor.

---

# Instantiating the client object #
Before doing anything, you will need to create an instance of the CollabrifyClient object. The client object manages all communications with the Collabrify backend. Construct a CollabrifyClient object by calling the static factory method :
```
      myClient = CollabrifyClient.newClient(mContext, APPLICATION_ID,  
        getLatestEvent);
```
**mContext** refers to an instance of the [Android Context](http://developer.android.com/reference/android/content/Context.html) class. Collabrify will always use your application context.

Use the **APPLICATION\_ID** we provision for you. Email info@collabrify.it for more information.

If **getLatestEvent** is set to true, the client will only retrieve the latest event. If more than one event is broadcasted before they can be retrieved, the app will only see the latest event. A use case for this is an app that broadcasts it’s entire model in each event instead of broadcasting state changes.

---


# Collabrify Users #
Each installation of your app is associated with a unique userId. The id persists as long as the application's data isn't cleared. This allows you to track your users activities. In the future you will be able to manually assign userId's as well as provide some user metadata.

Each CollabrifyEvent contains the author's user info such as userId, displayName, etc.

### The display name ###
The display name is meant to be viewable by other app users. The display name also persists as long as the application's data isn't cleared.

To set a user's display name, call setDisplayName()
```
 myClient.setDisplayName("Collabratron");
```


---

# Using the client #

## Connecting to a session ##

### Creating a session ###
To create a session, call createSession()
```
      myClient.createSession(sessionName, tags, password, participantLimit, createSessionListener);

```

**sessionName** is a string that helps the user identify the session.

**tags** is a list of strings that helps describe the session. A session requires a minimum of 1 tag.

**password** is the session password, pass null or empty string if you do not want the session to be password protected.

**participantLimit** limits the number of clients that can connect to the session at any given time. Pass in 0 for no limit.

**createSessionListener** is an instance of CollabrifyCreateSessionListener.

> There is another version of this method that takes in a boolean argument **startPaused**. If set to true,  all incoming events will be paused upon creating a session.

Once the client has successfully created the session, createSessionListener.onSessionCreated() will be called. You will get a CollabrifySession object containing information about the newly created session.
```
  @Override
  public void onSessionCreated(final CollabrifySession session)
  {
    sessionId = session.id();
    sessionName = session.name();
    runOnUiThread(new Runnable()
    {
      @Override
      public void run()
      {
        showToast("Session created, id: " + session.id());
        connectButton.setText(sessionName);
      }
    });
  }
```
The above code saves the session id and name, displays a toast message to notify the user that the session has been created, and displays the session name on a button.

### Base Files ###
A base file is some data that the session creator can pass to the server when creating a session. When other participants join the session, they will be given the base file. The most common use case for the base file is to initialize the session with some previously saved data.

To create a session with a base file, call createSessionWithBase():
```
  myClient.createSessionWithBase(sessionName, tags, password,
    participantLimit, baseFileInputStream, createSessionListener);
```

**baseFileInput** stream is an input stream where the client can retrieve the base file from.

### Joining a session ###

To join a session, you need:
  * The **sessionID**
  * The **password** if the session has one
```
  myClient.joinSession(sessionId, password, joinSessionListener);
```

Once the client has successfully joined the session, joinSessionListener.onSessionJoined() will be called.

```
  @Override
  public void onSessionJoined(CollabrifySession session)
  {
    sessionName = session.name();
    sessionId = session.id();
    runOnUiThread(new Runnable()
    {
      @Override
      public void run()
      {
        showToast("Session Joined");
        connectButton.setText(sessionName);
      }
    });
  }
```

The above code saves the session id and name, displays a toast message to notify the user that they have joined the session, and displays the session name on a button.

### Finding a session ###

One way to join a session is to display a list of sessions for the user to choose from and then join it.

We can request a list of sessions for a given tag by calling listSessions()
```
  myClient.listSessions(tags, listSessionsListener);
```

Once the client has received the list of sessions, listSessionsListener.onReceiveSessionList() will be called, passing a list of sessions that match the specified tag.

```
  @Override
  public void onReceiveSessionList(List<CollabrifySession> sessionList)
  {
    if( sessionList.isEmpty() )
    {
      showToast("No session available");
      return;
    }
    displaySessionList(sessionList);
  }

  private void displaySessionList(final List<CollabrifySession> sessionList)
  {
    // Create a list of session names
    List<String> sessionNames = new ArrayList<String>();
    for( CollabrifySession s : sessionList )
    {
      sessionNames.add(s.name());
    }

    // create a dialog to show the list of session names to the user
    final AlertDialog.Builder builder = new AlertDialog.Builder(MainActivity.this);
    builder.setTitle("Choose Session");
    builder.setItems(sessionNames.toArray(new String[sessionList.size()]),
        new DialogInterface.OnClickListener()
        {
          @Override
          public void onClick(DialogInterface dialog, int which)
          {
            // when the user chooses a session, join it
            try
            {
              sessionId = sessionList.get(which).id();
              myClient.joinSession(sessionId, password, joinSessionListener,
                  sessionListener);
            }
            catch( CollabrifyException e )
            {
              onError(e);
            }
          }
        });

    // display the dialog
    runOnUiThread(new Runnable()
    {
      @Override
      public void run()
      {
        builder.show();
      }
    });
  }
```

## While in a session ##

### Broadcasting ###

A device can broadcast events to all other devices connected to the same session (including the device that broadcasted that event). Collabrify guarantees that all devices see the events in the same order.

```
  public void doBroadcast(View v)
  {
    if( broadcastText.getText().toString().isEmpty() )
      return;
    if( myClient != null && myClient.inSession() )
    {
      try
      {
        myClient.broadcast(broadcastText.getText().toString().getBytes(), "lol", broadcastListener);
        broadcastText.getText().clear();
      }
      catch( CollabrifyException e )
      {
        onError(e);
      }
    }
  }
```
The above code checks if broadcastText (a text field) is non-empty. If it is, and the client is currently in a session, the app extracts the text from broadcastText, serializes it into a byte array and then broadcasts that byte array as an event. The second argument of broadcast() is a string representing the event’s type. This is not used internally by Collabrify and serves as a means for the app to provide extra information with the event.

broadcast() returns an integer representing the Submission Registration ID of the submitted event.

Once the client has successfully submitted that event to the server, broadcastListener.onBroadcastDone() will be called.

The CollabrifyEvent object contains:
  * _orderId_ is the global order ID of the event, each event in a session has a unique order id assigned sequentially by the server (starting at 0).
  * _srid_ is the submission registration id that was returned from broadcast() for this event, -1 if this event was broadcasted from another client. This is useful to identify events before their order id is assigned/known.
  * _type_ is the event type that was passed in to broadcast(). This value is not used internally by Collabrify, it is provided here to help developers distinguish different events
  * _data_ is the raw event data
  * _elapsed_ is an approximation of how much time has passed since this event was originally received on the server (in milliseconds).

```
  @Override
  public void onBroadcastDone(CollabrifyEvent event)
  {
    showToast(new String(event.data()) + " broadcasted");
  }
```

The above code shows a popup message notifying the user when an event has been broadcasted.

### Leaving a Session ###
To leave a session, call leaveSession().
```
  public void doLeaveSession(View v)
  {
    if( !myClient.inSession() )
    {
      return;
    }
    try
    {
      myClient.leaveSession(false, leaveSessionListener);
    }
    catch( CollabrifyException e )
    {
      onError(e);
    }
  }
```
The boolean passed in to leave session determines whether or not the session will be terminated when this user successfully leaves. A terminated session will not appear in a session list and also will not allow new participants to join it. Participants already in the session cannot broadcast new events, but they will receive all remaining events that haven’t been already received. Only the original creator of the session can terminate it. If a session creator left the session without terminating it, they cannot terminate the session even if they rejoin the session. All sessions are automatically terminated after 12 hours have passed since it’s creation.

Upon successfully terminating/leaving a session, onDisconnect() will be called.
```
  @Override
  public void onDisconnect()
  {
    runOnUiThread(new Runnable()
    {
      @Override
      public void run()
      {
        showToast("Left session");
        connectButton.setText("CreateSession");
      }
    });
  }
```

The above code shows a pop up notification telling the user that they have left the session, and removed the session name from the button.

### Listening to session events ###

To listen to session events, set the client's session listener.
```
myClient.setSessionListener(sessionListener);
```

The session listener has several callbacks when different events happen during the course of the session. The most important of these callbacks is onReceiveEvent()

### Receiving events ###

When the client retrieves an event, it will call onReceiveEvent() with a CollabrifyEvent object as the parameter.

```
  @Override
  public void onReceiveEvent(CollabrifyEvent event)
  {
    runOnUiThread(new Runnable()
    {
      @Override
      public void run()
      {
        String message = new String(event.data());
        broadcastedText.setText(message);
      }
    });
  }
```

The above code converts the raw event data into a string and displays it.

### Receiving a basefile ###

This can be the second most important session callback depending on whether or not your app will utilize the base file.

If you join a session that has a base file, the client will automatically download it in the background. When the base file is complete, onBaseFileReceived() will be called with the baseFile as the argument. The client will not use the baseFile after the callback returns, and will attempt to delete it when the client leaves the session.

**Consult the CollabrifySessionListener interface reference to find out about other useful session callbacks**

# Error Handling #

All Collabrify listener interfaces has an onError() callback to notify you that an error has occured.

Collabrify has a simple error model: errors are either recoverable or unrecoverable.

When a recoverable exception occurs, the client's internal state is still valid. When an unrecoverable exception occurs, something has gone wrong and the client is left in an invalid state. In this case, the client will reset it's internal state and it will no longer be in a session. You should reset your interface to a default state to indicate that the user is no longer in a session.

```
  @Override
  public void onError(CollabrifyException e)
  {
    if(!e.isRecoverable())
    {
      //the client has been reset and we are no longer in a session, update UI
    }
    Log.e(TAG, "error", e);
  }
```
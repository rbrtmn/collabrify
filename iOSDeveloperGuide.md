# Installation #

Follow these steps:
  1. Download the Collabrify SDK and the protocol buffer library file.
  1. Drag the Collabrify framework to your project and select the checkbox to copy the files over.
  1. Drag the libprotobuf.a library to your project and select the checkbox to copy the file over.
  1. Drag the config.h file to your project and select the checkbox to copy the file over.
  1. Drag the Google Protocol Buffers folder to your project and copy the files over. When this group appears in Xcode, right click the group and select Show In Finder. Save this window for a subsequent step.
    1. Select your project in the navigator.
    1. Select **Build Settings**.
    1. Search or find **Header Search Paths**.
    1. Double click in this row to open up a list of search paths. Locate the Google Protocol Buffer folder location and drag the folder to this list. (Or type in: **"$(SRCROOT)/ProjectName/Google Protocol Buffers"** where ProjectName is replaced by your project name. Ensure that the Google Protocol Buffers have actually been added to your project)

**The Collabrify iOS zip file has a document with images to explain the last few steps from above**

Add the following dependencies:
  * SystemConfiguration.framework
  * libc++.dylib

You can add these dependencies by selecting your project in the navigator and selecting **Build Phases**. Once in **Build Phases**, select **Link Binaries With Libraries** and then select the **+** button. Type the name of those dependencies and press the **Add** button.

# Preparing the Client #

Because the Collabrify framework is an Objective-C framework, your project needs to use a Bridging Header to access the API from your Swift code. Your birding header has the following format: **ProjectName-Bridging-Header.h**

In this file, import the framework:
```
#import <Collabrify/Collabrify.h>
```

You can now access the client from your Swift code.

## Creating the Client ##

To interact with the Collabrify service, you must create a CollabrifyClient object:
```
   let client = CollabrifyClient(applicationID: 4891981239025664, getLatestEventOnly: false, baseURL: NSURL.URLWithString("http://166.collabrify-cloud.appspot.com"), userID: "collabrify.tester@gmail.com")

```

In your **viewDidLoad**, set the delegate:
```
client.delegate = self
```

```
NOTE: Do not forget to set the delegate above. Failure to do so will result in a failure of the client to inform your class of new events.
```

# The Delegate #

It is important to set the CollabrifyClient delegate so that your application can be informed about the happenings of a Collabrify session once you are in one.

First you must conform to the CollabrifyClientDelegate protocol in your class:
```
class ViewController: UIViewController, CollabrifyClientDelegate
```

Now that you conform to the protocol, you must implement the following delegate methods:
```
    func client(client: CollabrifyClient!, encounteredError error: CollabrifyError!)
    func client(client: CollabrifyClient!, receivedEvent event: CollabrifyEvent!) 
    func client(client: CollabrifyClient!, receivedBaseFile baseFile: NSData!) 
```

More information about these methods will be given below.

Creating a session has a few parameters:
  * **Name:** Each session must have a unique session name
  * **Password:** An optional password for the session. Pass nil if you do not want a password.
  * **Tags:** Tags describe a session and each session must have at least one tag
  * **Base File:** The Base File for the new session
  * **Start Paused:** Allows you to immediately pause all incoming events upon creating a session.
  * **Completion Handler:** A completion handler that is called when a session is created or when an error occurs.

The completion handler is a convenient way to specify what your application should do when creating a session is completed. It is an inline block of code:
```
    client.createSessionWithName(sessionName, password: "password", tags: tags, baseFile: nil, startPaused: false) { session, error in
            if error == nil {
               //You have created the session
            } else {
                println("Error Creating = \(error)")
            }
    }
```

The client will call your block of code passing in the newly created session as a CollabrifySession object. If an error occurs, the **error** object will contain the details.

```
NOTE: The example above uses a Swift trailing closure to keep the syntax simple and easier to read.
```

## Base Files ##
You may also create a session with a base file. A base file is the starting data for your session. Imagine you begin Collabrifying a drawing that you were working alone on. When you create the session, you would want to upload the current drawing so others can work on the same image.

You do this by passing a base file as **NSData** in to the create session method:
```
    client.createSessionWithName(sessionName, password: "password", tags: tags, baseFile: baseFileData, startPaused: false) { session, error in
            if error == nil {
               //You have created the session
            } else {
                println("Error Creating = \(error)")
            }
    }
```

```
NOTE: There is an optional delegate method you may implement to be informed when your base file has been uploaded. 

When uploading a base file, the base file is broken down into chunks and sent to Collabrify and may not be uploaded all at once if the base file is large. Implement the delegate if you want to update your interface when the base file has been completely uploaded:

    func client(client: CollabrifyClient!, uploadedBaseFileWithSize baseFileSize: UInt) 
```

# Joining a Session #

If you are not creating a session, you may want to join one! It is very easy to join a session.

When joining a session, you need:
  * The **sessionID**
  * The **password** if the session has one
  * Whether you want to pause events upon joining the session
  * A completion handler that is called upon joining a session or if an error occurs

Here is what all of this looks like:
```
     client.joinSessionWithID(session.ID, password: "password", startPaused: false) { session, error in
                if error == nil {
                    // You are now in a session
                } else {
                    println("Error Joining = \(error)")
                }
            }
```

The completion handler has a few parameters that are important:
  * **session:** The current session upon joining it
  * **error:** An error object that describes what went wrong. If everything is good, this will be nil

## Downloading the Base File ##
If you join a session that has a base file, the client will automatically download it in the background. When the base file is complete, it will call the delegate method:

```
    func client(client: CollabrifyClient!, receivedBaseFile baseFile: NSData!) 
```

# Finding a Session #

Because the user will not know the sessionID of a session before joining it, there is a convenience method to retrieve the sessions for the given **tags**:
```
    client?.listSessionsWithTags(tags) { sessions, error in
            self.refreshControl.endRefreshing()
            
            if (error != nil) {
                println("Error = \(error)")
            }
            
            //We need to cast to an array of Collabrify Sessions
            self.sessions = (sessions as [CollabrifySession])
            self.tableView.reloadData()
        }

```

The completion handler will pass back an array of CollabrifySession objects that match the specified **tags**. A CollabrifySession object contains the sessionID of each session that you can join.

```
NOTE: The above code example shows how to update a UITableViewController. When the tableView is reloaded, it will retrieve all the sessions in and display them.

In this example, client is an optional property which requires a ? to safely test if it is non-nil. Also, this example explicitly casts an Array of AnyObject (sessions) to an Array of CollabrifySession objects
```

# Broadcasting #

The most important part of Collabrify is communicating with the participants in a session. Once you are in a session, you can easy send data to everyone:

```
    let submissionID = client.broadcast(data, eventType: "Chat") { event, error in
                if error != nil {
                    println("Broadcast Error = \(error)")
                } else {
                    println("Broadcasted")
                }
            }
```

Broadcast takes two pieces:
  * **data:** The information you want to send as NSData.
  * **eventType:** The type of the event as an NSString.
  * **completionHandler:** Some code you want run after the broadcast completes successfully or unsuccessfully. In this example, the completionHandler is a trailing closure.

The broadcast method will return a submissionRegistrationID that you may use to update your model immediately. For example, you can update your model when broadcast returns, save your submissionRegistrationID, and then when the server actually sends the event, you can compare against the submissionRegistrationID to determine if you need to fix something in your model.

This may be important when ordering things in your model. If one user adds a blue box and another user adds a red box on top of it, you can fix the order they appear by using the orderID and the submissionRegistrationID.

```
NOTE: After broadcasting data, your client will still receive that data in the delegate method (see below).
```

# Receiving Events #

In order to receive events, you **must** implement the appropriate delegate method:

```
     func client(client: CollabrifyClient!, receivedEvent event: CollabrifyEvent!) {
        println("Received Event")
        
        if event.eventType == "Chat" {
            let message = NSString(data: event.data, encoding: NSUTF8StringEncoding)
            
            dispatch_async(dispatch_get_main_queue()) {
                self.messageLabel.text = message
            }
        }
    }
```

This method is called every time a new event is received. There are a few important things that this method gives you:
  * **client:** The client that received the event.
  * **event:** The event that is received

```
IMPORTANT: This method is NOT called on the main thread. If you need to touch your interface at all,  follow the dispatch_async above passing in dispatch_get_main_queue() and your block of code you want called.
```

# Leaving a Session #

When a user is done with a session, they should leave it or else their presence will remain around in that session.

To leave the session, simply call:
```
     let isOwner = client.participantID == client.currentSession().ownerID ? true : false
        
        client.leaveAndEndSession(isOwner) { success, error in
            if success {
                self.leaveSession()
            } else {
                println("Error Leaving = \(error)")
            }
        }
```

The completion handler is called. If you have successfully left the session, the success flag will be YES. Otherwise it is NO and the error object will contain the reason why.

If you are the session owner, you have one decision to make: **whether you want to end the session upon leaving it**.

If you end the session when leaving it, other participants in that session will be kicked out. Each participant will be informed that this has happened by implementing the delegate method:

```
    func client(client: CollabrifyClient!, sessionEnded sessionID: Int64) 
```

# Error Handling #

Collabrify has a simple error model: errors are either recoverable or unrecoverable.

When a **Recoverable** occurs, the client can continue without hiccup. When a **Unrecoverable** occurs, something has gone wrong and the client can no longer function. In this case, the session is no longer valid and **you should reset your interface to a default state to indicate that the user is no longer in a session**.

To be informed of the errors that occur, implement the delegate method:
```
    func client(client: CollabrifyClient!, encounteredError error: CollabrifyError!) {
        println("Encountered Error = \(error)")
        
        if error.isRecoverable() {
            
        }
    }
```

This example shows a few important things:
  * **Each error has a message:** The message will give you more information of the type of error and what the cause possibly was.
  * **Each error has a type:** The type of the error. **See CollabrifyErrorType.h**

# Using Protocol Buffers #

```
NOTE: You cannot import any of these files directly into a Swift file or in a Swift bridging header. You must wrap these classes and calls with an Objective-C wrapper and then import the resulting class(es) in your bridging header
```

To make cross platform development easier, we use Protocol Buffers for efficient serialization and parsing.

Protocol Buffers support C++ for the iOS platform, meaning we have a few additional steps.

The first thing to remember is to change any file that touches C++ code extension to **.mm**.

Next, you need to import the proper headers:
```
#import <google/protobuf/io/zero_copy_stream_impl_lite.h>
#import <google/protobuf/io/coded_stream.h>
```

## Parsing Messages ##

Protocol Buffers for C++ do not have a parse delimited method, so we wrote one for you:

```
NSData *parseDelimitedProtoBufMessageFromData(::google::protobuf::Message &message, NSData *data)
{
    ::google::protobuf::io::ArrayInputStream arrayInputStream([data bytes], [data length]);
    ::google::protobuf::io::CodedInputStream codedInputStream(&arrayInputStream);
    
    uint32_t messageSize;
    codedInputStream.ReadVarint32(&messageSize);
    
    //lets not consume all the data
    codedInputStream.PushLimit(messageSize);
    message.ParseFromCodedStream(&codedInputStream);
    codedInputStream.PopLimit(messageSize);
    
    if ([data length] - codedInputStream.CurrentPosition() > 0)
    {
         return [NSData dataWithBytes:((char *)[data bytes] + codedInputStream.CurrentPosition()) length:[data length] - codedInputStream.CurrentPosition()];
    }
    
    return nil;
}
```

If you do not need parse delimited, use the built in message class parse method:
```
bool MessageLite::ParseFromArray(const void* data, int size);
```

## Serializing Messages ##

Similarly, there is no delimited serialization method, so we wrote one:

```
NSData *dataForProtoBufMessage(::google::protobuf::Message &message)
{
    const int bufferLength = message.ByteSize() + ::google::protobuf::io::CodedOutputStream::VarintSize32(message.ByteSize());
    std::vector<char> buffer(bufferLength);
    
    ::google::protobuf::io::ArrayOutputStream arrayOutput(&buffer[0], bufferLength);
    ::google::protobuf::io::CodedOutputStream codedOutput(&arrayOutput);
    
    codedOutput.WriteVarint32(message.ByteSize());
    message.SerializeToCodedStream(&codedOutput);
    
    return [NSData dataWithBytes:&buffer[0] length:bufferLength];
}
```

If you do not need a delimited serialization method, use the built in message class method:
```
bool MessageLite::SerializeToArray(void* data, int size) const;
```
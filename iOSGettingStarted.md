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

Now that your project is setup, you should be able to import the framework:
```
#import <Collabrify/Collabrify.h>
```

## Creating the Client ##

To interact with the Collabrify service, you must create a CollabrifyClient object:
```
        CollabrifyClient* client = [CollabrifyClient clientWithApplicationID:4891981239025664
                                                          getLatestEventOnly:NO
                                                                     baseURL:[NSURL URLWithString:@"http://collabrify-cloud.appspot.com"]
                                                                      userID:@"collabrify.tester@gmail.com"];
        [client setDelegate:self]; 

```

```
NOTE: Do not forget to set the delegate above. Failure to do so will result in a failure of the client to inform your class of new events.
```

Create a property to keep the client around:
```
@property (strong, nonatomic) CollabrifyClient *client;
```

Now set this property to the just created client:
```
[self setClient:client];
```

# The Delegate #

It is important to set the CollabrifyClient delegate so that your application can be informed about the happenings of a Collabrify session once you are in one.

First you must conform to the CollabrifyClientDelegate protocol in your class:
```
@interface CITViewController () < CollabrifyClientDelegate >
```

Now that you conform to the protocol, you must implement the following delegate methods:
```
- (void)client:(CollabrifyClient *)client encounteredError:(CollabrifyError *)error;
- (void)client:(CollabrifyClient *)client receivedEventWithOrderID:(int64_t)orderID submissionRegistrationID:(int32_t)submissionRegistrationID eventType:(NSString *)eventType data:(NSData *)data;
- (void)client:(CollabrifyClient *)client receivedBaseFile:(NSData *)baseFile;
```

More information about these methods will be given below.

# Creating a Session #

Creating a session has a few parameters:
  * **Name:** Each session must have a unique session name
  * **Password:** An optional password for the session. Pass nil if you do not want a password.
  * **Tags:** Tags describe a session and each session must have at least one tag
  * **Start Paused:** Allows you to immediately pause all incoming events upon creating a session.
  * **Completion Handler:** A completion handler that is called when a session is created or when an error occurs.

The completion handler is a convenient way to specify what your application should do when creating a session is completed. It is an inline block of code:
```
  [[self client] createSessionWithName:sessionName password:@"password" tags:[self tags] baseFile:nil startPaused:NO completionHandler:^(CollabrifySession *session, CollabrifyError *error) {
        if (!error)
        {
            // You have now created a session
        }
    }];
```

The client will call your block of code passing in the CollabrifySession object your just created session. If an error occurs, the **error** object will contain the details.

## Base Files ##
You may also create a session with a base file. A base file is the starting data for your session. Imagine you begin Collabrifying a drawing that you were working alone on. When you create the session, you would want to upload the current drawing so others can work on the same image.

You do this by passing a base file as **NSData** in to the create session method:
```
  [[self client] createSessionWithName:sessionName password:@"password" tags:[self tags] baseFile:baseFileData startPaused:NO completionHandler:^(CollabrifySession *session, CollabrifyError *error) {
        if (!error)
        {
            // You have now created a session
        }
    }];
```

```
NOTE: There is an optional delegate method you may implement to be informed when your base file has been uploaded. 

When uploading a base file, the base file is broken down into chunks and sent to Collabrify and may not be uploaded all at once if the base file is large. Implement the delegate if you want to update your interface when the base file has been completely uploaded:

- (void)client:(CollabrifyClient *)client uploadedBaseFileWithSize:(NSUInteger)baseFileSize;
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
[[self client] joinSessionWithID:[session sessionID] password:@"password" startPaused:NO  completionHandler:^(CollabrifySession *session, CollabrifyError *error) {
        if (!error)
        {
            //update your interface;
        }
}];
```

The completion handler has a few parameters that are important:
  * **session:** The current session upon joining it
  * **error:** An error object that describes what went wrong. If everything is good, this will be nil

## Downloading the Base File ##
If you join a session that has a base file, the client will automatically download it in the background. When the base file is complete, it will call the delegate method:

```
- (void)client:(CollabrifyClient *)client receivedBaseFile:(NSData *)baseFile
{
    NSString *baseFileString = [[NSString alloc] initWithData:baseFile encoding:NSUTF8StringEncoding];
}
```

In this example, the base file is just simple text.

# Finding a Session #

Because the user will not know the sessionID of a session before joining it, there is a convenience method to retrieve the sessions for the given **tags**:
```
[[self client] listSessionsWithTags:[self tags] completionHandler:^(NSArray *sessions, CollabrifyError *error) {
        if (error) NSLog(@"Error = %@", error);
        
        [self setSessions:sessions];
        [[self tableView] reloadData];
    }];
```

The completion handler will pass back an array of CollabrifySession objects that match the specified **tags**. A CollabrifySession object contains the sessionID of each session that you can join.

```
NOTE: The above code example shows how to update a UITableViewController. When the tableView is reloaded, it will retrieve all the sessions in and display them.
```

# Broadcasting #

The most important part of Collabrify is communicating with the participants in a session. Once you are in a session, you can easy send data to everyone:

```
int submissionRegistrationID = [[self client] broadcast:textData eventType:@"ChatMessageType" completionHandler:^(CollabrifyEvent *event, CollabrifyError *error){}];
```

Broadcast takes two pieces:
  * **data:** The information you want to send as NSData.
  * **eventType:** The type of the event as an NSString.
  * **completionHandler:** Some code you want run after the broadcast completes successfully or unsuccessfully

The broadcast method will return a submissionRegistrationID that you may use to update your model immediately. For example, you can update your model when broadcast returns, save your submissionRegistrationID, and then when the server actually sends the event, you can compare against the submissionRegistrationID to determine if you need to fix something in your model.

This may be important when ordering things in your model. If one user adds a blue box and another user adds a red box on top of it, you can fix the order they appear by using the orderID and the submissionRegistrationID.

```
NOTE: After broadcasting data, your client will still receive that data in the delegate method (see below).
```

# Receiving Events #

In order to receive events, you **must** implement the appropriate delegate method:

```
- (void)client:(CollabrifyClient *)client receivedEvent:(CollabrifyEvent*)event
{
    NSString *chatMessage = [[NSString alloc] initWithData:[event data] encoding:NSUTF8StringEncoding];
        
    dispatch_async(dispatch_get_main_queue(), ^{
         [[self statusLabel] setText:chatMessage];
    });
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
[[self client] leaveAndEndSession:YES completionHandler:^(BOOL success, CollabrifyError *error) {
}];
```

The completion handler is called. If you have successfully left the session, the success flag will be YES. Otherwise it is NO and the error object will contain the reason why.

If you are the session owner, you have one decision to make: **whether you want to end the session upon leaving it**.

If you end the session when leaving it, other participants in that session will be kicked out. Each participant will be informed that this has happened by implementing the delegate method:

```
- (void)client:(CollabrifyClient *)client sessionEnded:(int64_t)sessionID;
```

# Error Handling #

Collabrify has a simple error model: errors are either recoverable or unrecoverable.

When a **Recoverable** occurs, the client can continue without hiccup. When a **Unrecoverable** occurs, something has gone wrong and the client can no longer function. In this case, the session is no longer valid and **you should reset your interface to a default state to indicate that the user is no longer in a session**.

To be informed of the errors that occur, implement the delegate method:
```
- (void)client:(CollabrifyClient *)client encounteredError:(CollabrifyError *)error
{
    if(![error isRecoverable])
    {
        //the client has been reset and we are no longer in a session, update UI
    }
    NSLog(@" Message: %@, Type: %@", error.message, error.type);
}
```

This example shows a few important things:
  * **Each error has a message:** The message will give you more information of the type of error and what the cause possibly was.
  * **Each error has a type:** The type of the error. **See CollabrifyErrorType.h**


# Using Protocol Buffers #

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
# What is Collabrify? #

> “Unfortunately, implementing ['Operational Transformation'] sucks. There's a million algorithms with
> different tradeoffs, mostly trapped in academic papers. The algorithms are really hard and time
> consuming to implement correctly… I am an ex Google Wave engineer. Wave took 2 years to write and if
> we rewrote it today, it would take almost as long to write a second time. (What??)” - Joseph Gentle,
> former Google Engineer

# Overview #
Collabrify is a straightforward platform that allows applications to share state in real-time. Shared state between applications is nothing new. Games have been doing it for many years. There are multiple approaches to real-time collaboration, such as Operational Transformation and Differential Synchronization, to name a few. All of the traditional approaches require the server-side software to have intimate understanding of the application state as well as data that causes the state to change. That works fine for very specific scenarios. However, for IMLC's MyDesk educational platform, which has multiple applications (with more to come) and limited developer manpower, we needed a solution which fit the following criteria:
  1. The server-side software must be general purpose instead of app-specific. Specifically, we didn't want to have to add new server-side logic for every additional application that we want to “collabrify”.
  1. The client-side software must be easy to integrate into any apps which can take advantage of the [Command Pattern](http://en.wikipedia.org/wiki/Command_pattern).

The result was a nice, general-purpose SDK that can be used for collaboration across almost any app. Collabrify provides these advantages:
  * The app-developer doesn't have to write a backend.
  * The app-developer doesn't have to write any networking code.
  * The app-developer can focus solely on the part that matters most: _The App_




---


## Terminology and the Domain ##
**App/Application**: The device-side software that the human user is directly interacting with.

**Collabrify Backend**: The highly scalable cloud-based backend that facilitates all Collabrify activities. See also API Reference: Collabrify Backend

**Collabrify Client**: The language-and-platform-specific device-side library that the App uses to interact with other Apps via the Collabrify Platform. The Collabrify Client sends and receives events to/from the Collabrify Backend on behalf of the App. See also API Reference: Collabrify Client


![http://i41.tinypic.com/swekxs.png](http://i41.tinypic.com/swekxs.png)

**Session**: All app-to-app collaboration happens in the context of Collabrify Sessions (just “sessions” in the rest of this document). These are hosted on the Collabrify Backend and by default have a maximum life-span of twelve hours.

**Session Tag**: One or more strings that are used to “tag” a session. The tags are chosen at session creation, and are used by other users in searching for sessions to join. The specific combination of session tags plus session name uniquely identifies a session globally. There cannot be two sessions in existence at the same time that have both identical tags and an identical name. When creating sessions in a K-12 classroom, teachers will instruct the students which tags to use, for example [“Ms. Newton” “Science” “3rd Period”]. When searching for those tags, the user will see a list of session names for sessions which have those tags. From there, the user will select the desired session (name).
**It is up to the app-writer to expose session tags to the user in a meaningful way.**

**Session Name**: The name that uniquely identifies a session within a group of sessions that share the exact same tags. Using the previous example (Ms. Newton's 3rd period science class), each group of students would collaborate in their own session, the name of which would differentiate that session from other sessions that also have the tags  [“Ms. Newton” “Science” “3rd Period”]. The specific combination of tags + name uniquely identifies sessions globally.

**Participant**: A representation of a user on a particular device (a user may be in the same session using multiple devices). A session has a maximum capacity of 5000 participants.

**Owner**: The participant who created the session.

**Base File**: A session can be initialized with a Base File, which is stored on the Collabrify Backend. For example, suppose user Alice wants to share a file with user Bob. In this case the Collabrify Client on Alice's device will upload the Base File to the Collabrify Backend, and start a session with it. When Bob joins the session, the Collabrify Client on his device downloads the Base File, and passes it to the application. The application then opens it and displays it for Bob.
Note: A Base File can have a maximum size of ~30MB. Larger file sizes will increase the amount of time it takes to join a session.

**Event**: A serialized expression of a discrete change in application state. Events are app-specific, as Collabrify is message-blind. The app should use the Command Pattern for events. For ease of programming, events should ideally be [Protocol Buffers](https://developers.google.com/protocol-buffers/docs/overview), but they can be anything that is serializable.
Note: Events can have a maximum size of just under ~1MB. Larger events will naturally take longer to transmit over the wire, which can potentially affect the user-experience. A typical event size is less than 1KB.

**Order ID**: A long (i.e. long integer) value that specifies where in the global order of events that a particular event occurs. This value is assigned by the Collabrify Backend.
NOTE: The Collabrify Client always passes events to the App in strict global order. The app can, and should, take advantage of this.

**Submission ID** (Submission Registration ID, SRID): An int value assigned to an event by the Collabrify Client when the App submits an event to the Client. It is the return value given by the Client when the App calls broadcast(). This happens before the event is sent to the server and assigned an Order ID. The App can use the Submission ID in certain implementations to assist in keeping track of “pending” events (i.e. events which have not been assigned an Order ID yet).


---


## Application State and Late Joiners ##
The distinguishing feature of the Collabrify Platform is that it stores application state, yet is message blind. Put another way, base files and events are just chunks of raw bytes as far as Collabrify is concerned. That raw data can be anything that an application needs it to be. The current state of an application is the initial Base File plus all Events (in order) that have occurred since the start of the session. If Chuck wants to join Alice and Bob's session which is already in progress, he can do so. Here is the process:
The Collabrify Client on Chuck's device notifies the Collabrify Backend that Chuck is joining the session. At this point, the app should be preventing any user interaction which could change the application state.
The Collabrify Client downloads the Base File and passes it to the app.
The Collabrify Client begins downloading all events, passing each (in order) to the app. The app should be internally updating its model with each new event. The app should be displaying a progress bar as this happens.
Note: We do not recommend displaying changes to the user while joining a session. Our user tests have indicated that this is an annoyance since it increases the amount of time involved in joining a session.
When there are no more events available, the application will then display the current state. Chuck will see the same thing that Alice and Bob are looking at. At this point, the app allows user interaction. Chuck can begin collaborating with Alice and Bob.


---


## Application-side Usage of OrderIDs ##
The orderIDs provide global ordering to application-generated events. There are a number of approaches to making use of the orderID when a user causes a state-change in a session.
Note: The Collabrify Client always passes events to the App in strict global order.

The first two steps are always:
  1. The user interacts with the app, intending to change the application state.
  1. The app generates an event E to represent this change (using the Command Pattern).

**The Naive Approach**:
  1. The app does NOT immediately display the change to the user.
  1. The app submits event E to the Collabrify Client.
  1. Other events may arrive during this time. The app can safely display these new events because they are globally ordered.
  1. Upon receiving notification that event E has been added, the app then displays the change.
This approach is extremely sensitive to latency. The user doesn't see the results of his/her edit until there has been a round-trip to the Collabrify Backend. The user-experience is likely to suffer. For a limited class of apps, this might be acceptable. For most apps, however, it is recipe for disaster. For example, think about the effect of doing this in a collaborative text editor (i.e. every keystroke must go to the backend before it is visible to the user).


**Better Approach #1**:

This approach will only work for certain types of apps. Specifically, it is suited for apps in which all of the events are idempotent. An idempotent event is an event that can be applied multiple times without changing the application state beyond the very first application of that event.
  1. The app does immediately display the change to the user.
  1. The app submits event E to the Collabrify Client.
  1. Other events may arrive during this time. The app displays these new events.
  1. Upon receiving notification that event E has been added, the app re-applies E to the application state.
This approach is simpler to implement (although there are corner cases to be aware of). However, very few apps are able to take advantage of this.


**Better Approach #2**:
  1. The app does immediately display the change to the user. The app will need to keep track of the fact that event E is not yet globally ordered, because its effect may not be final; there are many ways to do this.
  1. The app submits event E to the Collabrify Client.
  1. Other events may arrive during this time. The app displays these new events.
  1. Upon receiving notification that event E has been added, the app checks to see if event E was applied out-of-order. If so, the app reconciles the discrepancy; again, there are many ways to do this.
This approach is harder to implement. However, it does not have the restrictive use-case of Better Approach #1, nor it does suffer the Naive Approach's delay that occurs between when a user interacts with the app and when the user actually sees the effect of that interaction.



---


## Some Notes on Performance ##
  1. Backend Write-Throughput
> > On the Collabrify Backend, events can be transactionally added to a session at an average throughput of approximately ~5 events/second. AppEngine actually only guarantees a write-throughput of 1 transaction/second for any particular entity (your session in this case), although we have observed the upper limit to be approximately 10 transactions/second or more. This holds whether there is just 1 session or 10,000 sessions happening at a given moment in time. This means that for a 5-person session, each person can be adding events at a rate of up to 2 events/second, sustained.
  1. Backend Read-Throughput
> > There is no throughput limit to the number of events that can be fetched simultaneously from a session since a fetch-operation does not incur a write-transaction.


## Roadmap ##
Eventually the Collabrify Client will fill a more prominent role alongside “Collabrified” applications by providing default implementations for pending-event-reconciliation and undo/redo stacks. New applications written specifically for Collabrify will simply provide callbacks for each event type. When an event comes in over the wire, the Collabrify Client will call the appropriate handler for that event.  The Collabrify Client will provide its own default undo/redo stack. The application can tie its undo/redo buttons to the Collabrify Client's undo/redo stack, which, in turn, will use the appropriate event handlers to undo or redo events. In essence, future apps will be built on top of and driven by the Collabrify Client instead of simply using it to access the Collabrify Backend.
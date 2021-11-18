[//]: # (title: Creating a WebSocket chat client)

<microformat>
<var name="example_name" value="tutorial-websockets-client"/>
<include src="lib.xml" include-id="download_example"/>
</microformat>

In this tutorial, we will learn how to create a client chat application which uses WebSockets. The client application will allow users to join a common chat server, send messages to other users, and see messages from other users in the terminal.

To learn how to create a chat server, see the [](creating_web_socket_chat.md) tutorial.

## Prerequisites {id="prerequisites"}
<include src="lib.xml" include-id="client_prerequisites"/>

## Create a new project {id="new-project"}

To create a WebSocket chat client, we need to create a new project first. [Open IntelliJ IDEA](https://www.jetbrains.com/help/idea/run-for-the-first-time.html) and follow
the steps below:

1. <include src="lib.xml" include-id="new_project_idea"/>
2. In the **New Project** wizard, choose **kotlin** from the list on the left. On the right pane, specify the following settings:
   ![New Ktor project](tutorial_websockets_client_new_project.png){width="706"}
    * **Name**: Specify a project name.
    * **Location**: Specify a directory for your project.
    * **Project Template**: Choose _Application_ in the _JVM_ group.
    * **Build System**: Make sure that _Gradle Kotlin_ is selected.

   Click **Next**.

3. On the next page, change **Test framework** to _None_, click **Finish** and wait until IntelliJ IDEA generates a project and installs the dependencies.


## Configure the build script {id="build-script"}
The next thing we need is to configure the build script: 
- Apply the `application` plugin to be able to start our application using Gradle.
- Add the `JavaExec` task to correctly handle a user's input when the application is running using Gradle.
- Add dependencies required for a Ktor client.

### Apply the Application plugin {id="apply-plugin"}
Open the `build.gradle.kts` file and follow the steps below:
1. Apply the application plugin in the `plugins` block:
   ```kotlin
   plugins {
       application
   }
   ```
2. Specify the application's main class:
   ```kotlin
   ```
   {src="snippets/tutorial-websockets-client/build.gradle.kts" lines="8-10"}
   
   We'll create the `Application.kt` file later.

### Add the JavaExec task {id="java-exec"}

In the `build.gradle.kts` file, add the `JavaExec` task and specify `standardInput` as show below:

```kotlin
```
{src="snippets/tutorial-websockets-client/build.gradle.kts" lines="17-19"}


### Add client dependencies {id="add-client-dependencies"}

1. Open the `gradle.properties` file and add the following line to specify the Ktor version:
   ```kotlin
   ktor_version=%ktor_version%
   ```
   {interpolate-variables="true"}

2. Open the `build.gradle.kts` file and add the following artifacts to the `dependencies` block:
   ```kotlin
   ```
   {src="snippets/tutorial-websockets-client/build.gradle.kts" lines="1-2,21-25"}
3. Click the **Load Gradle Changes** icon in the top right corner of the `build.gradle.kts` file to install the dependencies.


## Create the Application.kt file {id="create-kotlin-file"}
Now we are ready to create a client.

1. Invoke the [Project view<](https://www.jetbrains.com/help/idea/project-tool-window.html)and expand the `src/main` folder.
2. Right-click the `kotlin` folder and choose **New | Kotlin Class/File**.
3. Specify a file name in the invoked popup and press **Enter** to create a Kotlin file.
   ![Application.kt](client_get_started_new_kotlin_file.png){width="362"}

## Create the chat client {id="create-chat-client"}

### First implementation {id="first-implementation"}

After [creating a Kotlin file](#create-kotlin-file), we can add an implementation of sending and receiving messages:

```kotlin
import io.ktor.client.*
import io.ktor.client.plugins.websocket.*
import io.ktor.http.*
import io.ktor.http.cio.websocket.*
import io.ktor.util.*
import kotlinx.coroutines.*

fun main() {
    val client = HttpClient {
        install(WebSockets)
    }
    runBlocking {
        client.webSocket(method = HttpMethod.Get, host = "127.0.0.1", port = 8080, path = "/chat") {
            while(true) {
                val othersMessage = incoming.receive() as? Frame.Text ?: continue
                println(othersMessage.readText())
                val myMessage = readLine()
                if(myMessage != null) {
                    send(myMessage)
                }
            }
        }
    }
    client.close()
    println("Connection closed. Goodbye!")
}
```

Here, we first create an `HttpClient` and set up Ktor's `WebSocket` plugin (the analog of installing the `WebSocket` plugin in our server application's module in an earlier chapter). Functions in Ktor responsible for making network calls use the suspension mechanism from Kotlin's coroutines, so we wrap our network-related code in a `runBlocking` block. Inside the WebSocket handler, we once again process incoming messages and send outgoing messages: we ignore frames which do not contain text, read incoming text, and send the user input to the server.

However, this "straightforward" implementation actually contains an issue that prevents it from being used as a proper chat client: when invoking `readLine()`, our program waits until the user enters a message. During this time, we can't see any messages which have been typed out by other users. Likewise, because we invoke `readLine()` after every received message, we would only ever see one new message at a time.

You can also validate this for yourself: with the server process running, start two instances of the chat client by clicking play icon in the gutter in `Application.kt`. Use the tabs in the Run tool window to navigate between the two client instances and send some messages back and forth.

![Run tool window](image-20201111191815343.png){width="1207"}

Let's address this issue, and build a better solution!

### Improved solution {id="improved-solution"}

A better structure for our chat client would be to separate the message output and input mechanisms, allowing them to run concurrently: when new messages arrive, they are printed immediately, but our users can still start composing a new chat message at any point.

We know that to output messages, we need to be able to receive them from the WebSocket's `incoming` channel, and print them to the command line. Let's add a function called `outputMessages()` to the `Application.kt` file with the following implementation for this functionality:

```kotlin
```
{src="snippets/tutorial-websockets-client/src/main/kotlin/Application.kt" lines="24-33"}

Because the function operates in the context of a `DefaultClientWebSocketSession`, we define `outputMessages()` as an extension function on the type. We also don't forget to add the `suspend` modifier – because iterating over the `incoming` channel suspends the coroutine while no new message is available.

Next, let's define a second function which allows the user to input text. Add a function called `inputMessages()` in `Application.kt` with the following implementation

```kotlin
```
{src="snippets/tutorial-websockets-client/src/main/kotlin/Application.kt" lines="35-46"}

Once again defined as a suspending extension function on `DefaultClientWebSocketSession`, this function's only job is to read text from the command line and send it to the server or to return when the user types `exit`.

Where we previously had one loop which had to take care of reading input and printing output, we now have separated these tasks into their own functions, which can operate independently of each other.

### Wire it together

Let's make use of our two new functions! We can call them inside the body of our WebSocket handler by changing the code of our `main()` method in `Application.kt` to the following:

```kotlin
```
{src="snippets/tutorial-websockets-client/src/main/kotlin/Application.kt" lines="7-22"}

This new implementation improves the behavior of our application: Once the connection to our chat server is established, we use the `launch` function from Kotlin's Coroutines library to launch the two long-running functions `outputMessages()` and `inputMessages()` on a new coroutine (without blocking the current thread). The launch function also returns a `Job` object for both of them, which we use to keep the program running until the user types `exit` or encounters a network error when trying to send a message. After `inputMessages()` has returned, we cancel the execution of the `outputMessages()` function, and `close` the client.

Until this happens, both input and output can happily happen concurrently, with new messages being received while the client sits idle, and the option to start composing a new message at any point.

### Let's give it a try!

We have now finished implementing our WebSocket-based chat client with Kotlin and Ktor. To celebrate our success, let's give it a try! With the chat server running, start some instances of the chat client using the play button, and talk to yourself! Even if you send multiple messages right after each other, they should be correctly displayed on all connected clients.

You might still notice some smaller usability issues caused by the limitations of terminal input, like incoming messages overwriting messages which are currently being composed. Managing more complex terminal user interfaces is outside the scope of this tutorial, though, and as such, left as an exercise to the reader 😉.

We have included the final state of the client chat application in the [codeSnippets](https://github.com/ktorio/ktor-documentation/tree/main/codeSnippets) project: [tutorial-websockets-client](https://github.com/ktorio/ktor-documentation/tree/main/codeSnippets/snippets/tutorial-websockets-client).

![App in action](app_in_action.png){animated="true" width="674"}

That's it for this tutorial on WebSockets with Ktor – time to congratulate yourself for building a whole application! If you're looking for some inspiration of where to take this project next, as well as related materials, continue to the next section.



## What's next

Congratulations on finishing this tutorial on creating a chat application using Kotlin, Ktor & WebSockets. We now have a basic command-line application which allows multiple clients to have a conversation over the network in a shared chat.

### Feature requests

At this point, we have implemented the absolute basics for a chat service, both on the client and server side. If you want to, you can keep expanding on this project. To get you started, here are a few ideas of how to improve the application, in no particular order:

- **Nicer UI!** So far, the client's user interface is very rudimentary, with only text input and output. If you're feeling adventurous, you can pick up a framework like [TornadoFX](https://tornadofx.io/), [Compose for Desktop](https://www.jetbrains.com/lp/compose/), or other, and try implementing a fancy user interface for the chat.
- **Mobile app!** The Ktor client libraries are also available for mobile applications. Feel free to try integrating what you have learned in this tutorial in the context of an Android application, and build the next big mobile chat product!
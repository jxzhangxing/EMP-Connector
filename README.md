# emp-connector Example
A simplified connector example to the Enterprise Messaging Platform.

This example connector provides support for SSL, HTTP proxies and supports both the long polling and websocket
streaming transports.  Easy subscription management and full support for event replay is provided.

## Disclaimer
Please note that this repository is example code and is not supported by Salesforce.  This code has not been rigorously tested nor performance tested for throughput and scale.

This code is provided as an example only.  The underlying [CometD library](https://cometd.org/) is what provides the meat here, as EMP Connector is a thin wrapper around this library.
___

## Example Usage


    // Replay from the start of the event window - may be any valid replayFrom position in the event stream
    long replayFrom = EmpConnector.REPLAY_FROM_EARLIEST;

    // get parameters from login
    BayeuxParameters params = login("foo@bar.com", "password");

    // The event consumer
    Consumer<Map<String, Object>> consumer = event -> System.out.println(String.format("Received:\n%s", event));

    // The EMP connector
    EmpConnector connector = new EmpConnector(params);

    // Wait for handshake with Streaming API
    connector.start().get(5, TimeUnit.SECONDS);

    // Subscribe to a channel
    // Block and wait for the subscription to succeed for 5 seconds
    TopicSubscription subscription = connector.subscribe("<Channel>", replayFrom, consumer).get(5, TimeUnit.SECONDS);

    // Here's our subscription
    System.out.println(String.format("Subscribed: %s", subscription));

    // Cancel a subscription
    subscription.cancel();

    // Stop the connector
    connector.stop();

See [LoginExample.java](src/main/java/com/salesforce/emp/connector/example/LoginExample.java) for full example.

## Example Classes
Several example classes are provided to subscribe to a channel. All classes contain a `main` function that starts the tool. All examples authenticate to Salesforce and subscribe to a channel. Some examples use a different authentication mechanism or provide verbose logging.

### `LoginExample`
The `LoginExample` class is the default class that EMP Connector executes. This class authenticates to your production Salesforce org using your Salesforce username and password.

### `DevLoginExample`
The `DevLoginExample` class accepts a custom login URL, such as a sandbox instance (https://test.salesforce.com). Also, `DevLoginExample` logs to the console the Bayeux connection messages received on the `/meta` channels, such as `/meta/handshake` and `/meta/connect`.

### `BearerTokenExample`
The `BearerTokenExample` class uses the OAuth bearer token authentication and accepts an access token.

## Build and Execute EMP Connector
After cloning the project, build EMP Connector using Maven:
`$ mvn clean package`

The build generates the jar file in the target subfolder.

To run EMP Connector using the `LoginExample` class with username and password authentication, use this command.

`$ java -jar target/emp-connector-0.0.1-SNAPSHOT-phat.jar <username> <password> <channel> [optional_replay_id]`

To run EMP Connector using the `DevLoginExample` class with username and password authentication, use this command.

`$ java -classpath target/emp-connector-0.0.1-SNAPSHOT-phat.jar com.salesforce.emp.connector.example.DevLoginExample <login_URL> <username> <password> <channel> [optional_replay_id]`

To run EMP Connector using an OAuth access token, use this command.

`$ java -classpath target/emp-connector-0.0.1-SNAPSHOT-phat.jar com.salesforce.emp.connector.example.BearerTokenExample <instance_URL> <token> <channel> [optional_replay_id]`

The last parameter is the replay ID, which is the position in the stream from which you want to receive event messages. This parameter is optional. If not specified, EMP Connector fetches events starting from the earliest retained event message (-2 option). For more information, see [Message Durability](https://developer.salesforce.com/docs/atlas.en-us.api_streaming.meta/api_streaming/using_streaming_api_durability.htm).

## Subscription Filtering for PushTopic Channels
If you subscribe to a PushTopic channel with a filter, enclose the entire channel and filter information within quotes on the command line. Do not use single quotes around field values. Otherwise, EMP Connector doesn't work properly. For example, this command line uses filters on the TestAccount PushTopic.

`$ java -jar target/emp-connector-0.0.1-SNAPSHOT-phat.jar <username> <password> "/topic/TestAccount?Type=Technology Partner&Phone=(415) 555-1212"`

Only Pushtopic events support filtering. For more information, see [Filtered Subscriptions](https://developer.salesforce.com/docs/atlas.en-us.api_streaming.meta/api_streaming/using_filtered_subscriptions.htm).

## Debug Logging of Bayeux Messages
The [LoggingListener](src/main/java/com/salesforce/emp/connector/example/LoggingListener.java) class provides debug logging output of Bayeux messages received on the meta channels, such as `/meta/handshake` and `/meta/connect`. Each message is logged to the console with a timestamp, a "Success" prefix or a "Failure" prefix depending on whether the operation was successful or not, and then the body of the Bayeux message. For example, this log is for a handshake message.

    [2018-01-19 10:54:12.701] Success:[/meta/handshake]
    {ext={replay=true, payload.format=true}, minimumVersion=1.0, clientId=cn2vei6rz2pa01gqqvungzlppy,
        supportedConnectionTypes=[Ljava.lang.Object;@6e2ce7d1,
        channel=/meta/handshake, id=1, version=1.0, successful=true}

To add logging support to your connection, first create an instance of the `LoggingListener` class. The `LoggingListener` constructor accepts two boolean arguments that specify whether to log success and failure messages. Next, call the `EmpConnector.addListener()` method for each meta channel to add logging for and pass in the channel and the `LoggingListener` instance. This example adds logging for multiple channels.

    LoggingListener loggingListener = new LoggingListener(true, true);
    connector.addListener(META_HANDSHAKE, loggingListener)
             .addListener(META_CONNECT, loggingListener)
             .addListener(META_DISCONNECT, loggingListener)
             .addListener(META_SUBSCRIBE, loggingListener)
             .addListener(META_UNSUBSCRIBE, loggingListener);

The [DevLoginExample](src/main/java/com/salesforce/emp/connector/example/DevLoginExample.java) class uses `LoggingListener` to log the messages received.

## Reauthentication
Authentication becomes invalid when a Salesforce session is invalidated or an access token is revoked. EMP connector listens to `401::Authentication invalid` error messages that Streaming API sends when the authentication is no longer valid. To reauthenticate after a 401 error is received, call the `EmpConnector.setBearerTokenProvider()` method, which accepts a function that reauthenticates and returns a new session ID or access token.

    // Define the bearer token function
    Function<Boolean, String> bearerTokenProvider = (Boolean reAuth) -> {
      ...
    }
    // Set the bearer token function
    connector.setBearerTokenProvider(bearerTokenProvider);

For a full example, see [LoginExample.java](src/main/java/com/salesforce/emp/connector/example/LoginExample.java).

## Documentation
For more information about the components of the EMP Connector and a walkthrough, see the [Java Client Example](https://developer.salesforce.com/docs/atlas.en-us.api_streaming.meta/api_streaming/code_sample_java_client_intro.htm)
 in the *Streaming API Developer Guide*.

## TrailHead
To get a local copy of the EMP-Connector GitHub repository:

$ git clone  https://github.com/forcedotcom/EMP-Connector.git

To build the EMP-Connector tool:

$ cd EMP-Connector

$ mvn clean package

The mvn command generated JAR files in the target folder in the EMP-Connector project directory. The generated JAR file,  target/emp-connector-0.0.1-SNAPSHOT-phat.jar, includes the connector and the LoginExample functionality. The JAR contains all the dependencies for the connector, so you don’t have to download them separately.

To run EMP Connector against your Developer Edition or Trailhead Playground org, type this command and supply your Trailhead Playground org or Salesforce org login credentials and the channel to subscribe to. In this example, the channel is /data/Employee__ChangeEvent.

$ java -jar target/emp-connector-0.0.1-SNAPSHOT-phat.jar  username password /data/Employee__ChangeEvent

Once you subscribe the EMP-Connector tool to the channel for Employee__c, any changes to Employee records or creation of new records generates notifications that the tool prints to the console.

 ### Console Example
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
Subscribed: Subscription [/data/Employee__ChangeEvent:-2]

Received:
{"schema":"-pszPCNGMHqUPU1ftkjxEA","payload":{"LastModifiedDate":"2020-08-06T06:22:19.000Z","OwnerId":"0057F000001eSTfQAM","CreatedById":"0057F000001eSTfQAM","ChangeEventHeader":{"commitNumber":10915705508285,"commitUser":"0057F000001eSTfQAM","sequenceNumber":1,"entityName":"Employee__c","changeType":"CREATE","changedFields":[],"changeOrigin":"com/salesforce/api/soap/49.0;client=SfdcInternalAPI/","transactionKey":"00039547-b906-1e54-d709-ebbf7a6f2777","commitTimestamp":1596694939000,"recordIds":["a097F00000XpKo9QAF"]},"CreatedDate":"2020-08-06T06:22:19.000Z","First_Name__c":"Patricia","LastModifiedById":"0057F000001eSTfQAM","Name":"e-100","Last_Name__c":"Smith"},"event":{"replayId":1480821}}

Received:
{"schema":"-pszPCNGMHqUPU1ftkjxEA","payload":{"LastModifiedDate":"2020-08-06T06:24:43.000Z","ChangeEventHeader":{"commitNumber":10915706370983,"commitUser":"0057F000001eSTfQAM","sequenceNumber":1,"entityName":"Employee__c","changeType":"UPDATE","changedFields":["LastModifiedDate","Tenure__c"],"changeOrigin":"com/salesforce/api/soap/49.0;client=SfdcInternalAPI/","transactionKey":"00039569-53f3-0a03-743e-46538797bf9f","commitTimestamp":1596695083000,"recordIds":["a097F00000XpKo9QAF"]},"Tenure__c":3.0},"event":{"replayId":1480822}}

Received:
{"schema":"-pszPCNGMHqUPU1ftkjxEA","payload":{"ChangeEventHeader":{"commitNumber":10915706812874,"commitUser":"0057F000001eSTfQAM","sequenceNumber":1,"entityName":"Employee__c","changeType":"DELETE","changedFields":[],"changeOrigin":"com/salesforce/api/soap/49.0;client=SfdcInternalAPI/","transactionKey":"0003957a-1856-5435-a67c-7f7290cd9e7c","commitTimestamp":1596695156000,"recordIds":["a097F00000XpKo9QAF"]}},"event":{"replayId":1480823}}

Received:
{"schema":"-pszPCNGMHqUPU1ftkjxEA","payload":{"LastModifiedDate":"2020-08-06T06:26:38.000Z","OwnerId":"0057F000001eSTfQAM","CreatedById":"0057F000001eSTfQAM","ChangeEventHeader":{"commitNumber":10915707071000,"commitUser":"0057F000001eSTfQAM","sequenceNumber":1,"entityName":"Employee__c","changeType":"UNDELETE","changedFields":[],"changeOrigin":"com/salesforce/api/soap/49.0;client=devconsole","transactionKey":"0003d0d0-0de6-ffd4-c2e9-d91720d8611a","commitTimestamp":1596695198000,"recordIds":["a097F00000XpKo9QAF"]},"CreatedDate":"2020-08-06T06:22:19.000Z","First_Name__c":"Patricia","LastModifiedById":"0057F000001eSTfQAM","Tenure__c":3.0,"Name":"e-100","Last_Name__c":"Smith"},"event":{"replayId":1480824}}

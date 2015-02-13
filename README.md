I# Blynk server
Is Netty based Java server responsible for message forwarding between mobile application and any hardware (Arduino, Raspberry Pi for now).
Please read more precise project description [here](https://www.kickstarter.com/projects/167134865/blynk-build-an-app-for-your-arduino-project-in-5-m/description).

# Requirements
Java 8 required. (OpenJDK, Oracle)

# Protocol messages

Every message consists of 2 parts.

+ Header :
    + Protocol command (1 byte);
    + MessageId (2 bytes);
    + Body message length (2 bytes);

+ Body : string (could be up to 2^15 bytes).

Scheme:

	            	BEFORE DECODE (8 bytes)
	+------------------+------------+---------+------------------------+
	|       1 byte     |  2 bytes   | 2 bytes |    Variable length     |
	+------------------+------------+---------+------------------------+
	| Protocol Command |  MessageId |  Length |  Message body (String) |
	|       0x01       |   0x000A   |  0x0003 |         "1 2"          |
	+------------------+------------+---------+------------------------+
	                                          |        3 bytes         |
    	                                      +------------------------+

So message is always "1 byte + 2 bytes + 2 bytes + messageBody.length".

### Command field
Unsigned byte.
This is 1 byte field responsible for storing [command code](https://bitbucket.org/theblynk/blynk-server/src/a3861b0e9bcb9823bbb6dd2722500c55e197bbe6/common/src/main/java/cc/blynk/common/enums/Command.java?at=master) from client, like login, ping, etc...

### Message Id field
Unsigned short.
Is 2 bytes field for defining unique message identifier. It’s used in order to distinguish how to manage responses from hardware on mobile client. Message ID field should be generated on client’s side.
Any ‘read’ protocol command should always have same messageId for the same widget. Let's say, you have a Graph_1 widget which is configured to read data from some analog pin.
After you reconfigured Graph_1 to read another pin, load command will still look the same, and messageID will be an ID of the widget to display results at.

### Length field
Unsigned short.
Is 2 bytes field for defining body length. Could be 0 if body is empty or missing.



#### Protocol command codes

		0 - response; After every message client sends to server it retrieves response message back (exception commands are: LoadProfile, GetToken, Ping commands).
        1 - register; Must have 2 space-separated params as a content field (username and password) : “username@example.com UserPassword”
        2 - login:
            a) Mobile client must send send 2 space-separated parameters as a content field (username and password) : "username@example.com UserPassword"
            b) Hardware client must send 1 parameter, which is user Authentication Token : "6a7a3151cb044cd893a92033dd65f655"
        3 - save profile; Must have 1 parameter as content string : "{...}"
        4 - load profile; Don't have any parameters
        5 - getToken; Must have 1 int parameter, Dashboard ID : "1". ACCEPTED DASH_ID RANGE IS [1..100];
        6 - ping; Sends request from client to server, then from server to hardware, than back to server and back to the client.
        12 - tweet; Sends tweet request from hardware to server. 140 chars max. 
        20 - hardware; Command for hardware. Every Widget forms it's own body message for hardware command.


#### Hardware command body forming rules
//todo


## Response Codes
For every command client sends to the server it will receive a response back.
For commands (register, login, saveProfile, hardware) that doesn't request any data back - 'response' (command field 0x00) message is returned.
For commands (loadProfile, getToken, ping) that request data back - message will be returned with same command code. In case you sent 'loadProfile' you will receive 'loadProfile' command back with filled body.
[Here is the class with all of the codes](https://bitbucket.org/theblynk/blynk-server/src/a3861b0e9bcb9823bbb6dd2722500c55e197bbe6/common/src/main/java/cc/blynk/common/enums/Response.java?at=master)
Response message structure:

	    	       BEFORE DECODE
	+------------------+------------+----------------+
	|       1 byte     |  2 bytes   |     2 bytes    |
	+------------------+------------+----------------+
	| Protocol Command |  MessageId |  Response code |
	|       0x00       |   0x000A   |      0x0001    |
	+------------------+------------+----------------+
	|               always 5 bytes                   |
	+------------------------------------------------+

    200 - message was successfully processed/passed to the server
    
    2 - command is badly formed, check syntax and passed params
    3 - user is not registered
    4 - user with this name has been registered already
    5 - user haven’t made login command
    6 - user is not allowed to perform this operation (most probably is not logged or socket has been closed)
    7 - arduino board not in network
    8 - command is not supported
    9 - token is not valid
    10 - server error
    11 - user have already logged in. Happens in cases when same user tries to login more than one time.
    12 - tweet exception, exception occurred during posting request to Twitter could be in case messages are the same in a row;
    13 - tweet body invalid exception; body is empty or larger than 140 chars;
    14 - user has no access token.
    500 - server error. something went wrong on server

## User Profile JSON structure
	{ "dashBoards" : 
		[ 
			{
			 "id":1, "name":"My Dashboard", "isActive":true, "timestamp":333333,
			 "widgets"  : [...], 
			 "settings" : {"boardType":"UNO", ..., "someParam":"someValue"}
			}
		],
		"twitterAccessToken" : {
		    "token" : "USER_TOKEN",
		    "tokenSecret" : "USER_SECRET"
		}
	}

## Widgets JSON structure

	Button				: {"id":1, "x":1, "y":1, "dashBoardId":1, "label":"Some Text", "type":"BUTTON",         "pinType":"NONE", "pin":13, "value":"1"   } -- sends HIGH on digital pin 13. Possible values 1|0.
	Toggle Button ON	: {"id":1, "x":1, "y":1, "dashBoardId":1, "label":"Some Text", "type":"TOGGLE_BUTTON",  "pinType":"DIGITAL", "pin":18, "value":"1", "state":"ON"} -- sends 1 on digital pin 18
	Toggle Button OFF	: {"id":1, "x":1, "y":1, "dashBoardId":1, "label":"Some Text", "type":"TOGGLE_BUTTON",  "pinType":"VIRTUAL", "pin":18, "value":"0", "state":"OFF"} -- sends 0 on digital pin 18
	Slider				: {"id":1, "x":1, "y":1, "dashBoardId":1, "label":"Some Text", "type":"SLIDER",         "pinType":"ANALOG",  "pin":18, "value":"244" } -- sends 244 on analog pin 18. Possible values -9999 to 9999
	Timer				: {"id":1, "x":1, "y":1, "dashBoardId":1, "label":"Some Text", "type":"TIMER",          "pinType":"DIGITAL", "pin":13, "startTime" : 1111111111, "stopInterval" : 111111111} -- startTime is time in UTC when to start timer (milliseconds are ignored), stopInterval interval in milliseconds after which stop timer.

	//pin reading widgets
	LED					: {"id":1, "x":1, "y":1, "dashBoardId":1, "label":"Some Text", "type":"LED",            "pinType":"DIGITAL", "pin":10} - sends READ pin to server
	Digit Display		: {"id":1, "x":1, "y":1, "dashBoardId":1, "label":"Some Text", "type":"DIGIT4_DISPLAY", "pinType":"DIGITAL", "pin":10} - sends READ pin to server
	Graph				: {"id":1, "x":1, "y":1, "dashBoardId":1, "label":"Some Text", "type":"GRAPH",          "pinType":"DIGITAL", "pin":10, "readingFrequency":1000} - sends READ pin to server. Frequency in microseconds

## Commands order processing
Server guarantees that commands will be processed in same order in which they were send.

## GETTING STARTED

+ Run the server

        java -jar server.jar -port 8080

+ Run the client (simulates smartphone client)

        java -jar client.jar -host localhost -port 8080

+ In this client: register new user and/or login with the same credentials

        register username@example.com UserPassword
        login username@example.com UserPassword

+ Get the token for hardware (e.g Arduino)

        getToken 1

   	You will get server response similar to this:

    	00:05:18.086 INFO  - Sending : Message{messageId=30825, command=5, body='1'}
    	00:05:18.100 INFO  - Getting : Message{messageId=30825, command=5, body='33bcbe756b994a6768494d55d1543c74'}
Where `33bcbe756b994a6768494d55d1543c74` is your Auth Token.

+ Start another client (simulates hardware (e.g Arduino)) and use received token to login

    	java -jar client.jar -host localhost -port 8080
    	login 33bcbe756b994a6768494d55d1543c74
   

You can run as many clients as you want.

Clients with the same credentials and Auth Token are grouped in one ChatRoom/Group and can send messages to each other.
All client commands are human-friendly, so you don't have to remember the codes. Examples:

    	digitalWrite 1 1
    	digitalRead 1
    	analogWrite 1 1
    	analogRead 1
    	virtualWrite 1 1
    	virtualRead 1

Registered users are stored locally in TMP dir of your system in file "user.db". So after the restart you don't have to register again.


## Local server setup

### Behind wifi router
In case you start Blynk server behind wifi-router and want it to be accessible from internet you have to add port-forwarding rule
on your router. This is required in order all of the request which come to the router are forwarded to Blynk server within the local network of your router.
Im my router it look like this: {image here}

## Licensing
[MIT license](https://bitbucket.org/theblynk/blynk-server/src/c1b06bca3183aba9ea9ed1fad37b856d25cd8a10/license.txt?at=master)
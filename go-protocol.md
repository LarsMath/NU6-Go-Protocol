
# Protocol Go NU6

## Introduction
The following document explains protocol rules for communication between server and client in a game of Go. This protocol is part of the module 1 project of NU6.

The protocol is designed to cater to the different extensions that clients may implement. This is achieved by splitting the protocol in parts. The most important features to run a game of Go smoothly are part of the CORE protocol and its commands have the <code>CORE.</code> prefix. Any commands in the extensions supported by this protocol will have the <CODE>[Extension-name].</code> prefix.

In the commands the parameters in square brackets <code>[parameter]</code> are mandatory. The code <code>[CHOICE1\|CHOICE2]</code> means a choice between these two has to be made and given as parameter in that spot. A hekkie <code>[#number]</code> indicates an integer should be given as parameter. When a parameter is optional it is given with triangle brackets as <code>\<optional-parameter></code>. If an optional parameter may be given multiple times a <code>*</code> is added. In all cases, the delimiter <code>\~</code> is used. Whenever the client does not adhere to this protocol the server will send a <code>CORE.ERROR\~PROTOCOL\~[clients-faulty-message]</code> back.

## CORE
First we present an overview of the CORE commands, below we go into detail

##### Overview commands Client

|Command|Parameters|  
|---------|---------------|  
|<code>CORE.HELLO\~[name]\~\<extensions>*</code>|name of the client<br>extensions supported by the client|  
|<code>CORE.NEWGAME\~\<#boardsize></code>|size of the board on which client wants to play|  
|<code>CORE.MOVE\~[#coordinate]</code>|coordinate of the move client wants to play|  
|<code>CORE.PASS</code>|-|  
|<code>CORE.SURRENDER</code>|-|  
|<code>CORE.BYE</code>|-|  

##### Overview commands Server
|Command|Parameters|  
|---------|---------------|  
|<code>CORE.HELLO</code>|-|
|<code>CORE.MATCH\~[BLACK\|WHITE]\~[#boardsize]\~[name]</code>|color in which the client plays<br>size of the board in this match<br>name of the opponent|
|<code>CORE.MOVE\~[#coordinate]</code>|coordinate of last play|
|<code>CORE.PASS</code>|-|
|<code>CORE.ERROR\~[PATIENCE\|ILLEGAL\|PROTOCOL\|NAMETAKEN]<br>\~\<additional-info></code>|errortype<br>additional info depending on errortype|
|<code>CORE.GAMEOVER\~[BLACK\|WHITE\|DRAW]<br>\~[#points-black]\~[#points-white]<br>\~\<SURRENDER\|DISCONNECT></code>|winner/draw<br>points of black<br>points of white<br>reason of game over if not game end
|<code>CORE.BYE\~👋</code>|Good luck parsing this

### Handshake
In order to make sure that communication is set properly between the server and the client a handshake will be executed at the start of communication. This is started by the client sending the <code>HELLO</code> command. When the server receives this it should either respond back with the <code>HELLO</code> command or give an error if the name chosen by the client is already taken. When the user wants to enable extensions these should be added to the initial hello. Two possible handshakes can look like this:

<code>Client: CORE.HELLO\~AlphaGo</code> \
<code>Server: CORE.HELLO</code>

or

<code>Client: CORE.HELLO\~AlphaGo</code>\
<code>Server: CORE.ERROR\~NAMETAKEN</code>\
<code>Client: CORE.HELLO\~AlphaGoZERO</code>\
<code>Server: CORE.HELLO</code>

Whenever the handshake is accepted, that is, the server sent <code>HELLO</code> back. After that <code>HELLO</code> is not a viable command for the client anymore and the server will respond with a <code>CORE.ERROR\~PROTOCOL</code> message.

### Starting a game
In order to start a game a client has to let the server know it wants to play a game and on which board size. It does this by sending the <code>NEWGAME</code> command. The board is always a square, therefore only one integer is sent as boardsize. When no board size is chosen the standard of 19x19 is chosen. When a match between players is found the server will start a match and notify the clients using the <code>MATCH</code> command. This command passes the color the client plays in as well as the name of the opponent and the boardsize such that no confusion can arise. A possible communication:

<code>Client1: CORE.NEWGAME\~9</code>\
<code>Client2: CORE.NEWGAME\~9</code>\
<code>Server --> Client1: CORE.MATCH\~BLACK\~9\~Client2</code>\
<code>Server --> Client2: CORE.MATCH\~WHITE\~9\~Client1</code>

or

<code>Client1: CORE.NEWGAME</code>\
<code>Client2: CORE.NEWGAME\~9</code>\
<code>Client3: CORE.NEWGAME\~19</code>\
<code>Server --> Client1: CORE.MATCH\~BLACK\~19\~Client3</code>\
<code>Server --> Client3: CORE.MATCH\~WHITE\~19\~Client1</code>

Note: Client2 is not in a game as there is no opponent with the same boardsize as Client1 has "chosen" 19.

### Ingame
During the game clients can send their play when it is their turn. Clients should adhere to the rule but this is also checked server-side. Clients can communicate their move via <code>MOVE</code> or <code>PASS</code>. When the server deems the move to be legal then it propagates this move back to all players in this game.

<code>Client1: CORE.MOVE\~42</code>\
<code>Server --> Client1, Client2: CORE.MOVE\~42</code>

The following coordinate system is used:
|3|x|3|
|---|---|---|
|0|1|2|
|3|4|5|    
|6|7|8|

||5|x|5||
|---|---|---|---|---|
|0|  1|  2|  3|  4 |   
|5|  6 | 7 | 8|  9  |  
|10| 11| 12 |13 |14  |  
|15| 16| 17 |18| 19 |   
|20| 21| 22 |23| 24|

When the move is illegal the server will respond that it is and will give an indication what rule is broken. These are given by

<code>Server: CORE.ERROR\~ILLEGAL\~OCCUPIED</code>\
<code>Server: CORE.ERROR\~ILLEGAL\~KO</code>\
<code>Server: CORE.ERROR\~ILLEGAL\~OUTOFBOUNDS</code>

and mean that a client wanted to play a stone at an occupied intersection, wanted to break the ko rule or wanted to put a stone off the edge of the board (sponsored by the flat-go-society).

When a client sends a <code>MOVE</code> or <code>PASS</code> command without them being on turn the server will respond with

<code>Server: CORE.ERROR\~PATIENCE</code>

When the client wants to be freed of this agony it can send a <code>SURRENDER</code> command to surrend the current game and let the opponent win.

When the game is over the server will notify the clients of this game by sending a <code>GAMEOVER</code> message. Included in this message is the winner (or draw if so) and the amount of points of each player. When the game has ended by non-regular means such as a surrender or disconnect it will be included in the message. For example

<code>Client1: CORE.PASS</code>\
<code>Client2: CORE.PASS</code>\
<code>Server --> Client1, Client2: GAMEOVER\~BLACK\~181\~180</code>

or

<code>Client1: CORE.MOVE\~37</code>\
<code>Client2: CORE.SURRENDER</code>\
<code>Server --> Client1, Client2: GAMEOVER\~BLACK\~19\~0\~SURRENDER</code>

Now the players can either look for a new game by using the <code>NEWGAME</code> command or sit back and enjoy their sweet victory.

### Saying goodbyes
When a client is done playing the game it can send the <code>CORE.BYE</code> command to the server to let it know he is gone. When done in game the server will let the other player win by disconnect. The server will probably not beg the client to stay.

## Supported Extensions
All protocol messages start with appropriate extension command.
- CHAT

## CHAT

For these features the <code>CHAT</code> extension should be included in the extensions of the <code>HELLO</code> command.

##### Overview commands Client

|Command|Parameters|  
|---------|---------------|  
|<code>CHAT.MESSAGE\~[message]\~\<recipient></code>|message to be sent<br>name of the recipient|  
|<code>CHAT.GETLOBBY</code>|-|  

##### Overview commands Server
|Command|Parameters|  
|---------|---------------|  
|<code>CHAT.MESSAGE\~[message]\~\<sender></code>|message<br>name of the recipient|  
|<code>CHAT.LOBBY\~\<name>*</code>|list of the names of clients currently in the lobby separated by <code>\~</code>|  
|<code>CHAT.ENTERED\~[name]|name of the client|
|<code>CHAT.LEFT\~[name]|name of the client|
|<code>CHAT.ERROR\~UNKNOWNRECIPIENT</code>|-|
|<code>CHAT.ERROR\~CHATNOTSUPPORTED</code>|-|

### Sending and receiving messages
A client that wants to send messages to another client can use the <code>MESSAGE</code> command followed by the message and the recipient. If no recipient is added then the message will be sent to all clients. The server will forward it to the specified client. If this client does not support chat or the name of the client is not on the server then an error is sent back to the sending client. For example:

<code>Client: CHAT.MESSAGE\~Hello Hello\~World</code>\
<code>Server --> World: CHAT.MESSAGE\~Hello Hello\~Client1</code>

or

<code>Client1: CORE.HELLO\~Caesar</code>\
<code>Server --> Caesar: CORE.HELLO</code>\
<code>Client2: CORE.HELLO\~Informant\~CHAT</code>\
<code>Server --> Informant: CORE.HELLO</code>\
<code>Informant: CHAT.MESSAGE\~BRUTUS will betray you!\~Caesar</code>\
<code>Server --> Informant: CHAT.ERROR\~CHATNOTSUPPORTED</code>

### Available Clients
To get a list of other clients a client can send <code>CHAT.GETLOBBY</code> to the server. The server will respond with the clients currently on the server. For example:

<code>Client: CHAT.GETLOBBY</code>\
<code>Server: CHAT.LOBBY\~ANNE\~BASTIENNE\~ELINE\~GERARD\~LARS\~RUBEN</code>

The server will also send messages to the clients on the server that use the CHAT extension if new clients entered or left using the <code>ENTERED</code> and <code>LEFT</code> statements. This is done when clients enter by a successful <code>HELLO</code> or a <code>BYE</code> or disconnect.

# chromie

## Concepts:

All users have an avatar, on the server and on their client which are the broker
between both, the server can cache avatars in memory, store them in memcache for
distributed avatar servers to access on request, or whatever, thats not our 
descission (although there would be some common patterns)

Avatars exist on the server and the client and expose services to the other side

IDEA: The could also have 'live attributes' like a Model that can be set/watched at
each end.

### on the client:

The idea here is that the browser can use whatever Model/Collection/manual whatever
it likes, ember, chromie models etc, they talk to services via their api. you
could use this directly with jquery if you wanted to o.O

### on the server:

Services can be local to the avatar server and invoked directly, or ideally be
stand-alone services which can go up or down, and the avatar server connects to
them via socket.io. If a service goes down (or needs to be taken offline to fix
a critical bug) the avatar on the server can: avatar.disableService("chat"); 
and the client can listen for $av.chat.onDisable(func) and handle closing chat,
this is entirely optional but allows developers to build an entirely modular 
service that can have components go on and offline and still function. 

Services on the server are added to an avatar (perhaps based on permissions if
that is appropriate for the application) and contain a mapping of available 
functions that can be called. 

IDEA: and optionally a URL/Protocol pair if the client should connect directly 
to that service rather than through the avatar server (just gives more 
flexibility) 

### example server setup:

```javascript
app = connect();
portal = chromie.Portal("poll" /* or "socket.io" */);
portal.credentialCheckers = [checkCookie, checkPassword];
portal.avatarFactory = function(user, avatar) {
    avatar.addService("forum", require("forumService").service);
    avatar.addService("chat", require("chatService").service);

    // if avatars worked like chromie models we culd have 'live' attributes at
    // both ends like twisted PB RemoteReference..  
    avatar.set({username : user.username, email : user.email});
    avatar.watch('email', sendEmailConfirmation);
};
app.use(portal);
```

### example client setup:

```javascript
var connectionCheck = function(av) {
    if(!av.authenticated) {
        //Show auth dialog
        av.authenticate({/* credentials */}, connectionCheck);
    } else {
        // app logic here :)        
    }
};

var $av = chromie.ClientAvatar("http://example.com/service");
$av.onConnect(connectionCheck);
$av.onDisconnect(connectionCheck);
$av.addService("chat", {sendMessage : sendMessage, sendFile : sendFile});
$av.addService("friends", {newFriend : newFriend});
$av.connect();
```

client can: 

```javascript
$av.set({username : "Pomke"}, function(r) {...}); 
$av.forum.createPost({ title : 'test post', body : 'something'}, function(r){});
```

server can:

```javascript
avatar.friends.newFriend(args, optionalCallback);
avatar.set({username : "Melanie"});
```


## transports

There would be two transports, Poll and socket.io:

Poll is the default, when the server wants to invoke something on the client, 
the command is queued on the avatar (serverside) and returned on the next client
request, along with whatever the client request was. So the server and client 
avatars are actually a construct which all map to one function, 'remoteCall' 
which takes the arguments and sends them to the other side with "service:call" 
as the key, these can be batched up:  [{f:"friends:newFriend",args:{...},...]

When the client sends a request to the server, either as a normal request or 
perhaps a heartbeat request (if the developer wanted to write one, none by 
default although we could write a convenience for adding them in), they will
recieve the response to their query and possibly some client side commands that
were queued up. 

The other option is socket.io which would basically use socket.io as the 
transport and it would handle all buffering, the developers would have to take
the appropriate steps to load-balance connections etc..







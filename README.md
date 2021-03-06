Loowy
=====

Lua WAMP (WebSocket Application Messaging Protocol) client implementation on top of lua-websockets and lib-ev. 

Table of Contents
=================

* [Description](#description)
* [Usage example](#usage-example)
* [Installation](#installation)
* [Dependencies](#dependencies)
* [Loowy client instance methods](#loowy-client-instance-methods)
    * [options](#optionsopts)
    * [getOpStatus](#getopstatus)
    * [getSessionId](#getsessionid)
    * [connect](#connecturl)
    * [disconnect](#disconnect)
    * [abort](#abort)
    * [Challenge Response Authentication](#challenge-response-authentication)
    * [subscribe](#subscribetopicuri-callbacks)
    * [unsubscribe](#unsubscribetopicuri-callbacks)
    * [publish](#publishtopicuri-payload-callbacks-advancedoptions)
    * [call](#calltopicuri-payload-callbacks-advancedoptions)
    * [cancel](#cancelreqid-callbacks-advancedoptions)
    * [register](#registertopicuri-callbacks)
    * [unregister](#unregistertopicuri-callbacks)
* [Copyright and License](#copyright-and-license)
* [See Also](#see-also)

Description
===========

Loowy implements [WAMP][] v2 client specification.

Loowy supports next WAMP roles and features:

* Challenge Response Authentication (wampcra method)
* publisher:
    * subscriber blackwhite listing
    * publisher exclusion
    * publisher identification
* subscriber:
    * pattern-based subscription
    * publication trust levels
* caller:
    * caller identification
    * progressive call results
    * call canceling
    * call timeout
* callee:
    * caller identification
    * call trust levels
    * pattern-based registration
    * shared registration

Loowy supports JSON and msgpack serializers.

[Back to TOC](#table-of-contents)

Usage example
=============

For example usage, please see [test.lua](tests/test.lua) file.

[Back to TOC](#table-of-contents)

Installation
============

You can install Loowy via luarocks

```bash
> luarocks install loowy 
```

or simply put loowy/client.lua somewhere accessible by lua package.path. 
But in this case you also need to install dependencies.

**WARNING!** 
Loowy depends on [lua-websockets][]. But lua-websockets up to and including v2.2 doesn't contain necessary changes.
Please manually install lua-websockets from master branch and send message to 
[lua-websockets maintainer](https://github.com/lipp) to publish new release :)

[Back to TOC](#table-of-contents)

Dependencies
=============

Loowy depends on:

* [lua-websockets][]
* [lua-ev][]
* [rapidjson][]
* [lua-messagepack][]

also it uses [busted][] for testing.

[Back to TOC](#table-of-contents)

Loowy client instance methods
=============================

options([opts])
------------------------------------------

options() method can be called in two forms:

* without parameters it will return table with current options
* with one parameter as table it will set new options

Options keys description:

* **debug**. Default value: false. Enable to print some debugging info.
* **autoReconnect**. Default value: true. Enable autoreconnecting. In case of connection failure, 
Loowy will try to reconnect to WAMP server, and if you were subscribed to any topics,
or had registered some procedures, Loowy will resubscribe to that topics and reregister procedures.
* **reconnectInterval**. Default value: 2(s). Reconnection Interval in seconds.
* **maxRetries**. Default value: 25. Max reconnection attempts. After reaching this value [disconnect()](#disconnect) 
will be called.
* **transportEncoding**. Default value: json. Transport serializer to use. Supported 2 values: "json"/"msgpack".
* **realm**. Default value: nil. WAMP Realm to join on server. See WAMP spec for additional info.
* **helloCustomDetails**. Default value: nil. Custom attributes to send to router on hello.
* **authid**. Default value: nil. Authentication (user) id to use in challenge.
* **authmethods**. Default value: {}. Array of strings of supported authentication methods.
* **onChallenge**. Default value: nil. Callback function.
Is fired when wamp server requests authentication during session establishment.
This function receives two arguments: auth method and challenge details.
Function should return computed signature, based on challenge details.
See [Challenge Response Authentication](#challenge-response-authentication) section and [WAMP Spec CRA][] for more info.
* **onConnect**. Default value: nil. Callback function. Fired when connection to wamp server is established.
* **onClose**. Default value: nil. Callback function. Fired on closing connection to wamp server.
* **onError**. Default value: nil. Callback function. Fired on error in websocket communication.
* **onReconnect**. Default value: nil. Callback function. Fired every time on reconnection attempt.
* **onReconnectSuccess**. Default value: nil. Callback function. Fired every time when reconnection succeeded.

[Back to TOC](#table-of-contents)

getOpStatus()
------------------------------------------

Get the status of last operation.

This method returns table with 2 or 3 keys: code and description and possible request ID.
`code` is integer, and value > 0 means error.
`description` is a string description of code.
`reqId` is integer and may be useful in some cases (call canceling for example).

[Back to TOC](#table-of-contents)

getSessionId()
------------------------------------------

Get the WAMP Session ID.

[Back to TOC](#table-of-contents)

connect([wampurl])
------------------------------------------

Connect to WAMP router. 

Parameters:

* url - WAMP Server url (optional). Should be specified as URI. For example: ws://my-server/wamp

[Back to TOC](#table-of-contents)

disconnect()
------------------------------------------

Disconnect from WAMP router.

[Back to TOC](#table-of-contents)

abort()
------------------------------------------

Abort WAMP session establishment. Works only if websocket connection is established, 
but WAMP session establishment is in progress.

[Back to TOC](#table-of-contents)

Challenge Response Authentication
------------------------------------------

Loowy supports challenge response authentication. To use it you need to provide authid and onChallenge callback
as instance options. See example below:

```lua
local loowy = require 'loowy.client'

local client = loowy.new("ws://ws.wamp.server.url", {
    transportEncoding = 'json',
    realm = 'AppRealm',
    authid = 'user1',
    authmethods = { 'wampcra' },
    onChallenge = function (method, info)
        local hmac = require "resty.hmac"
        local hm = hmac:new("secret1")
        return hm:generate_signature("sha256", info.challenge)
    end
})
```

[Back to TOC](#table-of-contents)

subscribe(topicURI, callbacks[, advancedOptions])
------------------------------------------

Subscribe to a topic on a broker.

Parameters:

* **topicURI**. Required. A string that identifies the topic.
Must meet a WAMP Spec URI requirements.
* **callbacks**. If it is a function - it will be treated as published event callback or 
it can be hash table of callbacks:
    * **onSuccess**: will be called when subscription would be confirmed
    * **onError**: will be called if subscription would be aborted with one hash-table parameter with following attributes:
        * **error**: string error description
        * **details**: hash-table with some error details
    * **onEvent**:   will be called on receiving published event with one hash-table parameter with following attributes: 
        * **argsList**: array payload (may be omitted)
        * **argsDict**: object payload (may be omitted)
        * **details**: some publication options object. 
* **advancedOptions**. Optional parameters hash table. Must include any or all of the options:
    * **match**: string matching policy ("prefix"/"wildcard")

[Back to TOC](#table-of-contents)

unsubscribe(topicURI, callbacks)
------------------------------------------

Unsubscribe from topic.

Parameters:

* **topicURI**. Required. A string that identifies the topic.
Must meet a WAMP Spec URI requirements.
* **callbacks**. If it is a function - it will be treated as published event callback to remove
             or it can be hash table of callbacks:
    * **onSuccess**: will be called when unsubscription would be confirmed
    * **onError**: will be called if unsubscribe would be aborted with one hash-table parameter with following attributes:
        * **error**: string error description
        * **details**: hash-table with some error details
    * **onEvent**: published event callback instance to remove or it can be not specified, 
                   in this case all callbacks and subscription will be removed.

or it can be not specified, in this case all callbacks and subscription will be removed.

[Back to TOC](#table-of-contents)

publish(topicURI[, payload[, callbacks[, advancedOptions]]])
------------------------------------------

Publish event to topic.

Parameters:

* **topicURI**. Required. A string that identifies the topic.
Must meet a WAMP Spec URI requirements.
* **payload**. Publishing event data. Optional. May be any single value or array or hash-table object or null. Also it
is possible to pass array and object-like data simultaneously. In this case pass a hash-table with next attributes:
    * **argsList**: array payload (may be omitted)
    * **argsDict**: object payload (may be omitted)
* **callbacks**. Optional hash table of callbacks:
    * **onSuccess**: will be called when publishing would be confirmed
    * **onError**: will be called if publishing would be aborted with one hash-table parameter with following attributes:
        * **error**: string error description
        * **details**: hash-table with some error details
* **advancedOptions**. Optional parameters hash table. Must include any or all of the options:
    * **exclude**: integer|array WAMP session id(s) that won't receive a published event,
                 even though they may be subscribed
    * **exclude_authid**: string|array Authentication id(s) that won't receive
                        a published event, even though they may be subscribed
    * **exclude_authrole**: string|array Authentication role(s) that won't receive
                          a published event, even though they may be subscribed
    * **eligible**: integer|array WAMP session id(s) that are allowed to receive a published event
    * **eligible_authid**: string|array Authentication id(s) that are allowed to receive a published event
    * **eligible_authrole**: string|array Authentication role(s) that are allowed
                           to receive a published event
    * **exclude_me**: bool flag of receiving publishing event by initiator
                         (if it is subscribed to this topic)
    * **disclose_me**: bool flag of disclosure of publisher identity (its WAMP session ID)
                         to receivers of a published event 

[Back to TOC](#table-of-contents)

call(topicURI[, payload[, callbacks[, advancedOptions]]])
------------------------------------------

Remote Procedure Call.

Parameters:

* **topicURI**. Required. A string that identifies the remote procedure to be called.
Must meet a WAMP Spec URI requirements.
* **payload**. RPC data. Optional. May be any single value or array or hash-table object or null. Also it
is possible to pass array and object-like data simultaneously. In this case pass a hash-table with next attributes:
    * **argsList**: array payload (may be omitted)
    * **argsDict**: object payload (may be omitted)
* **callbacks**. If it is a function - it will be treated as result callback function
             or it can be hash table of callbacks:
    * **onSuccess**: will be called with result on successful call with one hash-table parameter with following attributes: 
        * **details**: hash-table with some additional details
        * **argsList**: optional array containing the original list of positional result
                        elements as returned by the _Callee_
        * **argsDict**: optional hash-table containing the original dictionary of keyword result
                        elements as returned by the _Callee_  
    * **onError**: will be called if invocation would be aborted with one hash-table parameter with following attributes:
        * **error**: string error description
        * **details**: hash-table with some error details
        * **argsList**: optional array containing the original error payload list as returned 
                        by the _Callee_ to the _Dealer_
        * **argsDict**: optional hash-table containing the original error
                        payload dictionary as returned by the _Callee_ to the _Dealer_
* **advancedOptions**. Optional parameters hash table. Must include any or all of the options:
    * **disclose_me**: bool flag of disclosure of Caller identity (WAMP session ID)
                        to endpoints of a routed call
    * **receive_progress**: bool flag for receiving progressive results. In this case onSuccess function
                        will be called every time on receiving result
    * **timeout**: integer timeout (in ms) for the call to finish 

[Back to TOC](#table-of-contents)

cancel(reqId[, callbacks[, advancedOptions]])
-----------------------------------------------

RPC invocation cancelling.

Parameters:

* **reqId**. Required. Request ID of RPC call that need to be canceled.
* **callbacks**. Optional. If it is a function - it will be called if successfully sent canceling message
            or it can be hash table of callbacks:
    * **onSuccess**: will be called if successfully sent canceling message 
    * **onError**: will be called if some error occurred 
* **advancedOptions**. Optional parameters hash table. Must include any or all of the options:
    * **mode**: string|one of the possible modes: "skip"/"kill"/"killnowait". Skip is default. 

[Back to TOC](#table-of-contents)

register(topicURI, callbacks[, advancedOptions])
------------------------------------------

RPC registration for invocation.

Parameters:

* **topicURI**. Required. A string that identifies the remote procedure to be called.
Must meet a WAMP Spec URI requirements.
* **callbacks**. Required. If it is a function - it will be treated as rpc itself
             or it can be hash table of callbacks:
    * **rpc**: registered procedure
    * **onSuccess**: will be called on successful registration
    * **onError**: will be called if registration would be aborted with one hash-table parameter with following attributes:
        * **error**: string error description
        * **details**: hash-table with some error details
* **advancedOptions**. Optional parameters hash table. Must include any or all of the options:
    * **match**: string matching policy ("prefix"/"wildcard")
    * **invoke**: string invocation policy ("single"/"roundrobin"/"random"/"first"/"last")

Registered PRC during invocation will receive one hash-table argument with following attributes:
 
* **argsList**: array payload (may be omitted)
* **argsDict**: object payload (may be omitted)
* **details**: some invocation options object. One attribute of interest in options is "receive_progress" (boolean), 
which indicates, that caller is willing to receive progressive results, if possible. Another one is "trustlevel", which 
indicates the call trust level, assigned by dealer (of course if it is configured accordingly).

RPC can return no result (undefined), or it must return an object with next attributes:

* **argsList**: array result or single value, (may be omitted)
* **argsDict**: object result payload (may be omitted)
* **options**: some result options object. Possible attribute of options is "progress": true, which
indicates, that it's a progressive result, so there will be more results in future. Be sure to unset "progress"
on last result message. 

Also it is possible to abort rpc processing and throw error with custom application specific data. 
This data will be passed to caller onError callback. 

Exception object with custom data may have next attributes:
* **uri**. String with custom error uri. Must meet a WAMP Spec URI requirements.
* **details**. Custom details dictionary object. The details object is used for the future extensibility, 
and used by the WAMP router. This object not passed to the client. For details see 
[WAMP specification 6.1](https://tools.ietf.org/html/draft-oberstet-hybi-tavendo-wamp-02#section-6.1)
* **argsList**. Custom arguments array-like table, this will be forwarded to the caller by the WAMP router's dealer 
role. Most cases this attribute is used to pass the human readable message to the client.
* **argsDict**. Custom arguments object-like table, this will be forwarded to the caller by the WAMP router's 
dealer role.

For more details see [WAMP specification 9.2.5](https://tools.ietf.org/html/draft-oberstet-hybi-tavendo-wamp-02#section-9.2.5).

**Note:** Any other type of errors and exceptions are catched by Loowy and sent back to the client's side, 
not just this type of custom errors. In this case the details of the error can be lost.

[Back to TOC](#table-of-contents)

unregister(topicURI[, callbacks])
------------------------------------------

RPC unregistration for invocations.

Parameters:

* **topicURI** - topic to unregister
* **callbacks** - optional parameter. If it is a function, it will be called on successful unregistration 
            or it can be hash table of callbacks:
    * **onSuccess**: will be called on successful unregistration
    * **onError**: will be called if unregistration would be aborted 

[Back to TOC](#table-of-contents)

Copyright and License
=====================

Loowy is licensed under the MIT license.

Copyright (c) 2014, Konstantin Burkalev

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.


[Back to TOC](#table-of-contents)

See Also
========

* [WAMP specification][]
* [Wampy.js][]. WAMP Javascript client-side implementation.
* [Wiola][]. WAMP router powered by LUA Nginx module, Lua WebSocket addon, and Redis as cache store.

[Back to TOC](#table-of-contents)

[WAMP]: http://wamp-proto.org/
[WAMP specification]: http://wamp-proto.org/
[Wiola]: http://ksdaemon.github.io/wiola/
[Wampy.js]: https://github.com/KSDaemon/wampy.js
[WAMP Spec CRA]: https://tools.ietf.org/html/draft-oberstet-hybi-tavendo-wamp-02#section-13.7.2.3
[lua-websockets]: https://github.com/lipp/lua-websockets
[lua-ev]: https://github.com/brimworks/lua-ev
[rapidjson]: https://github.com/xpol/lua-rapidjson
[lua-messagepack]: http://fperrad.github.io/lua-MessagePack/
[busted]: http://olivinelabs.com/busted/

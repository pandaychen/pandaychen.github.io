## gochat

```
title registry and login

note right of Client: user registry phrase
Client->API(S): user registry: clientUid and password
API(S)->LOGIC(S): forward user registry
note right of LOGIC(S): check valid
LOGIC(S)->MYSQL: check user valid,then save
LOGIC(S)->API(S): response
API(S)->Client: forward response
note right of Client: user login phrase
Client->API(S): user login: clientUid and password
API(S)->LOGIC(S): forward user login
note right of LOGIC(S): check userinfo valid
note right of LOGIC(S): generate authtoken
LOGIC(S)->REDIS: save authtoken
LOGIC(S)->API(S): response
API(S)->Client: response(authtoken)
Client->CONNECT(S): connect request with authtoken(want long-lived connection)
CONNECT(S)->LOGIC(S): forward authtoken
note right of LOGIC(S): valid auth_token,get related userinfo(clientUid)
note right of LOGIC(S): save user CONNECT(S) IP(user<->CONNECT)
note right of LOGIC(S): save room info(user<->room)
LOGIC(S)->CONNECT(S): response
CONNECT(S)->Client: handshake done(long-lived connection between client and connection module)
```


```text
title message flow(C2C):userA send msg to userB

note right of Client(A): client login succ
Client(A)->CONNECT(SA): long-lived tcp
Client(B)->CONNECT(SB): long-lived tcp
Client(A)->API(S): send Msg to Client(B), with userid[B]
API(S)->LOGIC(S): rpc request
note right of LOGIC(S): check if C2C message[to ClientB]
LOGIC(S)->Redis: get CONNECT(S) info by userid[B]
Redis->LOGIC(S): userid[B]==>CONNECT(S) info,like [roomid,connect_svr_id]
LOGIC(S)->Redis: LPush (room_id,connect_svr_id,message) to redis LIST
note left of TASK(S): async message consumer
TASK(S)->Redis: Rpop message From LIST
note left of TASK(S): check msg type[C2C],then find CONNECT(SB) connect_svr_id
TASK(S)->CONNECT(SB): forward message from Client(A)
CONNECT(SB)->Client(B): forward message from Client(A)
```


```text
title Client logout

note right of Client(A): client login succ
Client(A)->CONNECT(S): long-lived tcp
Client(A)->CONNECT(S): client close connection
CONNECT(S)->LOGIC(S): clientA logout(closed)
note right of LOGIC(S): do cleaning job
LOGIC(S)->Redis: DEL authtoken, DEL connect_server_id
note right of LOGIC(S): if clientA in roomA,then need to send all users[roomA] a ClientA quit message
LOGIC(S)->CONNECT(S):  reply done
CONNECT(S)->Client(A): Close long-lived tcp connection
```
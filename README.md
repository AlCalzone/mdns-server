# mdns-server

Low level Multicast DNS implementation in pure javascript. This is a fork of [Bryan Nielsen's](@bnielsen1965) version, with a small bug fix.

Based on [multicast-dns](https://github.com/mafintosh/multicast-dns) by Mathias Buus.

Can be configured to work with a single network interface or multiple network interfaces. When no interface is specified the OS package is used to detect all available interfaces.

```
npm install mdns-server
```

## Usage

``` js
var mdns = require('mdns-server')({
  reuseAddr: true, // in case other mdns service is running
  loopback: true,  // receive our own mdns messages
  noInit: true     // do not initialize on creation
})

// listen for response events from server
mdns.on('response', function(response) {
  console.log('got a response packet:')
  var a = []
  if (response.answers) {
    a = a.concat(response.answers)
  }
  if (response.additionals) {
    a = a.concat(response.additionals)
  }
  console.log(a)
})

// listen for query events from server
mdns.on('query', function(query) {
  console.log('got a query packet:')
  var q = []
  if (query.questions) {
    q = q.concat(query.questions)
  }
  console.log(q)
})

// listen for the server being destroyed
mdns.on('destroyed', function () {
  console.log('Server destroyed.')
  process.exit(0)
})

// query for 'foo.local' A record when the server is ready
mdns.on('ready', function () {
  mdns.query({
    questions:[{
      name: 'foo.local',
      type: 'A'
    }]
  })
})

// initialize the server now that we are watching for events
mdns.initServer()

// destroy the server after 10 seconds
setTimeout(function () { mdns.destroy() }, 10000)
```

### query packets

In this example a query will be sent on all available IPv4
and IPv6 interfaces. With loopback enabled these queries
should be recorded by the server. I.E.

``` text
got a query packet:
[ { name: 'foo.local', type: 'A', class: 1 } ]
got a query packet:
[ { name: 'foo.local', type: 'A', class: 1 } ]
```

Here there were two query packets because the system has
both an IPv4 and IPv6 address on one interface.

### response packets

Assuming a device on the local network matches the query
request for 'foo.local' the server should record a
response similar to the following...

``` text
got a response packet:
[ { name: 'foo.local',
    type: 'A',
    class: 1,
    ttl: 120,
    flush: true,
    data: '192.168.251.125' } ]
```


## API

A packet has the following format

``` js
{
  questions: [{
    name:'brunhilde.local',
    type:'A'
  }],
  answers: [{
    name:'brunhilde.local',
    type:'A',
    ttl:seconds,
    data:(record type specific data)
  }],
  additionals: [
    (same format as answers)
  ],
  authorities: [
    (same format as answers)
  ]
}
```

Currently data from `SRV`, `A`, `PTR`, `TXT`, `AAAA` and `HINFO` records is passed

### `mdns = mdns-server([options])`

Creates a new `mdns` instance. Options can contain the following

``` js
{
  reuseAddr: (true) || Boolean,
  interfaces: (null) || String || [String, String, ...],
  ttl: (255) || Integer,
  loopback: (true) || Boolean,
  noInit: (false) || Boolean
}
```

#### reuseAddr

Determines if socket will be allowed to reuse an address
already in use. (requires node >=0.11.13) This is helpful
if another process is also listening for mDNS packets.

#### interfaces

Specify the interface or interfaces to be used on the
mDNS server. If no interfaces are specified then all
available network interfaces will be used. To select a
single interface set interfaces to the ip address of
the selected interface. To select multiple interfaces
use an array of ip addresses.

For backward compatability with [multicast-dns](https://github.com/mafintosh/multicast-dns) *interface* may be used in place of *interfaces*.

##### single interface

``` js
var mdns = require('mdns-server')({
  interfaces: "192.168.8.233"
})
```

##### multiple interfaces

``` js
var mdns = require('mdns-server')({
  interfaces: ["192.168.8.233", "10.69.4.10", "fe80::e03c:16d2:1ba:e62"]
})
```

#### ttl

Set the multicast ttl.

#### loopback

Set whether server will report on mDNS packets
originating from the same machine.

#### noInit

Control automatic server initialization. If set to
false then server will not automatically initialize
and the initServer() method must be called when
your application is ready.


## Events

### `mdns.on('query', (packet, rinfo))`

Emitted when a query packet is received.

``` js
mdns.on('query', function(query) {
  if (query.questions[0] && query.questions[0].name === 'brunhilde.local') {
    mdns.respond(someResponse) // see below
  }
})
```

### `mdns.on('response', (packet, rinfo))`

Emitted when a response packet is received.

The response might not be a response to a query you send as this
is the result of someone multicasting a response.


## Methods

### `mdns.query(packet, [cb])`

Send a dns query. The callback will be called when the packet was sent.

The following shorthands are equivalent

``` js
mdns.query('brunhilde.local', 'A')
mdns.query([{name:'brunhilde.local', type:'A'}])
mdns.query({
  questions: [{name:'brunhilde.local', type:'A'}]
})
```

### `mdns.respond(packet, [cb])`

Send a dns response. The callback will be called when the packet was sent.

``` js
// reply with a SRV and a A record as an answer
mdns.respond({
  answers: [{
    name: 'my-service',
    type: 'SRV',
    data: {
      port:9999,
      weigth: 0,
      priority: 10,
      target: 'my-service.example.com'
    }
  }, {
    name: 'brunhilde.local',
    type: 'A',
    ttl: 300,
    data: '192.168.1.5'
  }]
})
```

The following shorthands are equivalent

``` js
mdns.respond([{name:'brunhilde.local', type:'A', data:'192.158.1.5'}])
mdns.respond({
  answers: [{name:'brunhilde.local', type:'A', data:'192.158.1.5'}]
})
```

### `mdns.destroy()`

Destroy the mdns instance. Closes the udp socket.

### `mdns.initServer()`

Initialize the server. Only call this method if the constructor
was called with the option noInit: true. This is used if your
application needs some time to setup events before the server initialization.


## License

MIT

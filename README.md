# AmiClient

Asterisk Management Interface (AMI) client library for .NET

**Features**

 * Designed to be used with `async/await`
 * Methods can be safely called from multiple threads
 * The `AmiClient` class is `IObservable` for use with Reactive Extensions (Rx)

**Dependencies**

 * `NetStandard.Library`
 * `System.Reactive.Interfaces`

**Note:** While it's not a dependency, you will probably want to use the `System.Reactive.Linq` package alongside this library.

### Quick start

Here's an easy way to set up an Asterisk 13 development environment:

 1. Download a local copy of [this basic Asterisk configuration](https://github.com/asterisk/asterisk/tree/13/configs/basic-pbx) and place it in `~/Desktop/etc-asterisk` (or your preferred location)
 2. Run a local Asterisk 13 Docker container...

  ```bash
docker run -dit --rm --privileged \
           --name asterisk13-dev \
           -p 5060:5060 -p 5060:5060/udp \
           -p 10000-10500:10000-10500/udp \
           -p 5038:5038 \
           -v ~/Desktop/etc-asterisk:/etc/asterisk \
           cleardevice/docker-cert-asterisk13-ubuntu
```

 3. Use the example code at the bottom as a starting point for learning the *AmiClient* API...

## Public API

```csharp
public sealed class AmiMessage : IEnumerable<KeyValuePair<String, String>>
{
    // creation

    public AmiMessage();

    public DateTimeOffset Timestamp { get; }

    // deserialization

    public static AmiMessage FromBytes(Byte[] bytes);

    public static AmiMessage FromString(String @string);

    // direct field access

    public readonly List<KeyValuePair<String, String>> Fields;

    // field initialization support

    public void Add(String key, String value);

    // field indexer support

    public String this[String key] { get; set; }

    // field enumeration support

    public IEnumerator<KeyValuePair<String, String>> GetEnumerator();

    IEnumerator IEnumerable.GetEnumerator();

    // serialization

    public Byte[] ToBytes();

    public override String ToString();
}

public sealed class AmiClient : IDisposable, IObservable<AmiMessage>
{
    // creation

    public AmiClient(Stream stream);

    // AMI protocol helpers

    public async Task<Boolean> Login(String username, String secret, Boolean md5 = true);

    public async Task<Boolean> Logoff();

    // AMI protocol debugging

    public sealed class DataEventArgs : EventArgs
    {
        public readonly String Data;
    }

    public event EventHandler<DataEventArgs> DataSent;

    public event EventHandler<DataEventArgs> DataReceived;

    // request/reply

    public async Task<AmiMessage> Publish(AmiMessage action);

    // IObservable<AmiMessage>

    public IDisposable Subscribe(IObserver<AmiMessage> observer);

    public void Unsubscribe(IObserver<AmiMessage> observer);

    public void Dispose();
}
```

## Example code

```csharp
using System;
using System.Diagnostics;
using System.Linq;
using System.Threading.Tasks;
using System.Net.Sockets;
using System.Reactive.Linq;

using Ami;

internal static class Program
{
    public static async Task Main(String[] args)
    {
        // To make testing possible, an AmiClient accepts any Stream object that is readable and writable.
        // This means that the user must establish a TCP connection to the Asterisk AMI server separately.

        // Note: constructing an AmiClient object starts a background reader/writer thread

        using(var socket = new TcpClient(hostname: "localhost", port: 5038))
        using(var client = new AmiClient(socket.GetStream()))
        {
            // Activity on the wire can be logged/debugged with the DataSent and DataReceived events.

            client.DataSent += (s, e) => Console.Error.Write(e.Data);
            client.DataReceived += (s, e) => Console.Error.Write(e.Data);

            // Log in...

            if(!await client.Login(username: "admin", secret: "amp111", md5: true))
            {
                Console.WriteLine("Login failed");
                return;
            }

            // Issue a PJSIPShowEndpoints command...

            var showEndpointsResponse = await client.Publish(new AmiMessage
            {
                { "Action", "PJSIPShowEndpoints" },
                // Note: if not provided, a random ActionID is automatically generated by AmiMessage.
            });

            Debug.Assert(showEndpointsResponse["Response"] == "Success");

            // After the PJSIPShowEndpoints command successfully executes, Asterisk will begin emitting
            // EndpointList events. Each EndpointList event represents a single PJSIP endpoint, and uses
            // the same ActionID as the PJSIPShowEndpoints command that caused it.

            // Once events have been emitted for all PJSIP endpoints, an EndpointListComplete event will
            // be emitted, again using the same ActionID as the PJSIPShowEndpoints command that caused it.

            // Here's how System.Reactive.Linq (Rx) lets us easily process these EndpointList events...

            await client
                  .Where(message => message["ActionID"] == showEndpointsResponse["ActionID"])
                  .TakeWhile(message => message["Event"] != "EndpointListComplete")
                  .Do(message =>
                  {
                      Console.Out.WriteLine($"^^^ {message["ObjectName"]} ({message["DeviceState"]})\r\n");
                  });

            // Log off...

            if(!await client.Logoff())
            {
                Console.WriteLine("Logoff failed");
                return;
            }
        }
    }
}
```

### Example output

```
Asterisk Call Manager/2.6.0
Action: Challenge
ActionID: c1a041b8-635d-49ee-9d0b-490c3824fb08
AuthType: MD5

Response: Success
ActionID: c1a041b8-635d-49ee-9d0b-490c3824fb08
Challenge: 152385155

Action: Login
ActionID: 49ab2f47-60fd-4dd9-9a40-b0e7fb715989
AuthType: MD5
Username: admin
Key: 574c088e9c21305bca95977fad139448

Response: Success
ActionID: 49ab2f47-60fd-4dd9-9a40-b0e7fb715989
Message: Authentication accepted

Action: PJSIPShowEndpoints
ActionID: 5554d3da-171f-486e-9895-11060a5dd990

Event: FullyBooted
Privilege: system,all
Status: Fully Booted

Event: SuccessfulAuth
Privilege: security,all
Timestamp: 1510669385.568869
EventTV: 2017-11-14T14:23:05.568+0000
Severity: Informational
Service: AMI
EventVersion: 1
AccountID: admin
SessionID: 0x7fc420003c58
LocalAddress: IPV4/TCP/0.0.0.0/5038
RemoteAddress: IPV4/TCP/172.17.0.1/48520
UsingPassword: 0
SessionTV: 2017-11-14T14:23:05.568+0000

Response: Success
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
EventList: start
Message: A listing of Endpoints follows, presented as EndpointList events

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: 1106
Transport: 
Aor: 1106
Auths: 1106
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1106 (Unavailable)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: 1101
Transport: 
Aor: 1101
Auths: 1101
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1101 (Unavailable)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: 1103
Transport: 
Aor: 1103
Auths: 1103
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1103 (Unavailable)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: 1102
Transport: 
Aor: 1102
Auths: 1102
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1102 (Unavailable)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: 1114
Transport: 
Aor: 1114
Auths: 1114
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1114 (Unavailable)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: 1115
Transport: 
Aor: 1115
Auths: 1115
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1115 (Unavailable)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: dcs-endpoint
Transport: 
Aor: dcs-aor
Auths: 
OutboundAuths: dcs-auth
Contacts: dcs-aor/sip:sip.digiumcloud.net,
DeviceState: Not in use
ActiveChannels: 

^^^ dcs-endpoint (Not in use)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
Auths: 1109
ObjectType: endpoint
ObjectName: 1109
Transport: 
Aor: 1109
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1109 (Unavailable)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: 1108
Transport: 
Aor: 1108
Auths: 1108
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1108 (Unavailable)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: 1110
Transport: 
Aor: 1110
Auths: 1110
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1110 (Unavailable)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: 1111
Transport: 
Aor: 1111
Auths: 1111
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1111 (Unavailable)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: 1112
Transport: 
Aor: 1112
Auths: 1112
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1112 (Unavailable)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: 1113
Transport: 
Aor: 1113
Auths: 1113
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1113 (Unavailable)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: 1105
Transport: 
Aor: 1105
Auths: 1105
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1105 (Unavailable)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: 1104
Transport: 
Aor: 1104
Auths: 1104
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1104 (Unavailable)

Event: EndpointList
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
ObjectType: endpoint
ObjectName: 1107
Transport: 
Aor: 1107
Auths: 1107
OutboundAuths: 
Contacts: 
DeviceState: Unavailable
ActiveChannels: 

^^^ 1107 (Unavailable)

Event: EndpointListComplete
ActionID: 5554d3da-171f-486e-9895-11060a5dd990
EventList: Complete
ListItems: 16

Action: Logoff
ActionID: 890b8d83-9bdd-43af-98aa-4b31be111d15

Response: Goodbye
ActionID: 890b8d83-9bdd-43af-98aa-4b31be111d15
Message: Thanks for all the fish.
```
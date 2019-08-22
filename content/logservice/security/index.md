---
title: LogService Security
---
<!---
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

This document aims to describe what the intended security deployment model of the Ratis LogService.

We will use integration into Apache HBase as an exemplar.

## Background

TLS is technology capable of giving us "strong authentication" over network communication. One-way TLS can provide
encrypted communication while two-way or "mutual" TLS can provide encrypted communication and authentication.

One feature of Ratis is that it is decoupled from the RPC transport in use. gRPC is the foremost transport, and
can be configured to use one-way or two-way/mutual TLS. gRPC is the only transport for Ratis which
supports TLS today.

However, the majority of components under the "Hadoop Umbrella" rely on Kerberos to guarantee strong authentication.
In this respect, use of TLS is jarring. However, gRPC does not support SPNEGO (which allows Kerberos authentication)
which all but requires the use of two authentication mechanisms when combining Ratis with other projects (like HBase).

We anticipate the use of the Ratis LogService as an "embedded WAL" inside of HBase RegionServers and Masters
will result in HBase services using Kerberos authentication to talk to HDFS as well as TLS for Ratis-internal
communication (intra-server Ratis communication and client-server Ratis communication).

## Mutual TLS

Mutual TLS relies on a common certificate authority (CA) to issue all certificates which forms a circle
of trust. Certificates generated by the same CA can be used to set up a mutual TLS connection. A certificate
generated by one CA cannot be used to set up a mutal TLS connection to a service using a certificate
generated by a different CA outside of the circle of trust. [1]

To control the clients and servers with one instance of the LogService, we want to use a single CA to generate
certificates for clients and servers. We will consider this as an invariant going forward.

## HBase Examplar

We expect the following material to be provided for every HBase service using Ratis:

* File containing an X.509 certificate in PEM format
* File containing the PKCS private key in PEM format
* File containing the X.509 certificate for the CA

OpenSSL is capable of creating each of these; however, for this document, we will assume
that you already have these pre-made. The server certificate and private key are unique to every
host participating in the HBase cluster. The server certificate and truststore are not sensitive,
but the private key is sensitive and should be protected like a password.

Every component in HBase using the Ratis LogService would need to ensure that each LogService StateMachine is
configured to use the server keystore and truststore. The LogService state machines would need to constructed
with the appropriate configuration options to specify this TLS material:

```java
RaftProperties properties = ...;

GrpcConfigKeys.TLS.tlsEnabled(properties);
GrpcConfigKeys.TLS.mutualAuthnEnabled(properties);
properties.set(GrpcConfigKeys.TLS.PRIVATE_KEY_FILE_KEY, "/path/to/server-private-key.pem");
properties.set(GrpcConfigKeys.TLS.TRUST_STORE_KEY, "/path/to/ca.crt");
properties.set(GrpcConfigKeys.TLS.CERT_CHAIN_FILE_KEY, "/path/to/server.crt");

RaftServer.Builder builder = RaftServer.newBuilder();
...
builder.setProperties(properties);

RaftServer server = builder.build();
```

Clients to the StateMachine would construct a similar configuration:

```java
RaftProperties properties = ...;

GrpcConfigKeys.TLS.tlsEnabled(properties);
GrpcConfigKeys.TLS.mutualAuthnEnabled(properties);
properties.set(GrpcConfigKeys.TLS.PRIVATE_KEY_FILE_KEY, "/path/to/client-private-key.pem");
properties.set(GrpcConfigKeys.TLS.TRUST_STORE_KEY, "/path/to/ca.crt");
properties.set(GrpcConfigKeys.TLS.CERT_CHAIN_FILE_KEY, "/path/to/client.crt");

RaftClient.Builder builder = RaftClient.newBuilder();
...
builder.setProperties(properties);

RaftClient client = builder.build();
```

With Mutual TLS, there is no notion of a "client" or "server" only certificate. In the above example code,
as long as the certificate and private key are generated using the same certificate authority, any
should function.

For the LogService, this client setup would be hidden behind the facade of the LogService client API.

The HBase WALProvider implementation that uses the Ratis LogService would be providing the location of
this TLS material via the HBase configuration (hbase-site.xml), passing it down into the WALProvider
implementation. As the WALProvider is the broker that doles out readers and writers, and would also, presumably
manage the creation of the StateMachines, it can set up the proper Ratis configuration from the HBase configuration.

[1] There are scenarios with shared trust across CA's that enable other scenarios but these are ignored for the purpose
of this document.
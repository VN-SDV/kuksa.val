# Kuksa Databroker

- [Kuksa Databroker](#kuksa-databroker)
  - [Intro](#intro)
  - [Relation to the COVESA Vehicle Signal Specification (VSS)](#relation-to-the-covesa-vehicle-signal-specification-vss)
  - [Building](#building)
    - [Build all](#build-all)
    - [Build all release](#build-all-release)
  - [Running](#running)
    - [Broker](#databroker)
    - [Test the broker - run client/cli](#test-the-databroker)
    - [Kuksa Data Broker Query Syntax](#data-broker-query-syntax)
    - [Configuration](#configuration)
    - [Build and run databroker container](#build-and-run-databroker)
  - [Limitations](#limitations)
  - [GRPC overview](#grpc-overview)
  - [GRPC Interfaces](#grpc-interfaces)

## Intro

Kuksa Databroker is a GRPC service acting as a broker of vehicle data / data points / signals.

## Relation to the COVESA Vehicle Signal Specification (VSS)

The data broker is designed to support data entries and branches as defined by [VSS](https://covesa.github.io/vehicle_signal_specification/).

In order to generate metadata from a VSS specification that can be loaded by the data broker, it's possible to use the `vspec2json.py` tool
that's available in the [vss-tools](https://github.com/COVESA/vss-tools) repository. E.g.

```shell
./vss-tools/vspec2json.py -I spec spec/VehicleSignalSpecification.vspec vss.json
```

The resulting vss.json can be loaded at startup by supplying the data broker with the command line argument:

```shell
--metadata vss.json
```

## Building

Prerequsites:
- [Rust](https://www.rust-lang.org/tools/install)
- Linux

### Build all

`cargo build --examples --bins`

### Build all release

`cargo build --examples --bins --release`

## Running

### Databroker
Run the broker with:

`cargo run --bin databroker`

Get help, options and version number with:

`cargo run --bin databroker -- -h`

```shell
Kuksa Data Broker

USAGE:
    databroker [OPTIONS]

OPTIONS:
        --address <IP>              Bind address [env: KUKSA_DATA_BROKER_ADDR=] [default: 127.0.0.1]
        --port <PORT>               Bind port [env: KUKSA_DATA_BROKER_PORT=] [default: 55555]
        --vss <FILE>...             Populate data broker with VSS metadata from (comma-separated)
                                    list of files [env: KUKSA_DATA_BROKER_METADATA_FILE=]
        --jwt-public-key <FILE>     Public key used to verify JWT access tokens
        --tls-cert <FILE>           TLS certificate file (.pem)
        --tls-private-key <FILE>    TLS private key file (.pem)
        --insecure                  Allow insecure connections
        --dummy-metadata            Populate data broker with dummy metadata
    -h, --help                      Print help information
    -V, --version                   Print version information

```

To enable authorization, provide a public key used to verify JWT access tokens (provided by connecting clients).

```shell
databroker --jwt-public-key kuksa_certificates/jwt/jwt.key.pub
```

To enable TLS, provide databroker with the path to the (public) certificate and it's corresponding private key.

```shell
databroker --tls-cert kuksa_certificates/Server.pem --tls-private-key kuksa_certificates/Server.key
```

### :warning: Default port not working on Mac OS
The databroker default port `55555` is not usable in many versions of Mac OS. You can not bind it, or if it seems bound you still can not receive messages.
Therefore, on Mac OS you need to start databroker on another port, e.g.

```
databroker --port 55556
```

Please note, this also applies if you use a container environment like K3S or Docker on Mac OS. If you forward the port or exposing the host network
interfaces, the same problem occurs.

For more information see also https://developer.apple.com/forums/thread/671197 

Currently, to run databroker-cli (see below), you do need to change the port it connects to in databroker-cli code and recompile it.

### Test the databroker

Run the cli with:

`cargo run --bin databroker-cli`

To get help and an overview to the offered commands, run the cli and type :

```shell
sdv.databroker.v1 > help
```


If server wasn't running at startup

```shell
not connected> connect
```


The server holds the metadata for the available properties, which is fetched on client startup.
This will enable `TAB`-completion for the available properties in the client. Run "metadata" in order to update it.


Get data points by running "get"
```shell
sdv.databroker.v1 > get Vehicle.ADAS.CruiseControl.IsEnabled 
[get]  OK
Vehicle.ADAS.CruiseControl.IsEnabled: ( NotAvailable )
```

Set data points by running "set"
```shell
sdv.databroker.v1 > set Vehicle.ADAS.CruiseControl.IsEnabled false
[set]  OK
```

#### Authorization
```shell
databroker-cli --token-file jwt/read-vehicle-speed.token
```

or interactively

```shell
sdv.databroker.v1 > token XXXXXXXXXX
```

or 

```shell
sdv.databroker.v1 > token-file jwt/read-vehicle-speed.token
```

#### Connect to databroker using TLS
```shell
databroker-cli --ca-cert kuksa_certificates/CA.pem
```

### Data Broker Query Syntax

Detailed information about the databroker rule engine can be found in [QUERY.md](doc/QUERY.md)


You can try it out in the client using the subscribe command in the client:

```shell
sdv.databroker.v1 > subscribe
SELECT
  Vehicle.ADAS.ABS.IsError
WHERE
  Vehicle.ADAS.ABS.IsEngaged

[subscribe]  OK
Subscription is now running in the background. Received data is identified by [1].
```

### Configuration

| parameter      | default value | cli parameter    | environment variable              | description                                  |
|----------------|---------------|------------------|-----------------------------------|----------------------------------------------|
| vss       | <no active>   | --vss       | KUKSA_DATA_BROKER_METADATA_FILE   | Populate data broker with metadata from file |
| dummy_metadata | <no active>   | --dummy-metadata | <no active>                       | Populate data broker with dummy metadata     |
| listen_address | "127.0.0.1"   | --address        | KUKSA_DATA_BROKER_ADDR            | Listen for rpc calls                         |
| listen_port    | 55555         | --port           | KUKSA_DATA_BROKER_PORT            | Listen for rpc calls                         |
| jwt_public_key | <no active>   | --jwt-public-key | <no active>                       | Public key used to verify JWT access tokens |
| tls_cert | <no active> | --tls-cert | <no active> | TLS certificate file (.pem) |
| tls_private_key | <no active> | --tls-private-key | <no active> | TLS private key file (.pem) |
| insecure | <no active> | --insecure | <no active> | Allow insecure connections |

To change the default configuration use the arguments during startup see [run section](#running) or environment variables.

### Build and run databroker

To build the release version of databroker, run the following command:

```shell
RUSTFLAGS='-C link-arg=-s' cargo build --release --bins --examples
```
Or use following commands for aarch64
```
cargo install cross

RUSTFLAGS='-C link-arg=-s' cross build --release --bins --examples --target aarch64-unknown-linux-gnu
```
Build tar file from generated binaries.

```shell
# For amd64
tar -czvf databroker_x86_64.tar.gz \
    target/release/databroker \
    target/release/databroker-cli \
    target/release/examples/perf_setter \
    target/release/examples/perf_subscriber
```
```shell
# For aarch64
tar -czvf databroker_aarch64.tar.gz \
    target/aarch64-unknown-linux-gnu/release/databroker-cli \
    target/aarch64-unknown-linux-gnu/release/databroker \
    target/aarch64-unknown-linux-gnu/release/examples/perf_setter \
    target/aarch64-unknown-linux-gnu/release/examples/perf_subscriber
```
To build the image execute following commands from root directory as context.
```shell
docker build -f kuksa_databroker/Dockerfile -t databroker:<tag> .
```

Use following command if buildplatform is required
```shell
DOCKER_BUILDKIT=1 docker build -f kuksa_databroker/Dockerfile -t databroker:<tag> .
```

The image creation may take around 2 minutes.
After the image is created the databroker container can be ran from any directory of the project:

```shell
#By default the container will execute the ./databroker command and load the latest VSS file.
docker run --rm -it  -p 55555:55555/tcp databroker
```

To run any specific command, just append you command at the end.

```shell
docker run --rm -it  -p 55555:55555/tcp databroker <command>
```

## Limitations

- Arrays are not supported in conditions as part of queries (i.e. in the WHERE clause).
- Arrays are not supported by the cli (except for displaying them)

## GRPC overview
Vehicle data broker uses GRPC for the communication between server & clients.

A GRPC service uses `.proto` files to specify the services and the data exchanged between server and client.
From this `.proto`, code is generated for the target language (it's available for C#, C++, Dart, Go, Java, Kotlin, Node, Objective-C, PHP, Python, Ruby, Rust...)

This implementation uses the default GRPC transport HTTP/2 and the default serialization format protobuf. The same `.proto` file can be used to generate server skeleton and client stubs for other transports and serialization formats as well.

HTTP/2 is a binary replacement for HTTP/1.1 used for handling connections / multiplexing (channels) & and providing a standardized way to add authorization headers for authorization & TLS for encryption / authentication. It support two way streaming between client and server.

## GRPC Interfaces

Databroker implements a couple of GRPC interfaces.

### `kuksa.val.v1.VAL`

This GRPC interface is under development and is meant to be used by kuksa-databroker, kuksa-val-server and the feeders.

### `sdv.databroker.v1.Broker`

This interface is currently used by databroker clients. It is defined as follows (see [file](proto/sdv/databroker/v1/broker.proto) in the proto folder):

```protobuf
service Broker {
    rpc GetDatapoints(GetDatapointsRequest) returns (GetDatapointsReply);
    rpc SetDatapoints(SetDatapointsRequest) returns (SetDatapointsReply);

    rpc Subscribe(SubscribeRequest) returns (stream Notification);

    rpc GetMetadata(GetMetadataRequest) returns (GetMetadataReply);
}
```

### `sdv.databroker.v1.Collector`
There is also a [Collector](proto/sdv/databroker/v1/collector.proto) interface which is used by providers to feed data into the broker.

```protobuf
service Collector {
    rpc RegisterDatapoints(RegisterDatapointsRequest) returns (RegisterDatapointsReply);

    rpc UpdateDatapoints(UpdateDatapointsRequest) returns (UpdateDatapointsReply);

    rpc StreamDatapoints(stream StreamDatapointsRequest) returns (stream StreamDatapointsReply);
}
```

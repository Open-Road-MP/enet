<div align="center">
    <img src="https://img.shields.io/github/package-json/v/Open-Road-MP/enet?style=for-the-badge" alt="version" />
    <a href="LICENSE"><img src="https://img.shields.io/github/license/Open-Road-MP/enet?style=for-the-badge" alt="license" /></a>
</div>

<br />

<div align="center">
  ENet - Simple, lightweight and reliable UDP networking library written in pure C
</div>

<div align="center">
  <sub>
    Maintained by the <a href="https://openroadmp.com">Open Road MP</a> team.
    Based on the work of
    <a href="https://github.com/lsalzman">@lsalzman</a> (original ENet),
    <a href="https://github.com/inlife">@inlife</a>,
    <a href="https://github.com/zpl-zak">@zpl-zak</a>,
    <a href="https://github.com/nxrighthere">@nxrighthere</a>
    and the contributors of <a href="https://github.com/zpl-c/enet">zpl-c/enet</a>.
  </sub>
</div>

## About this fork

This is the ENet used by [Open Road MP](https://openroadmp.com), a multiplayer mod for Euro Truck Simulator 2
where anyone can host their own server. The game connection between client and server runs over ENet, on
servers that are open to the internet. So we optimize and maintain this library as part of Open Road MP,
with a focus on robustness against hostile traffic, and we keep it usable for everyone else.

It is based on [zpl-c/enet](https://github.com/zpl-c/enet) (single header, IPv6, monotonic time), which in turn is
a fork of the original [lsalzman/enet](https://github.com/lsalzman/enet). The API stays compatible with zpl-c/enet;
our changes are bug and security fixes, listed below and in [misc/ChangeLog](misc/ChangeLog).

## Security fixes (2.7.1)

| Area | Problem | Fix |
|---|---|---|
| Handshake | A `CONNECT` with a forged sender address made the host send its `VERIFY_CONNECT` again and again to that address: a reflection amplifier. | The answer goes out at most `ENET_HANDSHAKE_ATTEMPTS` (2) times, then the half-open peer is dropped. A real client repeats its `CONNECT` and gets a fresh answer. |
| Timeouts | `1u << (sendAttempts - 1)` is undefined behaviour from 33 attempts on, which the linear backoff reaches with a low RTT. | The shift is bounded. |
| CRC32 | `reflect_crc` shifted `1 << 31` on a signed `int` (undefined behaviour). | Unsigned arithmetic. |
| Receiving | A datagram shorter than its own header (sent-time flag on 2-3 bytes) made the decompressor read with a length that wrapped around. | Such datagrams are dropped. |
| Fragments | A fragmented packet could be completed with gaps that were never written (overlapping fragment offsets). The application then got old heap memory as message bytes. | Reassembly buffers start zeroed. |
| Throttle | The packet throttle parameters from `CONNECT` and `THROTTLE_CONFIGURE` were taken unchecked. An acceleration near 2^32 wrapped the throttle around. | Clamped: interval at most 60 s, acceleration and deceleration at most `ENET_PEER_PACKET_THROTTLE_SCALE`. |

## Recommended settings for public servers

ENet does not authenticate datagrams by itself. These settings are not defaults (they would change the
behaviour of existing applications), but we recommend them for anything reachable from the internet:

```c
ENetHost *host = enet_host_create(&address, maxPeers, channels, 0, 0);

/* A CRC32 seeded with the random connectID of each connection: someone who cannot
 * see the traffic cannot inject packets (for example a DISCONNECT with a player's
 * address). Both ends must set it - it changes the datagram layout. */
host->checksum = enet_crc32;

/* The defaults allow 32 MB per packet and 32 MB of buffered data per peer, so a
 * few connections can make a server allocate gigabytes. Size them to your messages. */
host->maximumPacketSize  = 256 * 1024;
host->maximumWaitingData = 1024 * 1024;

/* Connections per IP address (default: unlimited). */
host->duplicatePeers = 8;
```

Also close connections on your side that never authenticate: ENet keeps an idle peer alive for as long as it
answers pings, so one host could hold all slots of your server.

## Installation

Copy [include/enet.h](include/enet.h) into your project.

## Usage (Shared library)

Build the shared library:

```sh
$ cmake -B build -DENET_SHARED=1 -DCMAKE_BUILD_TYPE=Release
$ cmake --build build
```

Use it:

```c
#define ENET_DLL
#include <enet.h>
#include <stdio.h>

int main() {
    if (enet_initialize () != 0) {
        printf("An error occurred while initializing ENet.\n");
        return 1;
    }

    enet_deinitialize();
    return 0;
}
```

## Usage (Static library)

Build the static library:

```sh
$ cmake -B build -DENET_STATIC=1 -DCMAKE_BUILD_TYPE=Release
$ cmake --build build
```

Use it:

```c
#include <enet.h>
#include <stdio.h>

int main() {
    if (enet_initialize () != 0) {
        printf("An error occurred while initializing ENet.\n");
        return 1;
    }

    enet_deinitialize();
    return 0;
}
```

## Usage (Single Header, Preferred)

In this case, library will be embedded to the project itself.

Make sure you add a define for `ENET_IMPLEMENTATION` exactly in one source file before including the `enet.h`.

Here is a simple server and client demo, it will wait 1 second for events, and then exit if none were found:

Server:

```c
#define ENET_IMPLEMENTATION
#include <enet.h>
#include <stdio.h>

int main() {
    if (enet_initialize () != 0) {
        printf("An error occurred while initializing ENet.\n");
        return 1;
    }

    ENetAddress address = {0};

    address.host = ENET_HOST_ANY; /* Bind the server to the default localhost.     */
    address.port = 7777; /* Bind the server to port 7777. */

    #define MAX_CLIENTS 32

    /* create a server */
    ENetHost * server = enet_host_create(&address, MAX_CLIENTS, 2, 0, 0);

    if (server == NULL) {
        printf("An error occurred while trying to create an ENet server host.\n");
        return 1;
    }

    printf("Started a server...\n");

    ENetEvent event;

    /* Wait up to 1000 milliseconds for an event. (WARNING: blocking) */
    while (enet_host_service(server, &event, 1000) > 0) {
        switch (event.type) {
            case ENET_EVENT_TYPE_CONNECT:
                printf("A new client connected from %x:%u.\n",  event.peer->address.host, event.peer->address.port);
                /* Store any relevant client information here. */
                event.peer->data = "Client information";
                break;

            case ENET_EVENT_TYPE_RECEIVE:
                printf("A packet of length %lu containing %s was received from %s on channel %u.\n",
                        event.packet->dataLength,
                        event.packet->data,
                        event.peer->data,
                        event.channelID);
                /* Clean up the packet now that we're done using it. */
                enet_packet_destroy (event.packet);
                break;

            case ENET_EVENT_TYPE_DISCONNECT:
                printf("%s disconnected.\n", event.peer->data);
                /* Reset the peer's client information. */
                event.peer->data = NULL;
                break;

            case ENET_EVENT_TYPE_DISCONNECT_TIMEOUT:
                printf("%s disconnected due to timeout.\n", event.peer->data);
                /* Reset the peer's client information. */
                event.peer->data = NULL;
                break;

            case ENET_EVENT_TYPE_NONE:
                break;
        }
    }

    enet_host_destroy(server);
    enet_deinitialize();
    return 0;
}

```

Client:

```c
#include <stdio.h>
#define ENET_IMPLEMENTATION
#include "enet.h"

int main() {
  if (enet_initialize() != 0) {
    fprintf(stderr, "An error occurred while initializing ENet.\n");
    return EXIT_FAILURE;
  }

  ENetHost* client = { 0 };
  client = enet_host_create(NULL /* create a client host */,
    1 /* only allow 1 outgoing connection */,
    2 /* allow up 2 channels to be used, 0 and 1 */,
    0 /* assume any amount of incoming bandwidth */,
    0 /* assume any amount of outgoing bandwidth */);
  if (client == NULL) {
    fprintf(stderr,
      "An error occurred while trying to create an ENet client host.\n");
    exit(EXIT_FAILURE);
  }

  ENetAddress address = { 0 };
  ENetEvent event = { 0 };
  ENetPeer* peer = { 0 };
  /* Connect to some.server.net:1234. */
  enet_address_set_host(&address, "127.0.0.1");
  address.port = 7777;
  /* Initiate the connection, allocating the two channels 0 and 1. */
  peer = enet_host_connect(client, &address, 2, 0);
  if (peer == NULL) {
    fprintf(stderr,
      "No available peers for initiating an ENet connection.\n");
    exit(EXIT_FAILURE);
  }
  /* Wait up to 5 seconds for the connection attempt to succeed. */
  if (enet_host_service(client, &event, 5000) > 0 &&
    event.type == ENET_EVENT_TYPE_CONNECT) {
    puts("Connection to some.server.net:1234 succeeded.");
  } else {
    /* Either the 5 seconds are up or a disconnect event was */
    /* received. Reset the peer in the event the 5 seconds   */
    /* had run out without any significant event.            */
    enet_peer_reset(peer);
    puts("Connection to some.server.net:1234 failed.");
  }

  // Receive some events
  enet_host_service(client, &event, 5000);

  // Disconnect
  enet_peer_disconnect(peer, 0);

  uint8_t disconnected = false;
  /* Allow up to 3 seconds for the disconnect to succeed
   * and drop any packets received packets.
   */
  while (enet_host_service(client, &event, 3000) > 0) {
      switch (event.type) {
      case ENET_EVENT_TYPE_RECEIVE:
          enet_packet_destroy(event.packet);
          break;
      case ENET_EVENT_TYPE_DISCONNECT:
          puts("Disconnection succeeded.");
          disconnected = true;
          break;
      }
  }

  // Drop connection, since disconnection didn't successed
  if (!disconnected) {
      enet_peer_reset(peer);
  }

  enet_host_destroy(client);
  enet_deinitialize();
}
```

## Tutorials

More information, examples and tutorials can be found at the site of the original ENet: http://enet.bespin.org/




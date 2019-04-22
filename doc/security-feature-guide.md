# Security Feature Guide

## Introduction

RIFT-Python supports the following security features:

 * Configure keys: key identifier, key algorithm, and key secret.

 * Select one key as the active key to be used to compute the security fingerprints for sent
   messages.

 * Select zero or more additional keys as accept keys to allow other nodes to use a different
   active key during key roll-over scenarios.

 * Generate, parse, and validate the envelope for all packets:

   * Generate packet-nrs for all sent packets (increasing within the scope of packet-type and
     address-family of the UDP packet).

   * Parse packet-nr for all received packet, and keep statistics on out-of-order packets. But 
     no other action is taken, e.g. flooding speed is not automatically throttled
     based on observed packet drops.

   * The visualization tool uses packet-nrs to associate packets sent by one node with packets 
     received on another node.

 * Generate, parse, and validate the outer security envelope for all packets:

   * Generate the outer fingerprint for all sent packets according to the configured active
     key (if any).

   * Parse and validate the outer fingerprint for all received packets according to the
     configured active key and accept keys (if any).

   * Ignore all packets with an unrecognized outer key ID.

   * Ignore all packets with an incorrect outer fingerprint.

   * Generate increasing local nonces for all sent packets.

   * Parse the local nonce in all received packets, and reflect it in the remote nonce on all
     sent packet.

   * Parse the remote nonce (reflected nonce) in all received packets.

   * Detect and ignore packets whose reflected nonce is too far out of sync with the local nonce.

   * Generate the appropriate TIE remaining lifetime for all sent packets (actual remaining lifetime
     for TIE packet, and all ones for other packets).

   * Parse and process the TIE remaining lifetime in all received packets.

 * Generate, parse, and validate the TIE origin security envelope for all packets:

   * Generate the TIE origin fingerprint for all sent TIE packets according to the configured active
     key (if any).

   * Parse and validate the TIE origin fingerprint for all received TIE packets according to the
     configured active key and accept keys (if any).

   * Ignore all packets with an unrecognized TIE origin key ID.

   * Ignore all packets with an incorrect TIE origin fingerprint.

 * Log and keep statistics on all types of outer security envelope authentication errors.

 * CLI command "show security" reports configured keys and detected authentication errors on all
   interfaces.

## Configure keys

You can configure the set of security keys at the top-level in the configuration file (also known
as the topology file) as follows:

<pre>
keys:
  - id: 1
    algorithm: hmac-sha-1
    secret: this-is-the-secret-for-key-1
  - id: 2
    algorithm: hmac-sha-256
    secret: this-is-the-secret-for-key-2
  - id: 3
    algorithm: hmac-sha-512
    secret: this-is-the-secret-for-key-3
</pre>

Each key is defined by:

 * A unique key ID. This is a number between 1 and 255.

 * The key algorithm. This is the algorithm used to calculate the fingerprint. The currently
   supported algorithms are: hmac-sha-1, hmac-sha-224, hmac-sha-256, hmac-sha-384, hmac-sha-512.
   These algorithms are defined in [RFC2104](https://tools.ietf.org/html/rfc2104).

 * The key secret. This is a string.

Note: in the current implementation the key secret is shown in plain text in both the configuration
file and in the output of show commands. Obfuscation of secrets is not yet supported.

Configuring a set of keys in itself and by itself does not cause any fingerprints to be generated
of checked. You must use the active-key element and optionally the accept-keys element (both
documented below) to enable to actual generation and checking of fingerprints.

## Configure the active key

The "active key" for a node is the key that the node uses to generate *all* fingerprints:

 * All outer fingerprints on all interfaces.

 * All TIE origin fingerprints.

For the sake of simplicity, RIFT-Python does currently not support using different outer keys on
different interfaces, nor does it support using a different outer key and TIE origin key.

As a consequence, RIFT-Python currently only supports the Fabric Association Model (FAM), and not
the the Node Association Model (NAM) or Port Association Model (PAM).

You can configure the active key at the node level in the configuration file (also known
as the topology file) as follows:

<pre>
shards:
  - id: 0
    nodes:
      - name: node-1
        level: 2
        systemid: 1
        <b>active_key: 1</b>
        interfaces:
          - name: if1
            [...]
</pre>

The active key must match one of the key IDs in the keys section (see 
the ["configure keys"](configure-keys) section.)

If the active_key is not configured, then null authentication is used: the sent key-id is zero,
and the sent fingerprint is empty.

## Configure accept keys

For key roll-over scenarios you can configure a list of accept keys.

Nodes always end packets with a key-id and fingerprint that is determined by the active key.
But when a node receives a packet with a non-zero key-id and a non-empty fingerprint, it will
validate and accept the received packet if the received key-id matches the active key *or* any one
of the accept keys.

You can configure the list of accept keys at the node level in the configuration file (also known
as the topology file) as follows:

<pre>
shards:
  - id: 0
    nodes:
      - name: node-1
        level: 2
        systemid: 1
        active_key: 1
        <b>accept_keys: [2, 3]</b>
        interfaces:
          - name: if1
            [...]
</pre>

Each accept key must match one of the key IDs in the keys section (see 
the ["configure keys"](configure-keys) section.)

See section TODO for more details on how to do key roll-overs.

## Packet numbers

RIFT-Python generates packet numbers (packet-nrs) for each sent packet.

The packet-nr is unique within the scope of the packet-type and the address family of the UDP
packet. For example:

 * All the IPv4 TIE packets that a give node sends, have increasing packet-nrs: 1, 2, 3, 4, ...

 * The IPv4 TIE packets and the IPv4 TIDE packets sent by a given node each have their own
   independent sequence of increasing packet-nrs.

 * The IPv4 TIE packets and the IPv6 TIE packets sent by a given node each have their own
   independent sequence of increasing packet-nrs.

The reason for this is to avoid false positives of reordering for packets that are sent on different
sockets or by different threads.

If debug logging is turned on (command line option "--log-level debug"), the packet-nrs are reported
in the logs for sent and received messages, for example:

<pre>
2019-04-22 19:22:55,179:DEBUG:node.if.tx:[node-1:if1] Send IPv4 TIE from 192.168.0.100:49307 to 192.168.0.100:10005 <b>packet-nr=2</b> outer-key-id=1 nonce-local=7 nonce-remote=5 remaining-lie-lifetime=604799 outer-fingerprint-len=5 origin-key-id=1 origin-fingerprint-len=5 protocol-packet=ProtocolPacket(...
</pre>

RIFT-Python uses the packet-nr to detect packet misordering: if the packet-nr of a received packet
is not equal to the packet-nr plus one of the previously received packet (of the same packet-type 
and the same address-family), then the packet is declared to be misordered. 

Note that we use the generic term misordering, but in reality it could indicate re-ordered,
dropped or dupplicated packets.

Misordered packets are reported in the output of "show ... statistics" for example:

<pre>
node-1> <b>show interface if1 statistics</b>
Traffic:
+---------------------------------------------------+------------------------+------------------------------------+-------------------+
| Description                                       | Value                  | Last Rate                          | Last Change       |
|                                                   |                        | Over Last 10 Changes               |                   |
+---------------------------------------------------+------------------------+------------------------------------+-------------------+
.                                                   .                        .                                    .                   .
.                                                   .                        .                                    .                   .
.                                                   .                        .                                    .                   .
+---------------------------------------------------+------------------------+------------------------------------+-------------------+
| RX IPv4 LIE Misorders                             | 6 Packets              | 1.03 Packets/Sec                   | 0d 00h:00m:00.56s |
+---------------------------------------------------+------------------------+------------------------------------+-------------------+
| RX IPv6 LIE Misorders                             | 6 Packets              | 1.03 Packets/Sec                   | 0d 00h:00m:00.56s |
+---------------------------------------------------+------------------------+------------------------------------+-------------------+
| RX IPv4 TIE Misorders                             | 0 Packets              |                                    |                   |
+---------------------------------------------------+------------------------+------------------------------------+-------------------+
| RX IPv6 TIE Misorders                             | 0 Packets              |                                    |                   |
+---------------------------------------------------+------------------------+------------------------------------+-------------------+
| RX IPv4 TIDE Misorders                            | 1 Packet               |                                    | 0d 00h:00m:01.54s |
+---------------------------------------------------+------------------------+------------------------------------+-------------------+
| RX IPv6 TIDE Misorders                            | 0 Packets              |                                    |                   |
+---------------------------------------------------+------------------------+------------------------------------+-------------------+
| RX IPv4 TIRE Misorders                            | 2 Packets              | 1.00 Packets/Sec                   | 0d 00h:00m:02.45s |
+---------------------------------------------------+------------------------+------------------------------------+-------------------+
| RX IPv6 TIRE Misorders                            | 0 Packets              |                                    |                   |
+---------------------------------------------------+------------------------+------------------------------------+-------------------+
| Total RX IPv4 Misorders                           | 9 Packets              | 1.65 Packets/Sec                   | 0d 00h:00m:00.56s |
+---------------------------------------------------+------------------------+------------------------------------+-------------------+
| Total RX IPv6 Misorders                           | 6 Packets              | 1.03 Packets/Sec                   | 0d 00h:00m:00.56s |
+---------------------------------------------------+------------------------+------------------------------------+-------------------+
| Total RX Misorders                                | 15 Packets             | 3.00 Packets/Sec                   | 0d 00h:00m:00.56s |
+---------------------------------------------------+------------------------+------------------------------------+-------------------+
</pre>

No log message is generate when a misordered packet is detected.

RIFT-Python currently does not take any action when it observed misordered packets. For example, it
does not throttle flooding when misordered packets are observed.

The [log visualization tool](doc/log-visualization.md) uses the packet-nrs to associate a sent
message on one node with a received message on a neighbor node.
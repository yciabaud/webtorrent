## Tracker Protocol

### Overview

A webclient communicates with a tracker through a websocket connection. They exchange websocket text messages containing a payload using the JSON format.

There is four category of payloads. Each payload category is represented by a field `action`:
- an `announce` payload represents an annoounce similar to the original BitTorrent spec. There is still a small difference, because an announce contains a list of initial WebRTC offers that the tracker will forwards to the clients in the swarm, insted of returning a list of peers to the client.
- a `scrape` payload is similar to the original BitTorrent spec.
- an `offer` payload which represents a WebRTC offer the tracker has to forward.
- an `answer` payload which represents a WebRTC answer the tracker has to forward.

#### announce

A client send an `announce` payload to the tracker.

```
{
   "action": "announce",
   "info_hash": "",
   "peer_id": "",
   "uploaded": "",
   "downloaded": "",
   "left": "",
   "event": "",
   "numwant": "",
   "offers": [
      {
         "id": "",
         "sdp": ""
      },
      ...
   ]
}
```

The fields used in the payload are as follows:
- `action`: string
- `info_hash`: 20-bytes SHA1 hash of the value of the `info` key from the Metainfo file. This field is encoded in hexadecimal.
- `peer_id`: 20-bytes string used as a unique ID for the client, generated at startup by the client.
- `uploaded`: Total amount of bytes uploaded since the client sent the `started` event to the tracker, represented in base ten ASCII.
- `downloaded`: Total amount of bytes downloaded since the client sent the `started` event to the client, represented in base ten ASCII.
- `left`: The number of bytes needed for the download to be 100% complete and get all the included files in the rorrentm represented in base 10 ASCII.
- `event`: One of:
   - `started`: The client started the download of the torrent.
   - `stopped`: The client stopped the download of the torrent.
   - `completed`: The client has completed the download of the torrent. This event should not be sent if the torrent was already 100% downloaded when it started.
   - `update`: This represents an announce done at regular intervals.
   *in the original BitTorrent BEP, this event could be missing or empty, which hhas been discarded for clarity and consistency.*
- `numwant`: Number of peers that the client would like to connect to.
- `offers`: A list of WebRTC offers. The total of offers must be atleast equal to `numwant`. Each offer is an object containg the following fields:
   - `id`: a random 20-bytes unique identifier string.
   - `sdp`: The content of a WebRTC offer.

Upon receiving an `announce` payload, the tracker will select a number of peers corresponding to `numwant` in the torrent swarm and will forward to each a unique WebRTC offer picked in the `offers` list, using an `offer` payload.

#### scrape

A scrape payload contains differents fields depending of its a request from the client, or an response from the tracker.

A client send a scrape `payload` to the tracker.

```
{
   "action": "scrape",
   "info_hash": ""
}
```

The tracker sent back a `scrape` payload which contains three more fields describing the stats of a torrent swarm.

```
{
   "action": "scrape",
   "info_hash": "",
   "complete": "",
   "incomplete": "",
   "downloaded": ""
}
```

The fields used in the payload are as follows:
- `action`: string
- `info_hash`: 20-bytes SHA1 hash of the value of the `info` key from the Metainfo file. This field is encoded in hexadecimal.
- `complete`: number of peers that have completed the download of the torrent, in base 10 ASCII. (also known as seeders).
- `incomplete`: number of peers that has not yet completed the download of the torrent, in base ten ASCII. (also known as leechers).
- `downloaded`: number of times the tracker has registered a completion event (`event: completed`), in base ten ASCII. (also know as snatched).

A client can also request a scrape for multiple torrents.
The fields are the same, but the structure of the request/response is different.

multi-scrape request
```
{
   "action": "scrape",
   "info_hash": ["ih1", "ih2", ...]
}
```

multi-scrape response
```
{
   "action": "scrape",
   "scrapes": [
      "ih1": {
         "complete": "0",
         "incomplete": "0",
         "downloaded": ""
      },
      "ih2": {
         "complete": "0",
         "incomplete": "0",
         "downloaded": ""
      },
      ...
   ]
}
```

#### offer

An `offer` payload is forwarded by the tracker to a client.
A client never send an offer payload by himself, instead it sends a list of `offer` in an `announce` payload.

```
{
   "action": "offer",
   "info_hash": "",
   "id": "",
   "from": "",
   "sdp": ""
}
```

The fields used in the payload are as follows:
- `action`: string
- `info_hash`: 20-bytes SHA1 hash of the value of the `info` key from the Metainfo file. This field is encoded in hexadecimal.
- `id`: unique identifier of the offer.
- `from`: identifier of the peer who created the offer.
- `sdp`: WebRTC offer.

#### answer

An `answer` payload is sent back to the tracker upon receiving an `offer` payload. The tracker has to forward it back to the client indicated by `to`. The tracker MAY discard the original field `to` of the `answer` when it forwards it, because the receiver won't need it.

```
{
   "action": "answer",
   "info_hash": "",
   "offer_id": "",
   "from": "",
   "to": "",
   "answer": ""
}
```

The fields used in the payload are as follows:
- `action`: string
- `info_hash`: 20-bytes SHA1 hash of the value of the `info` key from the Metainfo file. This field is encoded in hexadecimal.
- `offer_id`: unique identifier of the offer from which the answer was generated from.
- `from`: identifier of the peer who created the answer. this is the identifier found in the field `id` of an `offer` payload.
- `to`: identifier of the peer to whom the tracker must forward the answer. this is the identifier found in the field `from` of an `offer` payload.
- `sdp`: WebRTC answer.

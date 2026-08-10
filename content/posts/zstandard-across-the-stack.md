+++
date = '2026-04-02T12:15:48+02:00'
draft = false
title = 'Zstandard Across the Stack'
description = 'How we used zstd with custom dictionaries to compress real-time game data, patch game updates, and speed up downloads at Seed.'
tags = ['compression', 'zstd', 'gamedev']
ShowToc = true
TocOpen = true
+++

I love compression. There is something satisfying about applying math to data to shrink it and make systems faster and more efficient.

[Seed](https://seed.game) is a massively multiplayer online game. MMOs have a particular relationship with bandwidth. Thousands of players sharing a world means the server is constantly pushing state updates to every connected client, and every client is sending commands back. That volume of small, frequent messages adds up fast. On top of that, every game update means every player needs to download new content. At scale, both of these become real constraints on the player experience.

This is about how we used [Zstandard](https://facebook.github.io/zstd/) (zstd) across our stack to address those constraints. Three properties made it particularly useful: custom dictionaries that let you compress tiny messages effectively, streaming decompression that works as data arrives, and tunable compression levels that trade speed for size.

## Real-Time Game Data

This is probably the most impactful for our players. Our game client connects to our game gateways through a long-standing streaming gRPC connection over which we send binary data to connected clients in tiny messages, each saying "field 5 on object X is now 123" or "here is the response to your command." This data stream was becoming uncomfortably large as the game grew.

A perfect use case for compression, since these messages have a lot of redundant data in them: serialization overhead, repeated values, common structures. The problem is that we're compressing individual messages. We can't batch them together, where those repeated values would compound, because we're sending them out in real time, one at a time.

Think of it as compressing a whole book versus compressing an individual sentence. It's easier to find repetitive word sequences across the whole book. Since we're sending the equivalent of a sentence each time, individual messages barely compress at all.

[Discord's blog post](https://discord.com/blog/how-discord-reduced-websocket-traffic-by-40-percent) about reducing websocket traffic inspired us to look at zstd. Their big win came from streaming compression, maintaining context across the lifetime of a connection. We considered that, but streaming compression would mean managing compression state per connection and working around gRPC's per-message compression model. Custom dictionaries were a much cleaner fit. They're stateless, and gRPC already lets client and server negotiate a compressor per-call. So we went with dictionaries, and they worked out really well.

### How Custom Dictionaries Work

When using custom dictionaries, you scan a representative corpus of data ahead of time and build up a list of frequently occurring byte sequences. Then when compressing each message individually, zstd uses that dictionary to replace those sequences with compact references. The decompressor has the same dictionary and reverses the process.

I collected a corpus of training data by adding gRPC middleware that wrote out each individual message to a file, then used the `zstd` command line tool to train a dictionary from that corpus. We embed that dictionary into both the client and the gateway at build time.

### Negotiating Compression over gRPC

gRPC has built-in support for compressing its payload and allows the client and server to negotiate which compression to use via the [`grpc-accept-encoding` header](https://grpc.io/docs/guides/compression/). Our client sends "I can decompress using `zstd-XXX`" where `XXX` is the hash of its dictionary. If the server has the same dictionary, they agree to compress traffic using it. This also gives us a clean upgrade path: as the game evolves, we recreate dictionaries and roll them out with new client versions. Old and new clients simply negotiate different dictionary versions.

### Results

![Dictionary compression: without vs with](/images/posts/zstandard/dictionary-compression.svg)

Around 70-90% bandwidth reduction, with very little CPU overhead since compressing with a pre-built dictionary makes zstd even faster than its default mode.

This is meaningful for our players, reducing both the bandwidth needed to play and the latency caused by head-of-line blocking. For players on the mobile client, it really matters.

We applied the same dictionary approach to our database writes and reads, training a separate dictionary on stored data to reduce I/O on both sides.

This approach is not without precedent. RAD Game Tools has [Oodle Network Compression](https://www.radgametools.com/oodlenetwork.htm) that works the same way. Since RAD was acquired by Epic, Unreal Engine users can adopt this approach [for free](https://www.unrealengine.com/en-US/blog/oodle-now-free-to-use-in-unreal-engine-via-github).

## Patching Instead of Redownloading

We had a problem: every time we released a new version of the game, all players had to redownload the whole thing. When we're updating rapidly, this becomes a real nuisance for players who sometimes have to download the full game every day.

What we wanted was for players to only download the data that changed between versions.

Remarkably, we found ourselves using zstd with custom dictionaries again, but this time as a [patching engine](https://github.com/facebook/zstd/wiki/Zstandard-as-a-patching-engine).

![Patching: old version as dictionary](/images/posts/zstandard/patching.svg)

Since the old version of the game is known on both sides, we compress the new version using the old version as its dictionary. Wherever zstd finds a run of bytes in the new version that also exists in the old version, it collapses that into a dictionary reference. Any genuinely new bytes in between get normal zstd compression.

On the client side, players download only the difference between the two versions and decompress it using their existing installation as the dictionary.

Updates now scale with the amount of changes in them, rather than requiring a full redownload every time.

## Faster Full Downloads

We also used zstd to speed up full downloads.

Previously we used `.zip` to compress the full installation. We split the zip into multiple chunks, downloaded them in parallel, and only once we had reassembled the complete zip could we extract it. Zip's central directory lives at the end of the archive, so the extractor needs the whole file before it knows what's inside.

We replaced this with a `tar.zst` archive where we write a manifest at the start, followed by files laid out sequentially, all compressed as a zstd stream. Since tar is a streaming format and zstd decompresses as data arrives, we can start extracting files as we download them, overlapping the download and extraction phases. We download individual chunks, and as they arrive, we stream them into zstd which writes files to disk while the download is still running.

![Full download: ZIP vs Zstandard](/images/posts/zstandard/full-download.svg)

Since we compress once and decompress many times on player machines, we can afford slow compression times. Zstd lets you tune the compression level, and we found that level 19 yielded about 13% better compression than zip.

## Looking Back

One algorithm with a flexible dictionary abstraction ended up solving four different problems across our stack: real-time network traffic, database storage, game patching, and full downloads. Most engineers think of compression as "zip the file," but once you understand the primitives, dictionaries, streaming, tunable levels, the same mechanism becomes a networking optimization, a storage strategy, and a patching engine. The unlock isn't the algorithm itself, it's knowing enough about how compression works to recognize where it applies.

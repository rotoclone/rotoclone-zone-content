+++
title = "Making My Website Part 1: Monitoring A Raspberry Pi"
tags = ["web-dev", "raspberry-pi"]
+++

When I first set out to make a personal website, I knew I wanted to host it on a Raspberry Pi. Why? Well, my brother had given me a Raspberry Pi 3 B+ a few...years ago, and I never did anything with it because (1) I'm lazy and (2) I never came up with a good use for it. But I recently (well, recently-ish) started making a video game, and I wanted to have a dev blog for it at some point. So one day when I was feeling particularly distracted from my video game, I decided to make a website to host said blog on. I figured hosting it on the Raspberry Pi would be a fun adventure. And I recently learned [Rust](https://www.rust-lang.org/), so in an act born out of a desire to do more Rust things, I also decided to write my own web server/static site generator thing in Rust.

Sure, using a pre-existing static site generator and serving my site via a CDN would probably be faster and simpler and just better overall. But it wouldn't be as fun or provide as much of a learning experience, presumably. And most importantly it wouldn't allow me to use my Raspberry Pi for something. There's something cool about looking over and seeing my cute little Raspberry Pi, knowing it's running software I wrote myself to serve my website out to the internet.

## Ok let's get going on this web server
Hold on a minute now. As all good enterprise software engineers know, you have to have good monitoring set up for your stuff or you're in for a really annoying time if anything ever breaks. I like to think of myself as a good software engineer, so naturally I had to set up some sort of monitoring thing.

In the spirit of this project, of course I wouldn't use some existing monitoring solution. No, I would write my own in Rust. Those wheels aren't gonna re-invent themselves.

## So what does this monitoring thing need
Initially, I thought it would just be a simple API running on the Pi that would return the current system state, then I would call it via some client software running on my desktop to visualize the stats. But I realized pretty quickly that the easier route would be to just have a simple dashboard (in the form of a webpage) hosted on the Pi itself, that way it would be completely self-contained.

So to answer the question I posed to myself in the header up there, I would need a simple webpage hosted on the Raspberry Pi that displays system stats in graph form. And the graphs would necessitate storing a history of the stats.

## Sounds like you need some way to gather system stats
It does sound that way, narrative device in the form of headers. (As an aside, I'm now realizing the power of fasterthanli.me's "Cool Bear" thing, and apparently trying to emulate it via these headers.) Anyway, since I'm writing this in Rust, it'll be in the form of a crate. There are a few options here, but I'm going to focus on just a couple.

### Heim
[Heim](https://heim-rs.github.io/) seems to be the "robust" system stats crate. By that I mean it's complicated, but powerful, and built with a philosophy of "doing it the right way, not the easy way". As this seems to jive with the philosophy behind Rust, you would think this would be an easy pick. I mean, it's got a whole website and everything! This crate is surely going places. But I tried it out and it just seemed a little overly complex for me, while at the same time missing some features found in other system stats crates. (At this point I don't remember specifically what features I wanted but weren't there, sorry!) I'm sure at some point Heim will surpass the feature set of the other options (maybe it already has), but at the time I decided it wasn't for me.

### systemstat
I found [systemstat](https://github.com/unrelentingtech/systemstat) to be a good mix of a simple enough API and all the features I wanted. That's pretty much all I have to say about it I guess. Kind of underwhelming after writing so much about Heim. Oh well.

So I made [a struct](https://github.com/rotoclone/system-stats-dashboard/blob/0.1.0/src/stats.rs#L17) to store the stats in and used systemstat to fill it to the brim with delicious stats.

```rust
use chrono::{DateTime, Local};

pub struct AllStats {
    /// General system stats
    pub general: GeneralStats,
    /// CPU stats
    pub cpu: CpuStats,
    /// Memory stats
    pub memory: Option<MemoryStats>,
    /// Stats for each mounted filesystem
    pub filesystems: Option<Vec<MountStats>>,
    /// Network stats
    pub network: NetworkStats,
    /// The time at which the stats were collected
    pub collection_time: DateTime<Local>,
}
```

I'm not going to reproduce all the code for gathering the stats here ([check it out on GitHub if you're interested](https://github.com/rotoclone/system-stats-dashboard/blob/0.1.0/src/stats.rs)), but here's a taste of how I gather RAM stats:

```rust
use systemstat::{
    saturating_sub_bytes, ByteSize, System,
};

pub struct MemoryStats {
    /// Megabytes of memory used
    pub used_mb: u64,
    /// Megabytes of memory total
    pub total_mb: u64,
}

impl MemoryStats {
    /// Gets memory stats for the provided system. Returns `None` if an error occurs.
    pub fn from(sys: &System) -> Option<MemoryStats> {
        match sys.memory() {
            Ok(mem) => {
                let used_mem = saturating_sub_bytes(mem.total, mem.free);
                Some(MemoryStats {
                    used_mb: bytes_to_mb(used_mem),
                    total_mb: bytes_to_mb(mem.total),
                })
            }
            Err(e) => {
                log("Error getting memory usage: ", e);
                None
            }
        }
    }
}

const BYTES_PER_MB: u64 = 1_000_000;

/// Gets the number of megabytes represented by the provided `ByteSize`.
fn bytes_to_mb(byte_size: ByteSize) -> u64 {
    byte_size.as_u64() / BYTES_PER_MB
}
```

## Cool, so we're done with monitoring
Uh...no. There's still the matter of converting the stats from a big pile in memory to a graph you can look at with your human eyes. Also the stats need to update themselves periodically. Let's go over that next.

So the code to update the stats periodically and keep a rolling history of recent stats is actually pretty complicated. Basically, I start up a thread that periodically gathers new stats and adds them to a list. Once the list reaches a certain size, it gets cleared out and all the stats that were in it get "consolidated" (aka averaged) into a single `AllStats` struct, which is placed into a [circular buffer](https://en.wikipedia.org/wiki/Circular_buffer). That way I can keep the stats up to date by gathering data often, and the consolidation allows me to store a longer history of stats in memory than if I had to keep around every single data point.

I didn't want to lose all the stats history if the Pi shuts down though, so I made the stats history get written to disk periodically. I didn't want to deal with the complexity of a database, so I turned to the tried-and-true "files with JSON in them" strategy. Using [serde](https://serde.rs/) (the gold standard for serialization and deserialization in Rust), every time consolidated stats are written to the circular buffer, they also get serialized to JSON and written to a file. Once that file reaches a certain size, it gets renamed so new stats are written to a fresh file. And once the new file reaches that certain size, it gets renamed to overwrite the old file and the cycle repeats. Astute readers will note that this means half of the persisted stats history will suddenly disappear every so often. That is true. But I can live with it, since it makes the code to manage the stats persistence simpler.
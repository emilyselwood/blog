---
title: "Creating a command line mastodon archive search tool in rust"
date: 2023-02-15T12:11:00+00:00
author: "Emily Selwood"
draft: false
slug: "rust_mastodon_search"
tags: ["search", "rust", "mastodon"]
comments: false     # set false to hide Disqus comments
share: true        # set false to share buttons
ShowPostNavLinks: true
---
Today lets step through creating a new command line tool in rust. Recently I wanted to find one of my old toots, but it was from several months ago and mastodon doesn't have the ability to search for anything other than usernames and hashtags. I couldn't remember if I'd put hash tags in the post I was after so I couldn't use the built in thing. What I did have however is my backup archive. I'd downloaded it earlier for unrelated reasons. Wouldn't it be good if I could run a search on that archive and get back the url for a toot I was looking for? Yep, lets build it. I'm going to step through this in horrible detail as a kind of tutorial on creating a command line tool in rust. So here we go!

Ok, before we start there are some pre-requisites that I already  have setup and I'm not going to talk about.

* I already have [rust installed](https://www.rust-lang.org/tools/install)
* I have downloaded my archive (the option is in your settings under import/export, you may need to tap the hamburger button on mobile to find it)
* I have a text editor and coding environment set up. In my case VS Code and the Rust analyser plugins.

Now, lets have a terminal open and create our new tool.

```bash
cargo new magrep
```
```bash
     Created binary (application) `magrep2` package
```

This will create a new folder in the current directory called magrep (Mastodon Archive Grep) open that directory in your favourite text editor,

```bash
cd magrep
code .
```

There will be a `cargo.toml` a `.gitignore` and a `src/` directory with a `main.rs`. First thing we need is to run this. Ok, we don't strictly *need* to run this, but it is a good idea to make sure everything works.

```bash
cargo run
```
```
   Compiling magrep v0.1.0 (C:\Users\emily\git\magrep)
    Finished dev [unoptimized + debuginfo] target(s) in 1.86s
     Running `target\debug\magrep.exe`
Hello, world!
```

Hurrah it worked as expected. Now we need to have a quick think about what we want this tool to do and how it should work. We want to search the outbox.json file in the tar.gz archive file and look for any toots with a provided string in them. So we need to know where the archive file is, and what string to search for. When the thing runs it should open up the archive file, decode the outbox.json file, filter out anything thats not a toot (it includes likes and boosts too), and print out any that contain our search string.

Now we know that, the first thing the program is going to need to do is read the command line arguments and find out where the archive is and what we are searching for.

To do that we will use a library called Clap. This provides a bunch of command line parsing logic. We just need to add the following to the dependencies section of our `cargo.toml` file

```toml
clap = { version = "3.0", features = ["derive"] }
```

Then we can go to our `src/main.rs` file and start to write some code. We want an argument that is something like `-a <path to archive file>` so we need to create a new command and then add an argument to it. First add an import for the types we need,

```rust
use clap::{Arg, Command};
```

 Then add the following to your main function
```rust
let matches = Command::new("magrep")
        .arg(
            Arg::new("archive")
                .required(true)
                .short('a')
                .long("archive")
                .help("file path for the archive to search"),
        )
        .get_matches();
```
This creates a command magrep and an argument with a short name of `-a` and a long name of `--archive` that is required. Now the next argument we need to create is the search term. Now it would be nice if we didn't have to specify that this was the search term, like when you use sed or grep you just type the search parameter. So lets create a positional argument. 

```rust
let matches = Command::new("magrep")
        .arg(
            Arg::new("archive")
                .required(true)
                .short('a')
                .long("archive")
                .default_value("./archive.tar.gz")
                .help("file path for the archive to search"),
        )
        .arg(
            Arg::with_name("query")
                .index(1)
                .help("match string")
                .required(true),
        )
        .get_matches();
```

Now the matches variable will have our parsed command line arguments in it. We can pull them out into variables for us to use later like so

```rust
    let archive_path = matches.value_of("archive").unwrap();
    let pattern = matches.value_of("query").unwrap();
```

Lets do a quick test and print those out to make sure things are working...

```rust
    println!("path: {}, pattern: {}", archive_path, pattern);
``` 

```bash
cargo run -- -a something.gz "some pattern"
```
This will take a bit longer than before as it will need to build clap and all of its dependencies too. You should see something like this
```
   Compiling hashbrown v0.12.3
   Compiling os_str_bytes v6.4.1
   Compiling once_cell v1.17.1
   Compiling bitflags v1.3.2
   Compiling strsim v0.10.0
   Compiling textwrap v0.16.0
   Compiling winapi v0.3.9
   Compiling clap_lex v0.2.4
   Compiling indexmap v1.9.2
   Compiling winapi-util v0.1.5
   Compiling atty v0.2.14
   Compiling termcolor v1.2.0
   Compiling clap v3.2.23
   Compiling magrep v0.1.0 (C:\Users\emily\git\magrep)
    Finished dev [unoptimized + debuginfo] target(s) in 9.00s
     Running `target\debug\magrep.exe -a something.gz "some pattern"`
path: something.gz, pattern: some pattern
```
As you can see at the end we got our arguments printed out nicely. Good. Now we can move on to trying to open the archive file. Now this is a tar.gz file, if you've not encountered these before they are a compressed wrapper around a tar file, which is just some metadata and a bunch of files concatenated together. We'll use a couple of libraries to handle these. Add the following to your dependencies in your `cargo.toml`

```toml
flate2 = "1.0.25"
tar = "0.4.38"
```

We can import the bits we will shortly need,

```rust
use std::fs::File;
use flate2::read::GzDecoder;
use tar::Archive;
```

Now we can open the file, wrap that file in a gz decoder, and wrap that in a tar archive decoder, like so:

```rust
    let tar_gz = File::open(archive_path).unwrap();
    let tar = GzDecoder::new(tar_gz);
    let mut archive = Archive::new(tar);
```

Lets do a test and print out the filenames in the archive, to make sure things are working as we expect.

```rust
    for entry in archive.entries().unwrap() {
        let e = entry.unwrap();
        let path_path = e.path().unwrap();
        let path_string = path_path.to_str().unwrap();
        println!("file: {}", path_string);
    }
```

Now if we run it we should get a list of filenames printed out...

```bash
cargo run -- -a ../../Downloads/archive-20230214153535-daa49fde73e802447b521186ba95c8af.tar.gz "some pattern"
```

```
   Compiling cfg-if v1.0.0
   Compiling adler v1.0.2
   Compiling windows_x86_64_msvc v0.42.1
   Compiling windows-targets v0.42.1
   Compiling crc32fast v1.3.2
   Compiling windows-sys v0.45.0
   Compiling miniz_oxide v0.6.2
   Compiling flate2 v1.0.25
   Compiling filetime v0.2.20
   Compiling tar v0.4.38
   Compiling magrep2 v0.1.0 (C:\Users\emily\git\magrep2)
    Finished dev [unoptimized + debuginfo] target(s) in 4.58s
     Running `target\debug\magrep2.exe -a ..\..\Downloads\archive-20230214153535-daa49fde73e802447b521186ba95c8af.tar.gz "some pattern"`
path: ..\..\Downloads\archive-20230214153535-daa49fde73e802447b521186ba95c8af.tar.gz, pattern: some pattern
file: media_attachments/files/000/857/528/original/f55902524814b2f2.jpeg
... cropped for brevity but there is a load more media attachments in my archive...
file: media_attachments/files/109/834/160/834/865/079/original/ce8fe16924be4b35.jpg
file: outbox.json
file: likes.json
file: bookmarks.json
file: avatar.jpg
file: actor.json
```

There is the outbox.json file we want to have a look inside of, so lets see whats inside. The entry object we have above acts like a reader so we can read it to a string like any other file. Lets modify our loop from the previous entry to something like this

```rust
    for entry in archive.entries().unwrap() {
        let mut e = entry.unwrap();
        let path_path = e.path().unwrap();
        let path_string = path_path.to_str().unwrap();
        
        if path_string == "outbox.json" {
            let mut buffer = String::new();
            e.read_to_string(&mut buffer).unwrap();
            println!("{buffer}");
        }
    }
```

Note we need to make the entry e mutable here as the read_to_string method changes the position of of the read pointer in the "file"

I won't show the output here, suffice to say its a single giant line of json. So our next task is going to be decoding it. For this we need yet another dependency. This time `serde_json`. `serde` is a collection of libraries for encoding and decoding rust structs. The json library deals with json encoding and decoding. 

```toml
serde_json = "1.0.93"
```

At this point we could define a set of structs that could represent the json object structure inside the `outbox.json` file. However that seems like a lot of work considering we really only need to pick out a couple of fields, the content, and the url. Helpfully serde_json has a built in Value type that can represent any arbitrary json object. So we can use that and save a bunch of typing and code.

First as usual import the library

```rust
use serde_json::Value;
```

Then we can modify our loop:

```rust
    let mut outbox : Option<Value> = None;

    for entry in archive.entries().unwrap() {
        let e = entry.unwrap();
        let path_path = e.path().unwrap();
        let path_string = path_path.to_str().unwrap();

        if path_string == "outbox.json" && outbox.is_none() {
            outbox = Some(serde_json::from_reader(e).unwrap());
        }
    }

    if outbox.is_none() {
        println!("no outbox file in archive !invalid!");
        return
    }
```

Small change to our loop here to decode the json object. I'm making a style choice here to store the decoded outbox json object for later. I don't like to have things too deeply nested, they get difficult to understand and this allows us to handle the next bit outside the loop. We do have to take care of the case where we get a tar.gz file that doesn't contain an `outbox.json` file so we store it in a option and check that the option is not none after the loop.

So now we have the decoded json object out of the tar ball with out writing anything temporary to disk, which is nice, so now we can start searching for toots that match our `pattern`. Lets take a look at this json object we have and see what we can do with it. We could print it out but we already know that its huge and going to roll off the end of our shell if we try and print it all out. So lets have a look at what keys are in the top level and see if that helps.

```rust
    for key in outbox.unwrap().as_object().unwrap().keys() {
        println!("{}", key);
    }
```

Much unwrapping later... so we take our `Option<Value>` and unwrap it to a `Value` we know this is safe because we checked is_none() above and exited if it was. We then convert the Value to an object representation with the `as_object` call. This returns an `Option<&Map<String, Value>>`, we kind of know this will work and is safe to unwrap, but if it wasn't it would be ok to panic here as we are just playing around to figure out our data structure. Then we can get an iterator of the keys and loop over them.

If we run this we should get something like:

```
   Compiling ryu v1.0.12
   Compiling serde v1.0.152
   Compiling itoa v1.0.5
   Compiling serde_json v1.0.93
   Compiling magrep2 v0.1.0 (C:\Users\emily\git\magrep2)
    Finished dev [unoptimized + debuginfo] target(s) in 7.86s
     Running `target\debug\magrep2.exe -a ..\..\Downloads\archive-20230214153535-daa49fde73e802447b521186ba95c8af.tar.gz "some pattern"`
path: ..\..\Downloads\archive-20230214153535-daa49fde73e802447b521186ba95c8af.tar.gz, pattern: some pattern
@context
id
orderedItems
totalItems
type
```

That `orderedItems` key looks promising. Lets take a closer look. Hopefully its an array (if the world makes any sense ... we should probably check) lets try it and see what explodes

```rust
println!("{}", outbox.unwrap()["orderedItems"].as_array().unwrap()[0]);
```

This should output something like this, my first ever toot! (Hay nothing exploded!)

```
{"actor":"https://tech.lgbt/users/Emily_S","cc":["https://tech.lgbt/users/Emily_S/followers"],"id":"https://tech.lgbt/users/Emily_S/statuses/103363936687819074/activity","object":{"atomUri":"https://tech.lgbt/users/Emily_S/statuses/103363936687819074","attachment":[],"attributedTo":"https://tech.lgbt/users/Emily_S","cc":["https://tech.lgbt/users/Emily_S/followers"],"content":"<p>Going to try this mastodon thing again. </p><p>Hello everyone. I&#39;m Emily. A trans lady from the UK. (Yes we made a mess of things lately didn&#39;t we) software engineer, space nerd, maker of things and maps, parent and wife.</p><p>I&#39;m interested in lots of data processing stuff, I work in a geographic field so you&#39;ll probably get rants about projection systems, buggy code, interesting code tricks, and other general stuff.</p>","contentMap":{"en":"<p>Going to try this mastodon thing again. </p><p>Hello everyone. I&#39;m Emily. A trans lady from the UK. (Yes we made a mess of things lately didn&#39;t we) software engineer, space nerd, maker of things and maps, parent and wife.</p><p>I&#39;m interested in lots of data processing stuff, I work in a geographic field so you&#39;ll probably get rants about projection systems, buggy code, interesting code tricks, and other general stuff.</p>"},"conversation":"tag:tech.lgbt,2019-12-24:objectId=3818591:objectType=Conversation","id":"https://tech.lgbt/users/Emily_S/statuses/103363936687819074","inReplyTo":null,"inReplyToAtomUri":null,"published":"2019-12-24T17:28:26Z","replies":{"first":{"items":[],"next":"https://tech.lgbt/users/Emily_S/statuses/103363936687819074/replies?only_other_accounts=true&page=true","partOf":"https://tech.lgbt/users/Emily_S/statuses/103363936687819074/replies","type":"CollectionPage"},"id":"https://tech.lgbt/users/Emily_S/statuses/103363936687819074/replies","type":"Collection"},"sensitive":false,"summary":null,"tag":[],"to":["https://www.w3.org/ns/activitystreams#Public"],"type":"Note","url":"https://tech.lgbt/@Emily_S/103363936687819074"},"published":"2019-12-24T17:28:26Z","signature":{"created":"2023-02-14T15:32:30Z","creator":"https://tech.lgbt/users/Emily_S#main-key","signatureValue":"sdTf+YO4x3og1ApJWIOZ5sEsRq49cCJe35ZJdUzOX+jfna6nMzaWUeAgv4Maz0Iig3SJ7ODNqrONghfQPUdfp6ObwUKwWsSmIfA3mvGZsgElfpkkKFLzqCHGWnW+dOLN6CebFKUCMf5YpWwPvFrxAX4bzs/EAnjCnMK43VWFS/srQo7BEBYnv9qkNeBfR2UNz0xkjQ0HP2YeGOXavNlCovNyN6zSyenFA66h3vNehX3un4szMCdM/u0LodU3pKSonyw8kTBaBpkzkqFHmlTQ5jX+0ZyA/+szFE6FkdTR8zG16bjuvxXoBezyq1rKRxjvGPhWehThYk8YpLtsVl28Yg==","type":"RsaSignature2017"},"to":["https://www.w3.org/ns/activitystreams#Public"],"type":"Create"}
```

Ok, so we have the content of our toot twice for internationalisation reasons by the look of it. (I guess something got extended at some point in the history of activity pub and now we have both content and contentMap fields) the other one that looks promising is the `atomUri` which looks a lot like a toot url. It is, we can put that into a browser and it brings up the toot in question.

So lets finish this thing and be done.

```rust
    for item in outbox.unwrap()["orderedItems"].as_array().unwrap() {
        if item["object"]["content"].as_str().unwrap().contains(pattern) {
            println!("*************");
            println!("{} : {}",item["object"]["atomUri"], item["object"]["content"]);
        }
    }
```

So we loop over all the items in the orderedItems list and then get the content as a string and see if it contains the pattern. If so print out its atomUri and the content so we can see it.

Unfortunately if we run this we'll get a panic

```
   Compiling magrep v0.1.0 (C:\Users\emily\git\magrep)
    Finished dev [unoptimized + debuginfo] target(s) in 1.47s
     Running `target\debug\magrep.exe -a ..\..\Downloads\archive-20230214153535-daa49fde73e802447b521186ba95c8af.tar.gz "some pattern"`
path: ..\..\Downloads\archive-20230214153535-daa49fde73e802447b521186ba95c8af.tar.gz, pattern: some pattern
thread 'main' panicked at 'called `Option::unwrap()` on a `None` value', src\main.rs:52:37
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
error: process didn't exit successfully: `target\debug\magrep.exe -a ..\..\Downloads\archive-20230214153535-daa49fde73e802447b521186ba95c8af.tar.gz "some pattern"` (exit code: 101)
```

What this is telling us is we tried to call unwrap on a option that was None, on line 52 of main.rs. Which is the if statement in the above block. The only unwrap in there is the as_str() call so we got a content that wasn't a string? More likely we got an item that didn't have a content field at all. Lets print some things out and see what happens.

```rust
    for item in outbox.unwrap()["orderedItems"].as_array().unwrap() {
        println!("%%%%%%%%%%%%%%%%");
        println!("{}", item);
        if item["object"]["content"].as_str().unwrap().contains(pattern) {
            println!("*************");
            println!("{} : {}", item["object"]["atomUri"], item["object"]["content"]);
        }
    }
```

I like to include separators so its easy to see whats going on, hence the '%' symbols. With luck when we run this now we should be able to see the cause of the panic.

```
...
%%%%%%%%%%%%%%%%
{"@context":"https://www.w3.org/ns/activitystreams","actor":"https://tech.lgbt/users/Emily_S","cc":["https://tiny.tilde.website/users/selfsame","https://tech.lgbt/users/Emily_S/followers"],"id":"https://tech.lgbt/users/Emily_S/statuses/103370741914550623/activity","object":"https://tiny.tilde.website/users/selfsame/statuses/101789047613657946","published":"2019-12-25T22:19:06Z","to":["https://www.w3.org/ns/activitystreams#Public"],"type":"Announce"}
thread 'main' panicked at 'called `Option::unwrap()` on a `None` value', src\main.rs:54:47
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
error: process didn't exit successfully: `target\debug\magrep.exe -a ..\..\Downloads\archive-20230214153535-daa49fde73e802447b521186ba95c8af.tar.gz "some pattern"` (exit code: 101)
```

Ah, its something that doesn't have a content field inside the object. In fact the object is just a string. I think this is a boost, which is not something we want to search so we can ignore them. I remember now I did mention this way way back at the start of this post, oops. That `type` field looks like a good candidate for picking out what we need. Lets add a filter on that being "Create"

```rust
    for item in outbox.unwrap()["orderedItems"].as_array().unwrap() {
        if item["type"].as_str().unwrap() == "Create" {
            if item["object"]["content"].as_str().unwrap().contains(pattern) {
                println!("*************");
                println!("{} : {}", item["object"]["atomUri"], item["object"]["content"]);
            }
        }
    }
```
And if we run it now we get...

```
   Compiling magrep v0.1.0 (C:\Users\emily\git\magrep)
    Finished dev [unoptimized + debuginfo] target(s) in 1.20s
     Running `target\debug\magrep.exe -a ..\..\Downloads\archive-20230214153535-daa49fde73e802447b521186ba95c8af.tar.gz "some pattern"`
path: ..\..\Downloads\archive-20230214153535-daa49fde73e802447b521186ba95c8af.tar.gz, pattern: some pattern
```
Ah I've never tooted "some pattern" lets try something else...

```
PS C:\Users\emily\git\magrep> cargo run -- -a ..\..\Downloads\archive-20230214153535-daa49fde73e802447b521186ba95c8af.tar.gz "fish"
   Compiling magrep v0.1.0 (C:\Users\emily\git\magrep)
    Finished dev [unoptimized + debuginfo] target(s) in 1.66s
     Running `target\debug\magrep.exe -a ..\..\Downloads\archive-20230214153535-daa49fde73e802447b521186ba95c8af.tar.gz fish`
path: ..\..\Downloads\archive-20230214153535-daa49fde73e802447b521186ba95c8af.tar.gz, pattern: fish
*************
"https://tech.lgbt/users/Emily_S/statuses/109295945016110296" : "<p>So let&#39;s dig in to some usecases: </p><p>Say your country has a system to license vessels to fish in your waters, but also can&#39;t really afford a big navy. With ais data you can track vessels in your waters and deal with any that look like they are fishing.</p><p>Boats that are fishing move very differently to a cargo ship trying to get to the next port, so even if it lies about being a fishing vessel you can tell.</p>"
*************
"https://tech.lgbt/users/Emily_S/statuses/109295960022052842" : "<p>What about verification that the fish you are selling in your super market is caught legally? This is a good one because it&#39;s just making sure vessels are doing what they say they are, so they will want to keep their transmitters on to prove they are good and can sell their fish.</p><p>You can also keep track of your cargo containers and know where the shipments are out side of what the shipping company says. That snafu in the suez was fun to watch.</p>"
*************
"https://tech.lgbt/users/Emily_S/statuses/109451801563109639" : "<p><span class=\"h-card\"><a href=\"https://scicomm.xyz/@delibrarian\" class=\"u-url mention\">@<span>delibrarian</span></a></span> good question. For the purposes of story telling I&#39;d go with nothing, but it acting like a fish bowl where they can&#39;t come here and we can&#39;t leave would be interesting.</p>"
```

Apparently I've tooted the word fish three times. And thats our problem solved. Now there is a bunch of stuff that can be improved here. Error handling being the big one. We are unwrapping a bunch of things here, which we should likely be checking. Also it could happily be broken down into more reusable functions, but if I ever actually need this code again I'll do that. Right now it would be a waste of time, and I need to go toot "some pattern".

If you want to see the entire thing the code is on [github](https://github.com/emilyselwood/magrep) Any questions or comments you can find me on [mastodon](https://tech.lgbt/@Emily_S). 

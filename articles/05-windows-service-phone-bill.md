# The Windows Service That Ate My Phone Bill (and Other Lessons From My First Serious Project)

*Eleven years ago I built an RSS podcast scraper with a runtime C# compiler baked into it. It worked perfectly — right up until it bankrupted my cellphone contract during a house move.*

---

Every developer has a project that taught them more than any course, bootcamp, or senior developer ever could. Mine was a podcast scraper I built in 2013, and the lesson it taught me had nothing to do with code.

But I'll get to the phone bill. First, let me tell you about the absurdly over-engineered thing I built, why I built it, and what it looks like through the eyes of someone who's spent the decade since writing enterprise software.

## The Origin Story

I was obsessed with drum and bass. Not casually — obsessively. DnBRadio, Bassdrive, various Feedburner-hosted podcast feeds. Every week, new mixes would drop as RSS-based podcasts, and I wanted all of them downloaded automatically onto my machine, catalogued by artist, genre, and show.

iTunes existed. Podcast apps existed. But they didn't let me build a database of every show ever published across multiple feeds, with custom parsing rules for each feed's slightly different XML format, automatically downloading the audio files to organized folder structures on disk.

So I did what any reasonable person would do: I built a Windows Service that ran as a background daemon, polled RSS feeds on a timer, parsed the XML, matched entries against a rule engine, inserted metadata into a SQL Server database, and downloaded the MP3 files to disk.

In 2013. As a relatively junior developer. With no one to code review me.

The result was glorious, functional, and horrifying.

## The Architecture (Such As It Was)

The solution had six projects, which for a podcast downloader is at least four too many:

A **Windows Service** (`RSSListener`) that ran on a 60-second timer, polling every configured podcast feed on each tick. A **Console Application** (`RSSListenerConsole`) for testing the same logic without installing the service. A **Data Model** project with entity classes for podcasts, shows, artists, genres, and XML definitions. A **SQL layer** with hand-written `SqlConnection` and `SqlCommand` wrappers — no ORM, no Dapper, just raw ADO.NET with string-concatenated queries. A **Repository Manager** project containing a single empty class called `Brain.cs` (aspirational naming at its finest). And a **Rules Engine** that is simultaneously the worst and best code in the entire solution.

The data flow was straightforward: the service fetched each podcast's RSS feed URL, downloaded the XML to `c:\temp\myxml.xml` (yes, a hardcoded temp path, shared across all feeds, overwritten each time), parsed it using `XmlDocument` and XPath queries, extracted show metadata, and inserted it into SQL Server. If the podcast was configured for automatic download, it would then use `System.Net.WebClient` to download each episode's audio file.

Each podcast feed had a different XML structure — different tag names for titles, descriptions, enclosures, publish dates. Rather than hardcoding parsers for each feed, I stored XML definitions in the database: XPath expressions for where to find each piece of data, plus references to transformation rules that could extract and clean the values.

## The Rule Engine (Where Junior Me Got Ambitious)

This is the part I'm still genuinely impressed by, even though the implementation makes me wince.

Each XML definition could reference a `Rule2` object (yes, `Rule2` — there was presumably a `Rule1` at some point, lost to history). A rule contained a C# class name, method name, method signature, return type, using statements, and the actual C# code as a string.

At runtime, the rule engine would:

1. Assemble a complete C# source file from these database fields
2. Compile it in memory using `CSharpCodeProvider` and `CompilerParameters`
3. Create an instance of the compiled class via reflection
4. Invoke the specified method, passing in the raw XML data
5. Return the transformed result

I had built a plugin system. A configurable, database-driven, runtime code compilation and execution engine. To parse podcast titles.

The thing is — it worked. When a new podcast feed had a weird title format like "ArtistName - ShowTitle [Genre] (2013-04-15)", I didn't need to redeploy the service. I inserted a new rule into the database with a C# string manipulation method, and the next polling cycle would use it. Different feeds could have completely different parsing logic without touching the service code.

Was it a security disaster? Absolutely — arbitrary C# execution from database strings is the kind of thing that gets you escorted out of a security review. Was it over-engineered for a personal project? Wildly. But the *idea* was sound: separate the parsing logic from the parsing infrastructure, make it configurable without redeployment, and handle the reality that every data source is slightly different.

I've since seen this same pattern implemented properly in enterprise systems — plugin architectures with sandboxed execution, configurable transformation pipelines, rule engines with proper compilation caching and error isolation. The instinct was right. The execution was eleven years too early for my skill level.

## The Phone Bill Incident

Here's where the story earns its title.

The service was configured to run every 60 seconds. On each tick, it would check all configured podcasts, download any new XML feeds, parse them, and if `AutomaticDownload` was set to `true`, download the audio files for any episodes that weren't already on disk.

This worked beautifully when the service was running on my desktop PC, connected to my home broadband. Hundreds of megabytes of podcast audio, downloaded in the background, organized into folders. The dream.

Then I moved houses.

During the move, my desktop was packed up, but my laptop — which also had the service installed for testing — was running. Connected to my phone's mobile hotspot, because the new place didn't have broadband set up yet.

I was carrying boxes. The service was carrying out its mission. Every 60 seconds, it checked for new episodes. And when it found them, it downloaded them. Over my mobile data connection. For hours.

I discovered the problem when my mobile carrier sent me a notification that I'd exceeded my data cap by a margin that I would describe as "financially significant." The service had been faithfully downloading gigabytes of podcast audio over a metered 3G connection while I was physically hauling furniture between houses.

The code worked perfectly. The deployment context was catastrophic. No amount of unit testing would have caught this, because the bug wasn't in the code — it was in the assumption that "this machine has cheap bandwidth" would always be true.

## What I See Now That I Couldn't See Then

Reading this code with ten years of additional experience is like reading your own diary from high school — recognizable, embarrassing, and occasionally surprising.

**The things that make me cringe:**

No async/await anywhere. Every HTTP request, every database call, every file operation blocks the thread. The service uses a `System.Timers.Timer` and a boolean flag (`iscurrentlyinuse`) to prevent overlapping executions — a homemade mutex that would fail spectacularly under race conditions.

No dependency injection. The `SqlConnection` is created in `OnStart()` and passed around as a constructor parameter to everything. If the connection drops, the entire service dies.

No connection pooling, no retry logic, no circuit breakers. If a podcast feed is temporarily unavailable, the service logs an error and moves on, but if SQL Server hiccups, it's game over.

Every entity class uses public fields instead of properties. The SQL layer uses string-formatted queries (though mercifully, most of the input comes from the database itself rather than user input, limiting the injection surface).

A temp file at a hardcoded path (`c:\temp\myxml.xml`) that gets overwritten on every feed refresh. If two feeds were somehow processed concurrently, they'd corrupt each other's data.

**The things that impress me:**

The separation between feed structure (XML definitions) and feed processing (the rule engine) was architecturally sound. The idea that each feed's parsing behavior should be configurable without code changes is the right instinct, even if the implementation was reckless.

The data model is thoughtful. Shows are linked to artists, artists to genres, genres to subgenres. There's a podcast XML archival table that stores the raw feed data alongside the parsed results, giving you an audit trail. The logging system has severity levels and multiple output targets (event log, text file).

The service was packaged with an InstallShield installer (`.isl` and `.isproj` files are in the solution). For a personal project, that's a level of polish that suggests someone who cared about the craft, even if the craft was still developing.

And `Brain.cs` — the empty class with a single `using` statement in the Repository Manager project — is the most honest piece of code in the entire solution. Every developer has created a class with grand ambitions that never got filled in. At least I had the decency to name it accurately.

## The Actual Lesson

The phone bill incident taught me something that took years to fully internalize: **code doesn't run in a vacuum.** It runs on machines, over networks, under resource constraints, in environments you didn't anticipate when you wrote it.

The service had no concept of bandwidth cost. No concept of metered versus unmetered connections. No concept of "maybe don't download 4GB of audio files over a mobile hotspot." These aren't code quality issues — they're context awareness issues. The service was a perfect implementation of the wrong assumptions.

In the enterprise software I write today, I think about this constantly. What happens when the network is slow? What happens when the database connection drops mid-transaction? What happens when the disk is full, the queue backs up, the third-party API rate-limits you, or the user's browser has 47 tabs open? The code that handles the happy path is easy. The code that handles reality is where the engineering lives.

Eleven years later, I still have the database somewhere. 4,200+ podcast episodes catalogued, tagged, and organized. The service hasn't run since the move. But the lessons from building it — and from paying the phone bill that resulted — are baked into every piece of software I've written since.

---

*Gregory Rebisz is a software developer and technical content writer who once over-engineered a podcast downloader so thoroughly that it had its own rule-based compiler. He's since applied those same instincts to enterprise SaaS platforms, where the stakes are higher but the debugging is eerily similar. He's based in South Africa and available for freelance technical writing.*

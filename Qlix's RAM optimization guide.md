# Introduction
As I see it, the ultimate challenge in Bitburner is to write a suite of scripts that fully automate the entire process of destroying any Bit Node with a single command (without using any truly broken exploits), even with SF 1 disabled, meaning you only have 8GiB on home to get the party started. I'm nowhere near accomplishing this, but I am scripting with that end goal in mind, and have made enough progress refining my optimization techniques that I think it's time to write a guide explaining them. It's a niche that appears to be as yet unfilled. Complete scripts will not be provided at this time (though I may eventually put everything  on GitHub); just an overview with snippets, similar to the in-game documentation describing basic and advanced techniques for automatic hacking.

(If you don't know what I mean by a Bit Node or SF 1, don't worry about it. That will be revealed to you later in the game, and you don't need to know to follow this guide.)

This guide is a work in progress. More advanced techniques are coming soon.

Written for Bitburner v2.8.1 and last updated 2025-07-10.
# Preliminaries: Ports
Understanding ports is crucial to understanding some of the techniques described here. Ports in Bitburner are queues that are identified by positive integers and enable communication between scripts. They have two additional characteristics that make them incredibly useful in optimization:

- They are shared across all servers, making them more versatile than JSON files (next section).
- They are 100% free of charge! Every function for working with ports (you should familiarize yourself with them in the NS documentation if you haven't already) costs 0GiB.


Compared to JSON files, they also have the advantages of creating no clutter and being awaitable.

## Assigning Port Numbers
There are two basic strategies here:

- Enumerate your ports in a utility file, then import the enumeration wherever you use ports
- Use the PID of the script that first uses a port as the port number


You can mix and match these by including a PID offset in the enumeration, after all the named ports. To get a dynamic port number, add the PID to the offset.

## Pitfalls to Avoid
Using ports makes debugging more difficult. If you're reading "NULL PORT DATA" when you shouldn't be, it's probable that either you forgot to await a port write or you have two scripts reading from the same port and causing problems for each other (two instances of the same script, typically).

Ports are also error-prone in scripts that persist across saves. It's important to remember that on load, every persistent script will start running from the top, not where it left off. So, if a persistent script is expecting port data to be immediately available when it starts running, that script is going to have a bad time and possibly crash. Any script that has such expectations should be temporary.

If you're hitting errors immediately on load and you're sure that's not the problem, try sleeping for a few seconds near the top of the script that errors out, before doing anything with ports. There's a minor bug (I don't know the details; I just know I've run into it and found a workaround) that can (might not, but can) cause problems if you don't do this. Sleeping for a few seconds at the top of persistent scripts is, incidentally, also good practice for protection against infinite loops (gives you time to hit the kill all button immediately on load, which makes a big difference for those of us who find that the built-in kill all & reload feature does not work properly).
# Basic: JSON Files
Many Netscript functions (e.g., getResetInfo) return data that changes infrequently, if at all (within a given Bit Node), and use RAM to do it. For such info, write a script that gets it and writes it to a file like so:

```
ns.write("info.json", JSON.stringify(ns.getInfo), "w");
```

(Note the "w" at the end. By default, ns.write appends to existing files, which causes problems if you're expecting the file to be overwritten, as you probably are when generating JSON.)
Later, other scripts running on the same server can read that data as much as they want, at no RAM cost, like so:

```
JSON.parse(ns.read("info.json"));
```

# Basic: Careful Naming
There are two RAM costs per script: the static cost and the dynamic cost. The dynamic cost is calculated while scripts are running. Each call to a Netscript function that has not previously been called adds to it. It's always accurate. Static costs, on the other hand, are calculated before the script starts running and determine how much RAM the script is allocated. If dynamic cost ever exceeds static cost, the script crashes. Bitburner mostly handles this automatically so you don't have to worry about it, but it's not very smart about calculating the static cost.

[screenshot](https://steamcommunity.com/sharedfiles/filedetails/?id=3521407947)

In this example script, I've declared a function and an object property that have the same names as two Netscript functions. When I then call the function and access the property, Bitburner mistakenly thinks I'm using the Netscript functions, and increases the static cost accordingly. Now this script takes up 0.25GiB more RAM than it actually needs.

Avoiding this problem is pretty simple: just don't use property/function names that are also used in Netscript. Alternately, in the case of property names, access the property with brackets and a string like this:

```
ns.print(obj["weaken"]);
```


Or, if you really want to, you can use ns.ramOverride to set the static cost yourself, but I don't recommend doing that without a much better reason than wanting to use "hack" as a variable name.
# Basic: Avoiding Unnecessary Checks
You might think, for example, that you should check whether you have enough money to buy a server before calling the function to buy a server. Don't bother. Checking first costs a little RAM, and if you don't have enough money, then ns.purchaseServer will simply return a falsy value to signal that it failed, which is fine. The important thing is that the script won't crash and you'll still be able to detect and handle the case where you don't have enough money. Many Netscript functions work like this.
# Basic: Modularization with ns.run and ns.exec
Suppose you have a script that looks like this:

```
ns.expensiveFunction();
while(true) {
    \\ Other functions
}
```

This is a good candidate for modularization. ns.expensiveFunction can be moved into its own script, which you can then run with ns.run or ns.exec (the difference being that ns.run uses the same server as the parent script, while ns.exec can use any server with enough RAM). This will cost 1GiB for the former or 1.3GiB for the latter, plus 1.6GiB for the second script (only while it's running, which may be just a few milliseconds), but many NS functions are expensive enough to make it worth doing. It's even more worth doing if you can factor out additional functions into separate scripts. However many times you call ns.run, it still only costs 1GiB. Sleeping for a few milliseconds between ns.run or ns.exec calls keeps RAM cost down by preventing child scripts from running concurrently.

Use ns.exec only where you need to run scripts on other servers, and where you use it, never use ns.run in the same script. If you use both, you're paying for both unnecessarily.

## Intermediate Level Refinement
Need to pass data to your child scripts that's more complex than a string or an integer? That's what ports are for. Write the data to a port, and read the same port in the child script.

Need to return data from the child script to the parent? Again, ports! Write the return value to a port in the child script, then in the parent, await a port write before reading.

Want to pass a port number from the parent to the child rather than hard-coding it? Try this in the child script:

```
const flags = ns.flags([["p", 0]]);
```

Pass "-p" followed by the port number as arguments to the child script, and it will be available as flags.p.

Need to make sure ns.run doesn't fail for lack of RAM? Put it in a while loop like so:

```
while (!ns.run(...args))
    await ns.sleep(20);
```

# Basic: Script Chaining with ns.spawn
Suppose you have a script that looks like this:

```
ns.expensiveFunction1();
ns.expensiveFunction2();
ns.expensiveFunction3();
ns.expensiveFunction4();
```

Sure, you could modularize with ns.run, or you could turn this into a chain of four scripts. Each will call only one of the expensive functions, and the first three will follow that up with a call to ns.spawn to run the next link in the chain. To create a loop, you could then have the fourth spawn the first.

ns.spawn itself costs a steep 2GiB, but has the advantage that the parent script and child script will not be running concurrently. That secures a guarantee that there will be sufficient RAM for the child script on the assumption that its static RAM allocation is no greater than the parent script's static RAM allocation.

To give a concrete example, I purchase the prerequisites for the TIX API this way.

## Pitfalls to Avoid
The aforementioned guarantee comes with a caveat: by default, ns.spawn starts the child script 10 seconds after terminating the parent. Not only is the delay generally mildly inconvenient, it gives other scripts opportunity to eat up the RAM that the child script needs. If that happens, the child script will fail to run and the parent will be able to do nothing about it because the parent already terminated. To prevent this and really secure that guarantee, pass {"spawnDelay": 0} to ns.spawn as spawnOptions.
# Intermediate: Shims
Some functions (in particular, the Go board analysis functions) can be reimplemented by the player. Reimplementations (AKA shims) are, of course, free to call assuming they themselves call no functions that aren't free. I'm marking this one as intermediate despite the conceptual simplicity because actually doing the work of writing shims is typically a challenge.
# Advanced: Executor Daemon
No relation to w0r1d-d4em0n. In software dev, a daemon is any process that runs in the background, without user interaction. Here, I'm using the term to refer to a looping script that does nothing but handle commands from other scripts. We can use such a script to make even more efficient use of ns.exec. The basic idea is simple enough: the daemon waits for writes to a particular named port. Each value read from that port is assumed to be an array of arguments, which the daemon passes to a call to ns.exec. Now only the daemon needs to pay the 1.3GiB for ns.exec, after which it's still small enough that it can run on n00dles. Other scripts can just write to its port for free and get the same effect!

In theory, anyway.

In practice, you need to be careful with this one. The latency introduced by waiting for port writes may be a problem in some cases. Worse, port queues have limited capacity. Exceed that capacity by writing faster than the daemon can read and data - which in this case means calls to ns.exec - will be silently lost. So, it's not at all suitable for high-volume purposes like batch hacking.

And in any event, using an executor daemon goes a step further in making it difficult to track what your code is doing, and therefore difficult to debug.

## More Advanced Features
Once you've got the daemon working, here are some ideas for improving it:

- An option to delay execution of a script would effectively give you a better ns.spawn - one that doesn't limit you to spawning a single script.
- Rather than have client scripts specify hostnames, scan all servers for one that has enough free RAM to run the script, and you won't have to worry quite so much about scripts failing to run.


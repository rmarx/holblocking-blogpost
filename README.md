# Will HTTP/3 really be faster than HTTP/2? Perhaps

As you may have heard, after 4 years of work, the new HTTP/3 and QUIC protocols are finally approaching official standardization. Preview versions are now [available for testing in servers and browsers alike][h3WithCaddy]. 

[h3WithCaddy]: https://ma.ttias.be/how-run-http-3-with-caddy-2/

HTTP/3 is promising major performance improvements compared to HTTP/2, mainly because it [changes its underlying transport protocol from TCP to QUIC][quicIntro]. In this post, we'll be taking an in-depth look at just one of these improvements, namely the removal of the **"Head-of-Line blocking" (HOL blocking) problem**. This is useful because I've read a lot of misconceptions on what this actually means and how much it helps in practice. Solving HOL blocking was also one of the main motivations behind not just HTTP/3 and QUIC but also HTTP/2, so it gives a fantastic insight in the reasons for protocol evolution as well.  

[quicIntro]: https://ma.ttias.be/googles-quic-protocol-moving-web-tcp-udp/

I'll first introduce the problem and then track different forms of it throughout HTTP's history. To make this post accessible for novices, we will touch on and explain many other aspects of modern Web protocols as well (like prioritization and congestion control). The goal is to help people make correct assumptions about HTTP/3's performance improvements, which might not be as amazing as sometimes claimed in marketing materials. Spoiler: much like the [cake][cake], HOL blocking removal in HTTP/3 is probably a lie and probably won't lead to much speedups in practice.  

[cake]: https://knowyourmeme.com/memes/the-cake-is-a-lie

## What is Head-of-Line blocking?

It's difficult to give you a single technical definition of HOL blocking, as this blogpost alone describes four different variations of it. A simple, conceptual definition however would be:

> When a single (slow) object prevents other/following objects from making progress

A good real-life metaphore would be a grocery store with just a single check-out counter. One customer who's buying a lot of items can end up delaying everyone behind them, as customers are served in a First In, First Out manner. Another example could be a highway with just a single lane. One car crash on this road can end up jamming the entire passage for a long time. As such, even a single issue at he "head of the line" can "block" the entire line. 

As it turns out, that concept has been one of the hardest Web performance problems to solve. To understand this, let's start at its incarnation in our trusted workhorse: HTTP version 1.1 

## HOL blocking in HTTP/1.1

HTTP/1.1 is a protocol from a simpler time. A time when protocols could still be text-based and readable on the wire. This is illustrated in Figure 1 below:

![server HTTP/1.1 response for script.js](https://github.com/rmarx/holblocking-blogpost/raw/master/images/1_H1_1file.png)
*Figure 1: server HTTP/1.1 response for `script.js`*

In this case, the browser requested the simple `script.js` file (green) over HTTP/1.1, and Figure 1 shows the server's response to that request. We can see that the HTTP aspect itself is really straightforward: it just adds some textual "headers" (red) directly in front of the plaintext file content, that's it. Headers + content are then passed down to the underlying TCP (orange) for actual transport to the receiver. For this example, let's pretend we cannot fit the entire file into 1 TCP packet and it has to be split up into two parts.

*Note: in reality, when using HTTPS, there is another layer in between HTTP and TCP for security with the TLS protocol. However, we omit that in our discussion here for clarity. I did include a [bonus section at the end of this post](#sec_tls) that details a TLS-specific HOL blocking variant and how QUIC prevents it.* 

Now let's see what happens when the browser also requests `style.css` in Figure 2:

![server HTTP/1.1  response for script.js and style.css](https://github.com/rmarx/holblocking-blogpost/raw/master/images/2_H1_2files.png)
*Figure 2: server HTTP/1.1  response for `script.js` and `style.css`*

In this case, we are sending `style.css` (purple) after the response for `script.js` has been transmitted. The headers and content for `style.css` are simply appended after the JavaScript (JS) file. The receiver uses the **Content-Length** header for each response to know where one ends and the other starts (in our simplified example, let's pretend `script.js` is 1000 bytes large, while `style.css` is just 600 bytes).  

All of that seems sensible enough in this simple example with two small files. However, let's imagine a scenario in which the JS file is much larger than CSS (say 1MB instead of 1KB). In this case, the CSS would have to wait before the entire JS file was downloaded, even though it is much smaller and thus could be parsed/used earlier, which can be better for Web performance. If we visualize this a bit more directly, using the number 1 for `large_script.js` and 2 for `style.css`, we would get something like this:

> 11111111111111111111111111111111111111122

You can see this is an instance of the Head-of-Line blocking problem! Now you might think: that's easy to solve! Just have the browser request the CSS file before the JS file! Crucially however, the browser has **no way of knowing up-front which of the two will end up being the larger file at request time**, as there is no way to for instance indicate in the HTML how large a file is (this would be lovely, HTML working group: `<img src="thisisfine.jpg" size="15000" />`).  

The "real" solution to this problem would be to employ **multiplexing**. If we can cut up each file into smaller pieces (instead of sending them as one big blob), we can "interleave" those chunks on the wire (send a chunk for the JS, one for the CSS, then another for the JS again, etc. until the files are downloaded). With this approach, the smaller CSS file will be downloaded (and usable) much earlier, while only delaying the larger JS file by a bit. Visualized with numbers we would get:

> 12121111111111111111111111111111111111111

Sadly however, this multiplexing is not possible in HTTP/1.1 due to some fundamental limitations with the protocol's assumptions. To understand this, we don't even need to keep looking at the large-vs-small resource scenario, as it already shows up in our example with the two smaller files. Consider Figure 3, where we interleave just 4 chunks for the two resources:

![server HTTP/1.1 multiplexing for script.js and style.css](https://github.com/rmarx/holblocking-blogpost/raw/master/images/3_H1_2files_multiplexed.png)
*Figure 3: server HTTP/1.1 multiplexing for `script.js` and `style.css`*
 
The main problem here is that HTTP/1.1 is a purely textual protocol that only appends headers to the front. It does nothing further to differentiate individual (chunks of) resources from one another. As such, in Figure 3, the browser starts parsing the headers for `script.js` and expects 1000 bytes of response body (the Content-Length). It however only receives 450 JS bytes (the first JS chunk) and then starts reading the headers for `style.css`. It ends up interpreting the CSS headers and the first CSS chunk as part of the JS, as the file contents and headers for both files are just plaintext. Making matters worse, it stops after reading 1000 bytes, ending up somewhere halfway through the second `script.js` chunk. At this point, it doesn't see valid new headers and has to drop the rest of TCP packet 3. The browser then passes this weird text blob to the JS parser, which fails because it's not valid JavaScript:

```
function first() { return "hello"; }
HTTP/1.1 200 OK
Content-Length: 600

.h1 { font-size: 4em; }
func
```

Again, you could say there's an easy solution: have the browser look for the `HTTP/1.1 {statusCode} {statusString}\n` pattern to see when a new header starts. That might work for TCP packet 2, but will fail in packet 3: how would the browser know where the green `script.js` chunk ends and the purple `style.css` chunk begins? 

This all is a fundamental limitation of the way the HTTP/1.1 protocol was designed. It means that, if you have a single HTTP/1.1 connection, resource responses always have to be delivered **in-full** before you can switch to sending a new resource. This can lead to severe HOL blocking issues if earlier resources are slow to create (for example a dynamically generated `index.html` that is filled from database queries) or, as above, if earlier resources are simply quite large.

This is why browsers [started opening multiple parallel TCP connections][parallelConnections] (typically 6 nowadays) for each page load over HTTP/1.1. That way, requests could be distributed across those individual connections and there is no more HOL blocking. That is, unless you have more than 6 resources per page... That's where the practice of "sharding" your resources over multiple domains and Content Delivery Networks (CDNs) comes from: as each sharded domain gets 6 connections, browsers will open up to 30-ish TCP connections in total for each page load. This works, but has considerable overhead: setting up a new TCP connection can be expensive (for example in terms of state and memory at the server) and takes some time (especially for an HTTPS connection). 

[parallelConnections]: http://www.stevesouders.com/blog/2008/03/20/roundup-on-parallel-connections/

As this problem cannot be solved with HTTP/1.1 and the patchwork solution of parallel TCP connections didn't scale too well over time, it was clear a totally new approach was needed, which is what became HTTP/2. 

*Note: the old guard reading this might wonder about HTTP/1.1 pipelining. I decided not to discuss that here to keep the overall story flowing, but people interested in even more technical depth can read the [bonus section at the end of this post](#sec_pipelining).

<a name="sec_http2"></a>
## HOL blocking in HTTP/2 over TCP

So, let's recap. We now know that HTTP/1.1 has a HOL blocking problem where a large or slow response can end up delaying other and smaller responses behind it. This is mainly because the protocol is purely textual in nature and doesn't use delimiters between resource chunks. As a workaround, browsers open many parallel TCP connections, which is not efficient and doesn't scale.

As such, the goal for HTTP/2 was quite clear: make it so that we can **move back to a single TCP connection by solving the HOL blocking problem**. Stated differently: we want to enable proper multiplexing of resource chunks. This wasn't possible in HTTP/1.1 because there was no way to discern to which resource a chunk belonged, or where it ended and another began. HTTP/2 solves this quite elegantly by prepending small control messages, called **frames**, before the resource chunks. This can be seen in Figure 4:

![server HTTP/1.1 vs HTTP/2 response for script.js](https://github.com/rmarx/holblocking-blogpost/raw/master/images/4_H2_1file.png)
*Figure 4: server HTTP/1.1 vs HTTP/2 response for `script.js`*

Unlike HTTP/1.1, HTTP/2 puts a so-called DATA frame in front of each chunk. These DATA frames mainly indicate two critical pieces of metadata. First: which resource the following chunk belongs to (each resource "stream" is assigned a unique number, the **stream id**). Second: how large the following chunk is. The protocol has many other frame types as well, of which Figure 5 also shows the HEADERS frame. This again uses a stream id to indicate which response these headers belong to (so that headers can even be split up from their actual response data).

Using these frames, it follows that HTTP/2 indeed allows proper multiplexing of several resources on one connection, see Figure 5:

![multiplexed server HTTP/2 responses for script.js and style.css](https://github.com/rmarx/holblocking-blogpost/raw/master/images/5_H2_2files_multiplexed.png)
*Figure 5: multiplexed server HTTP/2 responses for `script.js` and `style.css`*

Unlike our example for Figure 3, the browser can now deal with this situation perfectly. It first processes the HEADERS frame for `script.js` (which also has a length built-in) and then the DATA frame for the first JS chunk. Because the DATA frame includes length of the chunk and this is perfectly contained in TCP packet 1, it knows that it needs to look for a completely new frame starting in TCP packet 2, where it indeed finds the HEADERS for `style.css`. The next DATA frame has a different stream id (2) than the first DATA frame (1), so the browser knows this belongs to a different resource and can process it independently. The same happens for TCP packet 3, where the DATA frame stream ids are used to "de-multiplex" the response chunks to their correct resource "streams". 

We can see that, by "framing" individual messages, HTTP/2 is much more flexible than HTTP/1.1. It allows for many resources to be sent multiplexed on a single TCP connection by interleaving their chunks. It also solves HOL blocking in the case of a slow first resource: instead of waiting for the database-backed index.html to be generated, the server can simply start sending data for other resources in the mean time while it waits for index.html. 

An important consequence of HTTP/2's approach is that we suddenly also need a way for the browser to communicate to the server how it would like the single connection's bandwidth to be distributed across resources. Put differently: how resource chunks should be "scheduled" or interleaved. If we again visualize this with 1's and 2's, we see that for HTTP/1.1, the only option was 11112222 (let's call that sequential). HTTP/2 however has a lot more freedom:

> - Fair multiplexing (for example two progressive JPEGs): 12121212
> - Weighted multiplexing (2 is twice as important as 1): 221221221
> - Reversed sequential scheduling (for example 2 is a key Server Pushed resources): 22221111  
> - Partial scheduling (stream 1 is aborted and not sent in full): 112222

Which of these is used exactly is driven by the so-called "prioritization" system in HTTP/2 and the chosen approach can have a big impact on Web performance. That is however a very complex topic by itself and you don't really need to understand it for the rest of this blogpost, so I've left it out here (though I do have [an extensive lecture on this on YouTube][prioritiesVideo]).

[prioritiesVideo]: https://www.youtube.com/watch?v=nH4iRpFnf1c

I think you'll agree that, with HTTP/2's frames and its prioritization setup, it indeed solves HTTP/1.1's HOL blocking problem. This means my work here is done and we can all go home. Right? Well, not so fast there bucko. [We've solved HTTP/1.1 HOL blocking, yes, but what about TCP HOL blocking](https://4.bp.blogspot.com/-n4LJF-HJfS4/VuhfpUkYOxI/AAAAAAAAPTQ/H0I9ZU-lJGMY0dURTJZW-DwE_WenWooqQ/s1600/hobbit%2Bsecond.gif)? 

### TCP HOL blocking

As it turns out, HTTP/2 only solved HOL blocking at the HTTP level, what we might call "Application Layer" HOL blocking. There are however other layers below that to consider in the typical networking model. You can see this clearly in Figure 6:

![the top few protocol layers in the typical networking model](https://github.com/rmarx/holblocking-blogpost/raw/master/images/6_layers.png)
*Figure 6: the top few protocol layers in the typical networking model.*

HTTP is at the top, but is supported first by TLS at the Security Layer (see the [Bonus TLS section](#sec_tls)), which in turn is carried by TCP at the Transport layer. Each of these protocols wrap the data from the layer above it with some metadata (for example the TCP packet header is prepended to our HTTP data, while that then gets put inside an IP packet etc.). This allows for a relatively neat separation between the protocols. This in turn is good for their re-usability: a Transport Layer protocol like TCP doesn't have to care about what type of data it is transporting (it could be HTTP, it could be FTP, it could be SSH, who knows), and IP works fine for both TCP and UDP. 

This does however have important consequences if we want to multiplex multiple HTTP/2 resources onto one TCP connection. Consider Figure 7:

![difference in perspective between HTTP/2 and TCP](https://github.com/rmarx/holblocking-blogpost/raw/master/images/7_H2_bytetracking.png)
*Figure 7: difference in perspective between HTTP/2 and TCP*

While both we and the browser understand we are fetching JavaScript and CSS files, even HTTP/2 doesn't (need to) know that. All it knows is that it is working with chunks from different resource stream ids. However, **TCP doesn't even know that it's transporting HTTP!** All TCP knows is that it has been given a series of bytes that it has to transport from one computer to another, for which it uses packets of a certain maximum size. Each packet just tracks which part of the data (byte range) it carries so the original data can be reconstructed in the correct order.  

Put differently, there is a mismatch in perspective between the two Layers: HTTP/2 sees multiple, independent resource bytestreams, but TCP sees just a single, opaque bytestream. An example is Figure 7's TCP packet 3: TCP just knows it is carrying byte 750 to byte 1599 of whatever it is transporting. HTTP/2 on the other hand knows there are actually two chunks of two separate resources in packet 3. *(Note: In reality, each HTTP/2 frame (like DATA and HEADERS) is also a couple of bytes in size. For simplicity, I haven't counted that extra overhead or the HEADERS frames here to make the numbers more intuitive.)*

All of this might seem like unnecessary details, until you realize that the Internet is a fundamentally unreliable network. Packets can and do get lost and delayed during transport from one endpoint to the other. This is exactly one of the reasons why TCP is so popular: it ensures reliability on top of the unreliable IP. It does this quite simply by **retransmitting copies of lost packets**. 

We can now understand how that can lead to HOL blocking at the Transport Layer. Consider again Figure 7 and ask yourself: what should happen if TCP packet 2 is lost in the network, but somehow packet 1 and packet 3 do arrive? Remember that TCP doesn't know it's carrying HTTP/2, just that it needs to deliver data in-order. As such, it knows the contents of packet 1 are safe to use and passes those to the browser. However, it sees that there is a gap between the bytes in packet 1 and those in packet 3 (where packet 2 fits), and thus cannot yet pass packet 3 to the browser. TCP keeps packet 3 in its receive buffer until it receives the retransmitted copy of packet 2 (which takes at least 1 RTT), after which it can pass both to the browser in the correct order. Put differently: **the lost packet 2 is HOL blocking packet 3**!

It might not be clear why this is a problem though, so let's dig deeper by looking at what is actually inside the TCP packets at the HTTP layer in Figure 7. We can see that TCP packet 2 carries only data for stream id 2 (the CSS file) and that packet 3 carries data for both streams 1 (the JS file) and 2. At the HTTP level, we know those two streams are independent and clearly delineated by the DATA frames. As such, we could in theory perfectly pass packet 3 to the browser without waiting for packet 2 to arrive. The browser would see a DATA frame for stream id 1 and would be able to directly use it. Only stream 2 would have to be put on hold, waiting for packet 2's retransmit. This would be more efficient than what we get from TCPs approach, which ends up blocking both stream 1 and 2. 

Another example is the situation where packet 1 is lost, but 2 and 3 are received. TCP will again hold back both packets 2 and 3, waiting for 1. However, we can see that at the HTTP/2 level, the data for stream 2 (the CSS file) is present completely in packets 2 and 3 and doesn't have to wait for packet 1's retransmit. The browser could have perfectly parsed/processed/used the CSS file, but is stick waiting for the JS file's retransmit. 

In conclusion, the fact that TCP does not know about HTTP/2's independent streams means that **TCP-Layer HOL blocking (due to lost or delayed packets) also ends up HOL blocking HTTP**. 

Now, you might ask yourself: then what was the point? Why do HTTP/2 at all if we still have HOL blocking? Well, the main reason is that while packet loss does happen on networks, it is still relatively rare. Especially on high-speed, cabled networks, packet loss rates are on the order of 0.01%. Even on the worst cellular networks, you will rarely see rates higher than 2% in practice. This is combined with the fact that packet loss and also jitter (delay variations in the network), are often **bursty**. A packet loss rate of 2% does not mean that you will always have 2 packets out of every 100 being lost (for example packet nr 42 and nr 96). In practice, it would probably be more like 10 **consecutive** packets being lost in a total of 500 (say packet nr 255 to 265). This is because packet loss is often caused by temporarily overflowing memory buffers in routers in the network path, which start dropping packets they cannot store. Again though, the details aren't important here (but [available elsewhere if you'd like to know more][routerloss]). What is important is that: yes, TCP HOL blocking is real, but it has a much smaller impact on Web performance than HTTP/1.1 HOL blocking, which you are almost guaranteed to hit every time *and* which also suffers from TCP HOL blocking! 

[routerloss]: https://ripe80.ripe.net/presentations/5-2020-05-12-buffers.pdf

However, this is mainly true when comparing HTTP/2 on a single connection with HTTP/1.1 on a single connection. As we've seen before, that's not really how it works in practice, as HTTP/1.1 typically opens multiple connections. This allows HTTP/1.1 to somewhat mitigate not only the HTTP-level but also the TCP-level HOL blocking. As such, in some cases, HTTP/2 on a single connection has a hard time being faster than or even as fast as HTTP/1.1 on 6 connections. This is mainly due to TCP's **"congestion control"** mechanism. This is however yet another very deep topic we don't have time to explore in-depth here, so you'll have to take my word for it that the following conclusions are correct (if sometimes overly simplified!) ;) 

The congestion controller's main job is to make sure the network isn't overloaded by too much data at the same time. If that happens, the router buffers start overflowing, and we have packet loss, as described above. So, what it typically does is start by sending just a little bit of data (usually about 14KB) to see if that makes it through. If that works, it doubles that every RTT until a packet loss event is observed (meaning the network is overloaded (a bit) and it needs to back down (a bit)). This is how a TCP connection "probes" for its available bandwidth. Crucially: **this mechanism works for each TCP connection independently**!

Firstly, this means that HTTP/2's single connection initially only sends 14KB. However, HTTP/1.1's 6 connections each send 14KB in their first flight, which is about 84KB! This will compound over time, as each HTTP/1.1 connection doubles its data use each RTT. Secondly, a connection will only lower its sending rate if there is packet loss. For HTTP/2's single connection, even a single packet loss means it will slow down (in addition to causing TCP HOL blocking!). However, for HTTP/1.1, a single packet loss on just one of the connections will only slow down that one: the other 5 can just keep sending and growing as normal.

This all makes one thing very clear: **HTTP/2's multiplexing is not the same as HTTP/1.1's downloading resources at the same time**(something which I also still see some people claiming). The available bandwidth of the singular HTTP/2 connection is simply distributed across/shared among the different files, but chunks are still sent sequentially. This is different from HTTP/1.1, where things are sent in a truly parallel fashion. 

By now, you might be wondering: **how then can HTTP/2 ever be faster than HTTP/1.1**? That is a good question, one that I have been asking myself on and off for a long time as well. One obvious answer is in cases where you have -many- more than 6 files. This is how HTTP/2 was marketed back in the day: [by splitting an image into tiny squares and loading those over HTTP/1.1 vs HTTP/2][gophertiles]. This mainly shows off HTTP/2's HOL blocking removal. For normal/real websites however, things get a lot more nuanced quickly. It depends on the amount of resources, their size, the used prioritization/multiplexing scheme, the RTT to the server, how much loss there actually is and when it happens, how much other traffic is on the link at the same time, the used congestion controller logic, etc. One example where HTTP/1.1 can lose is on networks with limited available bandwidth: the 6 HTTP/1.1 connections each grow their send rate individually, causing them to overload the network quite quickly, after which they all have to back down and have to find their co-existent bandwidth limit through trial and error (prior to HTTP/2, [it was thought that HTTP/1.1's parallel connections could be a main cause of packet loss on the Internet][h1packetlossrates]). The single HTTP/2 connection instead grows more slowly, but it is faster to recover after a packet loss event and finds its optimal bandwidth faster.  

[h1packetlossrates]: https://a77db9aa-a-7b23c8ea-s-sites.googlegroups.com/a/chromium.org/dev/spdy/An_Argument_For_Changing_TCP_Slow_Start.pdf
[gophertiles]: https://http2.golang.org/gophertiles

**TODO: add refs: BBR, prioritization, fairness, etc.**

All in all, in practice, we see that (perhaps unexpectedly), HTTP/2 as it is currently deployed in browsers and servers is typically as fast or slightly faster than HTTP/1.1 in most conditions. This is in my opinion partly because websites got better at optimizing for HTTP/2, and partly because browsers often still open multiple parallel HTTP/2 connections (either because sites still [shard their resources over different servers][h2sharding], or because of security-related side-effects), thus getting the best of both worlds.

[h2sharding]: https://twitter.com/zachleat/status/1055219667894259712?s=20
[almanacH2]: https://almanac.httparchive.org/en/2019/http2

**TODO: link to Barry's H2 chapter in the web almanac**
**TODO: link to jake archibald's blogpost on "credentialed vs non-credentialed" connections + Matt Hobbs' examples from WPT**

However, there are also some cases (especially on slower networks with higher packet loss) where HTTP/1.1 on 6 connections will still outshine HTTP/2 on one connection, which is often due to the TCP-level HOL blocking problem. It is this fact that was a large motivator for the development of the new QUIC transport protocol as a replacement for TCP. 


<a name="sec_http3"></a>
## HOL blocking in HTTP/3 over QUIC

After all that, we're finally ready to start talking about the new stuff! But first, let's summarize what we've learned so far:

> - HTTP/1.1 had HOL blocking because it needs to send its responses in full and cannot multiplex them
> - HTTP/2 solves that by introducing "frames" that indicate to which "stream" each resource chunk belongs
> - TCP however does not know about these individual "streams" and just sees everything as 1 big stream
> - If a TCP packet is lost, all later packets need to wait for its retransmission, even if they contain unrelated data from different streams. TCP has Transport Layer HOL blocking.

I'm pretty sure you can by now predict how we can solve TCP's issues, right? After all, the solution is qutie simple: we "just" need to **make the Transport Layer aware of the different, independent streams** as well! That way, if data for one stream is lost, the Transport itself knows it does not need to hold back the other streams. 

While this is indeed simple enough in concept, in practice it has taken over 6 years to get this approach ready for prime-time in the new QUIC Transport Layer protocol and the accompanying HTTP/3. This blogpost is however already way too long to also discuss why this couldn't just be added to TCP instead, why QUIC runs on UDP, and so on. I will instead just focus on the parts that we need to understand our current HOL blocking discussion, which should be easy enough by looking at Figure 8:

![server HTTP/1.1 vs HTTP/2 vs HTTP/3 response for script.js](https://github.com/rmarx/holblocking-blogpost/raw/master/images/8_H3_1file.png)
*Figure 8: server HTTP/1.1 vs HTTP/2 vs HTTP/3 response for `script.js`*

We observe that making QUIC aware of the different streams was pretty straightforward. QUIC was inspired by HTTP/2's framing-approach and also adds its own frames; in this case the STREAM frame. Mainly the stream id, which was previously in HTTP/2's DATA frame, is now **moved down to the Transport Layer in QUIC's STREAM frame**. This also shows one of the reasons why we needed a new version of HTTP if we want to use QUIC: if we would just run HTTP/2 on top of QUIC, we would have two (potentially conflicting) "stream layers". HTTP/3 instead removes the stream concept from the HTTP layer and re-uses the underlying QUIC streams instead. 

*Note: this doesn't mean QUIC suddenly knows about JS or CSS files or even that it is transporting HTTP (like TCP, QUIC is supposed to be a generic, re-usable protocol). It just knows that there are independent streams that it can handle individually, without having to know what exactly is in them.*

Now that we know about QUIC's STREAM frames, it's also easy to see how they help solve Transport Layer HOL blocking in Figure 9:

![difference in perspective between TCP and QUIC](https://github.com/rmarx/holblocking-blogpost/raw/master/images/9_H3_bytetracking.png)
*Figure 9: difference in perspective between TCP and QUIC*

Much like HTTP/2's DATA frames, QUIC's STREAM frames track the byte ranges for each stream individually. This in contrast to TCP, which just appends all stream data in one big blob. Like before, let's consider what would happen if QUIC packet 2 is lost but 1 and 3 arrive. Like with TCP, the data for stream 1 in packet 1 can just be passed to the browser. However, for packet 3, QUIC can be smarter than TCP. It looks at the byte ranges for stream 1 and sees that this STREAM frame perfectly follows the first STREAM frame for stream id 1 (in other words: there are no byte gaps in the data). It can thus give that data to the browser for processing as well. For stream id 2 however, QUIC does see a gap (it hasn't received bytes 0-299 yet, those were in the lost QUIC packet 2). It will hold on to that STREAM frame until the retransmission of QUIC packet 2 arrives. Contrast this again to TCP, which also held back stream 1's data in packet 3!

Something similar happens in the other situation where packet 1 is lost but 2 and 3 arrive. QUIC knows it has received all expected data for stream 2 and just passed that along to the browser, only holding up stream 1. We can see that for this example, indeed, QUIC solves TCP's HOL blocking!

This approach has couple of important consequences though. 

- loss detection is done per stream
- means that data is no longer deterministically ordered... sent 1 2 3, but arrived 2 3(partial) 1 3(partial)...
- this in turn is the second, arguably more important reason for HTTP/3: many systems in HTTP/2 really rely on TCP's fully deterministic ordering. HTTP/3 re-imagines both prioritization and header compression for that (HPACK to QPACK).
- For an in-depth discussion for prioritization, see my paper. For qpack, see dmitri's excellent blogs on the subject. 

- Still, it's worth it to remove HOL blocking, right? Well, have we actually removed it? Still have per-stream HOL blocking... that's something we're never going to be able to remove, as we still want resources to be processed in-order. 
- Luckily, if one stream is HOL blocked, we can still make progress on the rest, right?
- well... if there is "the rest"... Now, resource multiplexing comes into play: sequential vs RR 

![impact of stream multiplexing on HOL blocking prevention in HTTP/3 over QUIC](https://github.com/rmarx/holblocking-blogpost/raw/master/images/10_H3_scheduler.png)
*Figure 10: impact of stream multiplexing on HOL blocking prevention in HTTP/3 over QUIC*

- we can see: if sequential = still holblocking, little benefit from QUIC. Only if we have some form of Round-Robin (and preferably not per-packet, because remember loss is bursty!) does it help in practice.
- Why does this matter? Sequential is thought to be better for web performance... so we usually won't have multiple streams being multiplexed, so we will typically still have per-stream HOL blocking in practice... darned

- other points: QUIC has different streams and you could say each stream is similar to one TCP connection in terms of loss detection.
- However: still just a single congestion controller! So it won't be as fast as the 6 HTTP/1.1 connections!


<a name="sec_conclusion"></a>
## Conclusion

- So... in theory: yes, HTTP/3 over QUIC solves HOL blocking
- In practice: the jury is still out on how much this will matter.
- Either way: it's not much of an issue in practice, especially on fast networks. HTTP/2 solved the "main" HOL blocking problem.

- So... will HTTP/3 be faster overall? Probably, but it won't be (much) because of HOL blocking removal. 
- Then why? 0-RTT connection establishment. However, many aspects (Amplification prevention, replay attacks, initial congestion window size, etc.0 that's for a future post (or read my paper or watch my youtube videos).

- Overall: it's too early to say. It will help for some use cases for sure, but maybe not that much for most typical websites (at least not ones that aren't explicitly optimized for HTTP/3). One of the research questiosn I'm looking into is if it makes sense to do multiplexing differently if there is more packet loss to get the HOL blocking benefits, but it's unclear if that'll matter much. 

- If you liked this, I'm working on a book about QUIC and HTTP/3: let me keep you up to date!

<a name="sec_pipelining"></a>
## Bonus: HTTP/1.1 pipelining

HTTP/1.1 includes a feature called "pipelining" which is in my opinion often misunderstood. I've seen many posts and even books where people claim that HTTP/1.1 pipelining solves the HOL blocking issue. I've even seen some people saying that pipelining is the same as proper multiplexing. Both statements are [false](https://theofficeanalytics.files.wordpress.com/2017/11/dwight.jpeg?w=1200). 

I find it easiest to explain HTTP/1.1 pipelining with an illustration like the one in Bonus Figure 1:

![HTTP/1.1 pipelining](https://github.com/rmarx/holblocking-blogpost/raw/master/images/bonus1_H1_pipelining.png)
*Bonus Figure 1: HTTP/1.1 pipelining*

Without pipelining (left side of Bonus Figure 1), the browser has to wait to send the second resource request until the response for the first request has been completely received (again using Content-Length). This adds one Round-Trip-Time (RTT) of delay for each request, which is bad for Web performance.  

With pipelining then (middle of Bonus Figure 1), the browser does not have to wait for any response data and can now send the requests back-to-back. This way we save some RTTs during the connection, making the loading process faster. *As a side-note, look back at Figure 2: you see that pipelining is actually used there as well, as the server bundles data from the `script.js` and `style.css` responses in TCP packet 2. This is of course only possible if the server received both requests around the same time.*

Crucially however, this pipelining only applies to the **requests** from the browser. As the [HTTP/1.1 specification][h1spec] says:

> A server MUST send its responses to those [pipelined] requests in the same order that the requests were received.

As such, we see that actual multiplexing of response chunks (illustrated on the right side of Bonus Figure 1) is still not possible with HTTP/1.1 pipelining. Put differently: **pipelining solves HOL blocking for requests, but not for responses**. Sadly, it's arguably the response HOL blocking that causes the most issues for Web performance. 

To make matters worse, most browsers actually do not use HTTP/1.1 pipelining in practice because it can make HOL blocking more unpredictable in the setup with multiple parallel TCP connections. To understand this, let's imagine a setup where three files A (large), B (small) and C (small) are being requested from the server over two TCP connections. A and B are each requested on a different connection. Now, on which connection should the browser pipeline the request for C? As we said before, it doesn't know up-front whether A or B will turn out to be the largest/slowest resource. 

If it guesses correctly (B), it can download both B and C in the time it takes to transfer A, resulting in a nice speedup. However, if the guess is wrong (A), the connection for B will be idle for a long time, while C is HOL blocked behind A. This is because HTTP/1.1 also does not provide a way to "abort" a request once it's been sent (something which HTTP/2 and HTTP/3 do allow). The browser can thus not simply request C over B's connection once it becomes clear that is going to be the faster one, as it would end up requesting C twice. 

To get around all this, modern browsers do not employ pipelining and will even actively delay requests of certain discovered resources (for example images) for a little while to see if more important files (say JS and CSS) are found, to make sure the high priority resources are not HOL blocked. 

It is clear that the failure of HTTP/1.1 pipelining was another motivation for HTTP/2's radically different approach. Yet, as HTTP/2's prioritization system to steer multiplexing often fails to perform in practice, some browsers have even taken to delaying HTTP/2 resource requests as well to get optimal performance. 

**TODO: add pipelining-in-browsers + prioritization-delaying-in-chrome refs (dig through my papers for these)**

<a name="sec_tls"></a>
## Bonus: TLS HOL blocking

- TLS encrypts in blocks of up to 16KB for efficiency
- if the last X of those blocks are lost, we cannot decrypt the first N - X blocks
- so there is a kind of TLS-specific HOL blocking happening if the final chunks of a TLS record are lost

- this is one of the reasons that QUIC encrypts one packet at a time
- This is less efficient (and one of the main reasons QUIC requires more CPU than TCP), but prevents QUIC-level HOL blocking even in these edge cases.


[tlsSizing]: https://www.igvita.com/2013/10/24/optimizing-tls-record-size-and-buffering-latency/
[tlsSizing2]: https://blog.cloudflare.com/optimizing-tls-over-tcp-to-reduce-latency/


## References


# Does HTTP/3 really fix Head-of-Line blocking?

As you may have heard, after 4 years of work, the new HTTP/3 and QUIC protocols are finally approaching official standardization. Preview versions are now [available for testing in servers and browsers alike](https://ma.ttias.be/how-run-http-3-with-caddy-2/). 

HTTP/3 is promising major performance improvements compared to HTTP/2, mainly because it [changes its underlying transport protocol from TCP to QUIC](https://ma.ttias.be/googles-quic-protocol-moving-web-tcp-udp/). In this post, we'll be taking an in-depth look at just one of these improvements, namely the promised removal of the **"Head-of-Line blocking" (HOL blocking) problem**. This is useful because I've read a lot of misconceptions on what this actually means and how much it helps in practice. Solving HOL blocking was also one of the main motivations behind not just HTTP/2 but also HTTP/3 and QUIC. 

I'll first introduce the problem and then track different forms of it throughout HTTP's history. Spoiler: much like the [cake](https://knowyourmeme.com/memes/the-cake-is-a-lie), HOL blocking removal in HTTP/3 is probably a lie. 

## What is Head-of-Line blocking?

It's difficult to give you a single technical definition of HOL blocking, as this blogpost alone describes four different variations of it. A simple, conceptual definition however would be:

> When a single (slow) object prevents other/following objects from making progress

A good real-life metaphore would be a grocery store with just a single check-out counter. One customer who's buying a lot of items can end up delaying everyone behind them, as customers are served in a First In, First Out manner. Another example could be a highway with just a single lane. One car crash on this road can end up jamming the entire passage for a long time. As such, even a single issue at he "head of the line" can "block" the entire line. 

As it turns out, that concept has been one of the hardest Web performance problems to solve. To understand this, lets start at its incarnation in our trusted workhorse: HTTP version 1.1 

## HOL blocking in HTTP/1.1

HTTP/1.1 is a protocol from a simpler time. A time when protocols could still be text-based and readable on the wire. This is illustrated in Figure 1 below:

[TODO: Figure 1]
*Figure 1: server HTTP/1.1 response for `script.js`*

In this case, the browser requested the simple `script.js` file (green) over HTTP/1.1, and Figure 1 shows the server's response to that request. We can see that the HTTP aspect itself is really straightforward: it just adds some textual "headers" (red) directly in front of the plaintext file content, that's it. Headers + content are then passed down to the underlying TCP (orange) for actual transport to the receiver. For this example, let's pretend we cannot fit the entire file into 1 TCP packet and it has to be split up into two parts.

*Note: in reality, when using HTTPS, there is another layer in between for security, with the TLS protocol. However, we omit that in our discussion here for clarity. If you want to know more about how TLS impacts things, see the [bonus section at the end of this post](TODO bonus section).* 

Now let's see what happens when the browser also requests `style.css` in Figure 2:

[TODO: Figure 2]
*Figure 2: server HTTP/1.1  response for `script.js` and `style.css`*

In this case, we are sending `style.css` (purple) after the response for `script.js` has been transmitted. In this case, the headers and content for `style.css` are simply appended after the JavaScript file. The receiver uses the **Content-Length** header for each response to know where one ends and the other starts (in our simplified example, `script.js` is 1000 bytes large, while `style.css` is just 600 bytes).  

All of that seems sensible enough. But let's consider what happens if we would like to send the JavaScript and CSS **at the same time**? (we will discuss [later in this blogpost](TODO scheduling) whether that's a good idea in practice or not). 






cannot be done, so browsers open parallel connections to get around this. 


- note that Figure 2 is actually showing pipelined requests. Otherwise, data for `script.js` and `style.css` couldn't share packet 2. 


- one response at a time
- because data is passed raw to TCP (or TLS, doesn't matter) without framing
- cannot be mixed and matched

- pipelining changes sending requests, but not responses, so no solution

## HOL blocking in HTTP/2 over TCP

- H2 introduces framing, so yay, multiplexing!
- However, still on 1 TCP stream, so vulnerable to packet loss and re-ordering

- However: much less of a problem in practice: low % is packet loss + bursty (important later)

## HOL blocking in HTTP/3 over QUIC

- QUIC takes H2's streams over to the transport (put differently: each QUIC stream is its own TCP connection. like setting up as many TCP connections as you want with 1 handshake)
- This does get rid of Transport-layer HOL blocking due to packet loss, because each stream does retransmits individually
- Or does it...

- Note: we abstract away a lot. Normally, the sizes of the FRAMES would count as well, but we omit those here to get nice round numbers.

- Consider that, per stream, data also needs to be delivered in-order of course (put differently: previously we were HOL blocked inter-stream, now only intra-stream)
- Now, resource multiplexing comes into play: sequential vs RR 
- we can see: if sequential = still holblocking, little benefit from QUIC

- Why does this matter? Sequential is thought to be better for web performance...

## Conclusion

- So... in theory: it's solved
- In practice: the jury is still out. 
- Either way: it's not much of an issue in practice, especially on fast networks

## Bonus: TLS HOL blocking



## References


## original email for reference while writing

HTTP/1.1's HOL problem was at the application layer. You can't start sending response data for request 2 before the response to request 1 is fully downloaded (not even when using "pipelining")
HTTP/2 solves this application-layer HOL problem. This means that you now -can- start sending response data for request 2 before 1 is finished. 

This comes up in a few different ways:
- if request 1 becomes outdated you can simply stop sending its response and switch to only response 2 (e.g., this happens in video streaming, where old sections of the video might no longer be needed)
- if you want to send request 1 and 2 at the same time (multiplexing/bandwidth sharing. Mainly useful if the file can be used incrementally, like progressive jpegs/HTML/video. This is the typical example)
- If request 2 is of a higher priority than request 1 (in HTTP/1.1, to get around this, browsers often delay sendings requests until they can determine the optimal order by reading the HTML)
- If you want to use Server Push and the pushed stream is more important than the requesting stream
- (there are probably others)

Put differently, in HTTP/1.1, you always have to do:  111111222222, while in HTTP/2, you can do: 121212121212, 111222222, 222222111111, 221221221221, etc.
That is already a massive benefit to HTTP/1.1.


Now, you're correct in stating that this does not entirely solve all HOL problems, as they also occur on the TCP level. 
Because TCP doesn't know anything about HTTP/2, it just sees everything as a single bytestream, say: 12345678 (instead of HTTP/2, which knows it's really 11112222), and so TCP just knows that bytes have to be delivered in that exact order. 
That indeed also gives a HOL problem in the case of packet loss or heavy packet re-ordering: if say packet 4 is lost, but 5678 do arrive, they have to wait until the retransmission of 4 arrives. 
H2 knows that is not needed (as 5678 contain data for stream 2 and 4 is for stream 1 and they are independent) but TCP does not. 

Crucially though: that's only if you have packet loss. And while that -does- happen, it's not as much of a problem as you might think. 
Especially on wired connections, packet loss is typically extremely low (to the point that researchers sometimes have to manually drop packets when testing over the internet to observe this kind of behaviour). 
Another aspect is that packet loss is often bursty: it's not 1 in every 200 packets that is lost, but say groups of 10-20 every 2000-4000.
As such, TCP HOL blocking is real, but much less of an issue in practice and thus HTTP/2's solution at the application layer is still quite useful when compared to HTTP/1.1, as that has both the application- and transport-layer HOL problem *combined*. 

------------------

So, that's then one of the main reasons for doing QUIC: there, they took HTTP/2's concept of independent streams and brought it down to the transport layer. 
Now, QUIC also knows it's actually 11112222 and if the 4th packet is lost, it can still deliver packets 5 to 8 to for example the browser for processing (meaning image 1 might be stalled a bit, but image 2 keeps loading, which it wouldn't on TCP+HTTP/2).
HTTP/3 makes use of QUIC's streams, and so benefits from this as well.

Now, it's time to be critical about how much QUIC actually helps in practice though... As I said, it's only useful in situations with (a lot of) packet loss, which are uncommon.
A second problem is that for websites, most resources are NOT progressively load-able. For example JavaScript, CSS, PNGs, etc. all need to be downloaded fully to be usable. 
As such, you typically don't want to send data multiplexed like this: 11111111111222222222222222222111111111111111111111112222222222222222222221111111111111111111222222222222222222222, 
but rather sequentially like this: 1111111111111111111111111111111111111111111111111111111111111111222222222222222222222222222222222222222222222222222.

However, not all of those packets are typically "in flight" at the same time: due to congestion control, you'll first send only 10 packets, then 20, then 40, and then maybe go up linearly by 1 (41, 42, 43 packets).
So, when you're worried about web performance and loading resources sequentially (which e.g., Google Chrome does), you'd have say 20 packets on the wire, but they are all of resource 1... 
This means that if packet 11 of those 20 is lost, you'd still have HOL blocking for packet 12-20, because they are of the same stream as packet 11... 
(and per-stream, data of course still needs to be delivered in-order) basically undoing QUIC's "solution" to the HOL problem.

As such, QUIC will mainly help if in those 20 packets, you for example first have 10 packets of resource 1, and the next 10 of resource 2. Then, if a packet is lost, you can still make progress on the other stream. 
However, as I said, mixing stream data like that is typically worse for web performance anyway, so anything you gain from HOL removal might be undone by that fact alone.
This last aspect is however difficult to measure and estimate. We've done initial testing on this in our paper (see below), but it's not yet clear what the best approach is there.

-----------

Wew, what a wall of text :) I hope this makes things clearer. 
If you would like to know more about HOL blocking nuances, I have a paper that focuses on these aspects for QUIC and HTTP/3: https://h3.edm.uhasselt.be/files/ResourceMultiplexing_H2andH3_Marx2020.pdf
If you would like to know more about why sending resources sequentially is better for web performance, I have another YouTube talk: https://www.youtube.com/watch?v=nH4iRpFnf1c

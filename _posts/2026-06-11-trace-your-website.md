---
layout: post
title: "Trace a Website to the Edge of the Internet"
description: "Using only dig, curl, and traceroute, follow one web request from a laptop to the machine that answers it — and learn anycast, CDN caching, and BGP by watching real systems work."
tag: "Networking fundamentals"
readtime: "~12 min read"
standfirst: "Type a domain, press enter, a page appears. Between those two moments is one of the most elegant distributed systems ever built — and you can watch the whole thing work with three commands you already have installed."
date: 2026-06-11
slug: trace-your-website
---

<p>Most explanations of how the web works are diagrams. Boxes, arrows, a cloud labeled "Internet." They're fine, but they ask you to take the interesting parts on faith. This article takes the opposite approach: we'll point three standard command-line tools at a website and read what they actually say, layer by layer, until we've followed a single request from a laptop all the way to the machine that answers it.</p>

      <p>The three tools are <code>dig</code> (ask the DNS system a question), <code>curl</code> (make an HTTP request and inspect the response), and <code>traceroute</code> (map the network path packets take). Everything below uses <code>example.com</code> as a neutral stand-in; the output is representative of what a CDN-hosted site returns, lightly trimmed for clarity. Run the same commands against any busy site and you'll see the same patterns.</p>

      <h2>Step 1 — A name is not an address</h2>

      <p>Computers don't route to names; they route to numbers. Before anything else happens, your device has to translate the human-friendly domain into a machine-friendly IP address by asking the Domain Name System. <code>dig</code> lets you ask that same question directly and see the raw answer, with no browser caching in the way:</p>

      <pre><span class="c">$</span> <span class="k">dig</span> example.com +short
<span class="o">93.184.216.34</span></pre>

      <p>The <code>+short</code> flag strips the formality and returns just the answer. One name, one address. But busy sites usually return <em>several</em> addresses, and that detail turns out to matter:</p>

      <pre><span class="c">$</span> <span class="k">dig</span> cdn-hosted-site.example +short
<span class="o">151.101.0.10
151.101.64.10
151.101.128.10
151.101.192.10</span></pre>

      <p>Four addresses for one name. Your computer will try the first and fall back to the others if needed — that's the first layer of redundancy. But here's the twist that makes the modern web work, and it's invisible in this output: those addresses are very likely <strong>anycast</strong>. The same number is announced from data centers all over the planet at once, and the network quietly routes you to the nearest one. We'll prove that in Step 3.</p>

      <div class="note">
        <b>Why dig and not nslookup</b>
        If you come from a Windows background, <code>nslookup</code> does the same job. <code>dig</code> is the Unix-world equivalent and shows the full DNS response the way it travels on the wire — including record types, TTLs, and the alias chains we'll see next. Both ask the same question; <code>dig</code> just shows more of the answer.
      </div>

      <h3>The alias chain</h3>

      <p>Ask about the <code>www</code> version of a site and you'll often see a second record type appear — a <strong>CNAME</strong>, which is DNS for "this name is just another name for that name, go look that one up instead":</p>

      <pre><span class="c">$</span> <span class="k">dig</span> www.example.com

<span class="c">;; ANSWER SECTION:</span>
www.example.com.    3600  IN  <span class="o">CNAME</span>   target.cdn-provider.net.
target.cdn-provider.net.  60   IN  <span class="o">A</span>      151.101.0.10</pre>

      <p>The lookup resolves in a chain: <code>www</code> is an alias for the CDN's hostname, and <em>that</em> name carries the real address. The reason this exists is maintenance — an alias follows its target forever, so if the provider renumbers its servers, the alias updates itself automatically. One canonical record holds the truth; any number of aliases ride along for free. (A quirk worth knowing: DNS rules forbid a CNAME on the bare root domain, which is why the root usually carries hard-coded address records while only <code>www</code> gets the elegant alias.)</p>

      <h2>Step 2 — The address is a cache, not the website</h2>

      <p>Now we have an address. You might assume it points at "the server with the website on it." It almost never does. For any site behind a CDN, that address belongs to a <strong>cache server at the edge of the network</strong> — close to you geographically — and the real site lives somewhere else entirely. <code>curl</code> can show us the evidence, because the response headers confess the whole arrangement:</p>

      <pre><span class="c">$</span> <span class="k">curl</span> -sI https://example.com | grep -iE <span class="o">"x-cache|age|x-served-by|cache-control"</span>
cache-control: max-age=600
age: 0
x-served-by: <span class="o">cache-sea-xxxx</span>
x-cache: <span class="o">MISS</span></pre>

      <p>Read those four lines like a witness statement:</p>

      <ul>
        <li><code>x-served-by: cache-sea-xxxx</code> — the request was answered by a cache node in a Seattle-area point of presence. Not the origin; an edge.</li>
        <li><code>x-cache: MISS</code> — that edge did <em>not</em> already have the page in local storage, so it reached back to the origin server over the network, fetched the file, served it to me, and kept a copy.</li>
        <li><code>age: 0</code> — the copy is zero seconds old. Freshly fetched, just now.</li>
        <li><code>cache-control: max-age=600</code> — the origin's instruction to every cache: keep this copy for 600 seconds, then check back.</li>
      </ul>

      <p>The interesting part is what happens on the <em>second</em> request. Run the exact same command again within those 600 seconds:</p>

      <pre><span class="c">$</span> <span class="k">curl</span> -sI https://example.com | grep -iE <span class="o">"x-cache|age"</span>
age: <span class="o">42</span>
x-cache: <span class="g">HIT</span></pre>

      <p>Now it's a <span style="color:var(--ok)">HIT</span> — served straight from the edge's memory, with an <code>age</code> counting up the seconds since that first fetch populated it. The origin server never heard about this second request at all. <strong>This is the entire economic model of the modern web:</strong> populate a cache once, near each cluster of users, then serve thousands of requests from local memory without ever bothering the origin. It's the same idea as a DNS resolver caching answers — content delivery is just that pattern applied to web pages, with the time-to-live deciding how fresh "fresh" has to be.</p>

      <div class="note">
        <b>The two-tier model</b>
        Tier one is the fleet of edge caches answering on that anycast address — fast, local, temporary. Tier two is the origin, the authoritative copy, consulted only on a cache miss. When a site owner publishes a change, the caches expire and re-pull from origin — which is exactly why edits go live "in about a minute" rather than instantly: that minute is the TTL clock and propagation.
      </div>

      <h2>Step 3 — Following the packets to the edge</h2>

      <p>We've claimed the address is anycast and the server is nearby. <code>traceroute</code> lets us watch the packets actually travel there, printing each network hop along the way. With the <code>-a</code> flag it also annotates each hop with the autonomous system number — the identifier for the network that owns that router — which is what makes the path legible:</p>

      <pre><span class="c">$</span> <span class="k">traceroute</span> -a example.com
 1  [AS0]      10.0.0.1                          3 ms     <span class="c"># home router</span>
 2  [AS0]      10.x.x.x                          12 ms    <span class="c"># ISP private network</span>
 3  [AS7922]   xxx.seattle.isp-backbone.net      14 ms    <span class="c"># ISP, local metro</span>
 4  [AS7922]   xxx.seattle.isp-backbone.net      15 ms    <span class="c"># still the ISP</span>
 5  [AS7922]   xxx.seattle.isp-backbone.net      14 ms    <span class="c"># ISP edge / peering</span>
 6  [AS54113]  cdn-edge.cdn-provider.net         16 ms    <span class="c"># handoff to the CDN</span>
 7  [AS54113]  served-from-here                  16 ms    <span class="c"># the edge that answers</span></pre>

      <p>Read the autonomous-system numbers in brackets and the whole journey becomes obvious. Hops 1–2 are inside your home and your ISP's access network (the <code>[AS0]</code> just means private, unrouted address space). Hops 3–5 stay inside a single ISP — and notice the hostnames name a specific city, all in your own metro area, all under 15 milliseconds away. Hop 6 is the moment your packets cross from your ISP into the CDN's network — a single handoff between two organizations. Hop 7 is the edge server that answers.</p>

      <p>The headline: <strong>two networks, about seven hops, sixteen milliseconds, and the packets never left the metropolitan area.</strong> A website "on the internet" turned out to be served from a building a few miles away. That's anycast doing its job — the globally-announced address delivered you to the topologically nearest edge, which for most people is startlingly close.</p>

      <h3>When the trace goes dark — and why that's not a failure</h3>

      <p>Run traceroute against a CDN and you'll often see something alarming: the early hops resolve, then the rest of the output is nothing but asterisks.</p>

      <pre><span class="c">$</span> <span class="k">traceroute</span> example.com
 5  xxx.seattle.isp-backbone.net   14 ms
 6  <span class="o">* * *</span>
 7  <span class="o">* * *</span>
 8  <span class="o">* * *</span></pre>

      <p>This does <em>not</em> mean the site is unreachable — you just loaded it with <code>curl</code> a moment ago. It means the diagnostic plane is filtered, not that the data plane is down. The default <code>traceroute</code> sends UDP probes and listens for "time exceeded" replies; many CDN edges simply decline to send those replies, because they cost CPU and reveal internal structure to strangers. The page still loads perfectly; the network just stops <em>narrating</em> the path. Switch the probe type to ICMP and the edges often answer again:</p>

      <pre><span class="c">$</span> <span class="k">traceroute</span> -aI example.com   <span class="c"># -I uses ICMP echo instead of UDP</span></pre>

      <p>The distinction between "the path is broken" and "the path won't describe itself" is one every network engineer internalizes early, usually after filing one false outage ticket. The control plane and the data plane are independent: silence on one says nothing about the health of the other.</p>

      <h2>What you've actually seen</h2>

      <p>Three commands, and the abstract diagram became concrete and specific. A name resolved to four anycast addresses through a chain of DNS records. Those addresses turned out to be edge caches, not the website, populating themselves on first request and serving everyone after from local memory under a 600-second freshness contract. And the packets themselves traveled two networks and a handful of milliseconds to reach an edge server practically next door.</p>

      <p>None of this required special access or privileged tools — just curiosity pointed at the response headers and routing tables that are sitting there in plain sight on every request you make. The next time a page loads in the blink of an eye, you'll know it isn't magic. It's a cache near your house, answering on an address that exists in a hundred places at once, backed by an origin it only bothers when it has to. That's the architecture serving the entire web — and now you can watch it work.</p>

# The Most Dangerous Code in the World: Validating SSL Certificates in Non-Browser Software
_Tue Jan 28_

## TLS Handshake
Goals:
- Establish shared key
- Authentication of at least one side (client authenticates the server)
    - Server will generally use some other means like passwords to authenitcate the client, but nowadays with passkeys client authentication is becoming more popular.

### Diffie Hellman Key Exchange
- Generator : $g$
- Alice private key : $a$
- Alice public key : $g^a$ (sent to Bob)
- Bob private key: $b$
- Bob public key: $g^b$ (sent to Alice)
- Shared value is $g^{ab}$
- $Z_p$ what is this?
- Susceptible to MitM, since there's no binding between the identity of either party with their public key)

### Certificates
- So, use Public Key Infrastructure, which is Certificate Authorities signing bindings between identities and public keys
    - To do this attestation, _challenge_ matthew to host something at matthew.com, that kind of thing.
    - In your browser or software you'd have the public keys of your trusted CAs, which are often the root CAs who sign others down the line, until your "leaf" (website, whatever) is signed.
- When you're verifying a cert, make sure it
    - came from the right website
    - isn't revoked
    - isn't expired
    - is signed by a CA that you trust (which might be well down the chain, check all the way)

## SSL Problems
1. Pooly designed low level libraries, without sensible defaults
2. Middleware: libraries don't do all the extra work required to do things
3. Applications: ignore errors, end up trusting all certs, etc.

## How they find vulnerabilities
"Black box fuzzing"
- Using DNS to redirect stuff on their network, with bad certs
- MitM box to send invalid certs (checking both cert checking and hostname checking):
    1. self-signed cert with correct hostname
    2. self-signed cert with incorrect hostname
    3. valid cert, wrong domain

## Recommendations
- TEST your code!
- Need to rework a lot of SSL and networking software so it's secure by default, and you have to opt into insecure behavior
    - Since then, many people have forked OpenSSL, removed pieces, simplified

## Other
- TLS 1.3 is very different, set up to try and encrypt as much of the handshake as possible, to prevent MitM from changing out cipher specs, etc. `ClientHello` is the only unencrypted thing.
- Validation steps are all teh same across versions
- Only thing lacking in 2012 when this was published is __certificate transparency__ (we'll talk about that on Thursday)
# CRLite: A Scalable System for  Pushing All TLS Revocations to All Browsers
_Thu Jan 30_

WOW i did not pay attention today

## Recursive Algorithm for assembling CRLite list
- Use Bloom filters to iteratively get rid of false positives
### Bloom Filter
- bit array on length of items
- put item into hash function and then set that bit in the array
- There will be some collisions, but they you use all the set bits I guess and continue filtering recursively until there are no more  ?
- Whitelist on each subsequent recursion level

## Certificate Transparency
Goal: make public log of every certificate that has ever been issued. Append-only, uses Merkle Tree

## Questions
- Large scale implementation?
    - CRLite used on Firefox!
    - But only adopted by Mozilla, not Google or Apple or anything :(
- Make the server do these checks for you?
    - Website owner periodically does the request for you, and staples it to the cert during the TLS handshake
# A Longitudinal, End-to-End View of the DNSSEC Ecosystem
_Thu Feb 4_

## The Paper
- DNS is a federated system, where each DNS zone is responsible for their mappings
- With standard DNS, there is no integrity or authentication that the hostname:IP mapping you get is actually correct
- Kaminsky showed it was easy to pollute DNS resolvers
- DNSSEC decided we need to make sure all the responses are signed
- Two keys for DNSSEC, one is zone signing key, and one is key signing key

Say we're `foo.com`, and we have this as our "Resource Record"
```
foo.com     A       1.2.3.4
foo.com     A       1.2.3.5

            RRSIG   SIGNATURE (signed by ZSK)
```
With DNSSEC, we'll have the `RRSIG` type, which is a signature.
We also have records that correspond to the key for this signature
(ZSK and KSK, both of type DNSKEY).
This is all its own resource record:
```
            DNSKEY  ZSK
            DNSKEY  KSK

            RRSIG   SIGNATURE (signed by KSK)
```
Who signs the KSK? The level above you.
In this case, whoever is in charge of `.com` has this resource record:
```
foo.com     DS      KSK
```
Their thing is signed by the root CA.

- The DNSKEY system requires a lot of people to do the right thing, and rely on one another.

## Go Project 1

### Parsing Command-Line Options Example
- Command-line args in go: `os.Args` is `argv`, and `len(os.Args)` is `argc`
- Easier way for more options: `import "flag"` (example on the website)
- Create a struct for all the program's possible options
- Create a `parseOptions` function
- Flag automatically has a help option which will print the options you provided, but you can override it with a custom usage statement
- in `parseOptions`, he returns a pointer to a local variable, which in C would be on the stack, so this would be a bad idea, since it will be removed/junked once the function exits. But, the Go compiler is smart enough to copy that local thing to the heap because it's being returned.

### Simple TCP Client and Server Example
- As opposed to `socket` library/module in C/Python
- `net` library in Go
```go
conn, err := net.Dial("tcp", hostport)
```

 
## Two "Gotchas" in Go
1. Why do we have to call append with `a` as an arg, and also re-assign it to `a`?

```go
var a []int
a = append(a, 100)
```

Slices in Go are backed by an array, with a length equal to the slice's capacity. When you call append, a copy of the slice is actually what's passed in, so you have to reassign it back to the original.

2. What's the difference between these two?

```go
a := 1
a,b := 2,3
```
```go
a := 1
if true {
    a,b := 2,3
}
```

At the end of the first one, a=2, b=3. At the end of the second one, a=1, b=3 because you declared a in a block. The `:=` operator always needs to declare at least one "new" variable, and declarations just affect the local scope. It's recommended to just declare everything at the beginning of a function, for clarity and to avoid problems like this.
# TsuKing: Coordinating DNS Resolvers and Queries into Potent DoS Amplifiers
_Thu Feb 6_

## Discussion of previous paper since we spent so much time on Go
- When this came out RSA keys were still mainstream, but now it's usually elliptic curve keys
- Paper uses Luminati, which could be a source of bias if you don't have evenly distributed Luminati hosts
- Problem with DNSSEC and IPv6, is that they are much more complicated than the things they're extending (DNS, IPv4)
- There is a contingent of people who say DNSSEC is worthless, because even if an attacker cache poisons and sends you to malicious site when you wanted to go to `apple.com`, they won't be able to give you apple's cert, so application level protocols will sort it out

## TsuKing
- Paper is written in a way that makes things too complicated
- Brings in a lot of topics though
- TsuKing is a __reflection attack__ and __amplification attack__:
    - __reflection attack__: change the source address in your packet to the victim's address so that response to your request will go to the victim
    - __amplification attack__: change the above into an amplification attack by making a small request that will yield a huge response. DNS has that feature, in the form of an `ANY` query, which returns all records for a domain.
- The paper is mostly concerned with the `NS` type of DNS request
- There are all sorts of different types of things that could be considered a DNS server, but serve a specific role:
    1. Stub Resolver: something on the host machine, eg. C library functions like `getaddrinfo`. These requests go to, for example, your home router which acts as a forwarder to the ISP's recursive resolver.
    2. Recursive Resolver: has a big cache, tries to serve the request from that, and then tries to contact the authoritative name server. For example, finding google.com. First contact root nameserver that knows `.com`, and then from there contact the one that knows `google.com`, etc. So, it walks the domain hierarchy from the top down, and then NS records get returned on the way back up, in addition to the IP address. If it's already seen `apple.com`, and it's looking for `sales.apple.com`, it can start its recursion with `apple.com` as the root, instead of `.com`.
    3. Ingress Server: Google's `8.8.8.8` isn't just one server that's accepting requests and then walking the DNS tree. It's a system with multiple ingress servers, which check for the cached stuff, and then if it's a miss it gets sent to egress servers.
    4. Egress Server: responsible for querying the authoritative nameservers recureively, and handling the return at each level to make the next recursive query

### DNSRetry attack variant
1. Make your own authoritative DNS nameserver, for say, `attack.com`, and put in some malicious DNS records, of the `NS` type, which indicate who the authoritative nameserver is for a particular domain.
2. Attacker makes request to `a.attack.com`, and it gets routed to our nameserver. But, our nameserver says no I'm not the authoritative nameserver for that, this victim IP is. So the victim just gets bombarded by DNS traffic, which it can't handle. Since it's not getting a response (cause the victim isn't even a DNS server), the DNS system keeps retrying, overloading the victim even more with traffic.

### DNSChain attack variant
1. Same setup of malicious nameserver
2. Measurement study of vulnerable DNS Servers
3. Multiple levls of attack somehow?

### DNSLoop attack variant
- Same as DNSChain, but last level sends them back to first level?
- Victim is the DNS services themselves, since they're spending resources servicing traffic amongst themselves

## Other Discussion
- RD flag: when set to 0, requester expects that server is the authoritative name server, and/or will return an answer. Problem is that sometimes people who aren't authoritative nameservers don't check that someone sent something with that bit set, and go do recursive resolve stuff anyway
- Also negative caching is important
- Adding more levels or more queries lead to diminishing marginal returns, becuase resolvers do things like request merging

---

## TCP vs. TLS in Go
### Client
```go
// TCP
net.Dial

// TLS
config := tls.Config{RootCa = CERTFILE_FOR_ROOT_CA}
tls.Dial
```
### Server
```go
// TCP
net.Listen

// TLS
tls.Listen // and you pass some config with server's certificate file
```
 ---
 With these two examples we acn do everything but mutual authentication. Take a look a the TLS package and google around for that. VERY small change, just set a couple more fields in server's config.
# WireGuard: Next Generation Kernel Network Tunnel
_Tue Feb 11_

- Motivation: Existing IPSec is "academically perfect" but really hard to use/configure
- What's the difference between allowed IP and endpoint in network (I think this is what he asked)
- "ton tap": Something coming in gets rerouted
- SYN attack: force server to fill its memory maintaining state for non-fully-connected clients


# OpenVPN is Open to VPN Fingerprinting
_Thu Feb 13_

## Go

### pointers

```go
type Foo struct {
    ... 
}

func NewFoo() *Foo {
    // Three ways to do this
    
    f := new(Foo)
    return f
    
    f := &Foo{}
    return f

    f := Foo{}
    return &f
}
```

### iota

In C, you can use `#define` or `enum` to create mappings from words/statuses to numbers.

In Go, use iota, but they can only increment by 1 for each label.

```go
const (
    OK = iota + 200
    BAR
    BAZ
    ...
)
```
BAR will be 201, BAZ will be 202, etc.

# Tor: The Second-Generation Onion Router
_Tue Feb 18_

(Tor and two other papers)

- Tor onion routers run at user level with no special privileges
- Entry node will know your IP but not the destination of your traffic
- Middle nodes only know previous and next nodes
- Exit node knows destination but not original sender
- Onion encryption means each relay decrypts only its own layer of encryption: __is this why it's slow or are there other reasons as well?__
- Tor's security is somewhat proportional to the number of nodes
- The fact that someone is using a Tor browser is not hidden, and it could be blockable
- Protocol normalization: Tor will NOT try and strip out anything from http traffic that would allow you to be fingerprinted as a user: a different tool would be needed for something like that
- __What's stopping one node from decrypting all layers?__: The whole circuit gets built incrementally for the connection, so all the public keys will get available to the entry node, and they can onion encrypt the data with all those keys
- Why not just a minimum of two nodes (only entry and exit)? Why three? According to the Tor authors, paranoia!
- Tor assumes a powerful adversary, but one that can't take over the whole network. Some nation states have a huge view of the network, and are able to probably have 'end to end' monitoring, of exit and entry nodes
- Some systems in the related work can actually defeat huge governments by sending out junk traffic in the mixers at time intervals, but this makes it EVEN slower
- Problems with Tor (from this article: https://blog.cloudflare.com/the-trouble-with-tor/)
    - A lot of Exit Nodes have terrible reputation scores, and get sent captchas to solve, and it's pretty annoying for the Tor users to have to solve captchas all the time
    - Cloudflare developed technology to make Tor a better experience (avoid Captcha bombardment): __Privacy Pass__
    - __Privacy Pass__: When you first go to site, you have to solve some crypto puzzle to get some tokens to be able to do later stuff

# How Do Tor Users Interact With Onion Services?

# The Double Ratchet Algorithm
1. Forward Secrecy: if a breakin occurs at time $t$, they can't decrypt things that happened in time $t-1$
2. Something else for the other direction I think?
3. Key Derivation Function (KDF): pass your password to a KDF and it will give you an AES key, for example. You keep chaining your sending and recieving keys so that each message gets encrypted with a different key

If someone gets your key (compromise), how do you provide post-compromise security?
# On Ends-to-Ends Encryption: Asynchronous Group Messaging with Strong Security Guarantees
_Thu Feb 27_

- Relies on trusted group initiator
- Initial leaf keys are result of setup key and each user's pre-keys
- Every new key has relationship with previous private key. When you post a key update message, you need to MAC it with the original group key thing
# End-to-End Measurements of Email Spoofing Attacks
_Tue Mar 4_

- Missed class

# A Longitudinal and Comprehensive Study of the DANE Ecosystem in Email
_Thu Mar 6_

- Section 4 is simple initial study to see how prevelant TLSA records are: piggyback on openintel to get records and info
- Section 5 gets a little more involved and figures out some stuff about misconfiguration within the group that does use DANE.
    - For the most part, if you're attempting to use DANE, they find that a vast majority of people are configuring it fine
    - Similar to the DNSSEC paper it shows that like $17\%$ of them fail to publish DS record to the parent domain tho
- There's a good UChicago video telling people how to write from

## Go Project: sodoh
- Use http package in Go standard library
- Going to be similar to python or Node.js web programming
- Hardest part will be specifying that we trust a certain CA. For this, use `Transport` struct

# A Delay-Tolerant Network Architecture for Challenged  Internets
_Tue Mar 16_
- Assumptions about TCP that don't always hold:
    - Lossless transmission / the ability to always recover
    - Immobile network nodes/clients
    - End-to-end connectivity: always assuming there is a path from src to dest
    - Low latency
- DTN deals with cases where these assumptions don't always hold, such as the ocean, space, combat/hostile zones, etc.
- DTN is an overlay network, over existing network types
- The convergence layer is an adapter to go from the DTN protocol to whatever the underlying network stack is

# Design and Evaluation of IPFS: A Storage Layer for the Decentralized Web
_Thu Mar 20_

- Decentralized web: if one guy goes down you don't lose _everything_
- Paper has two parts: IPFS overview, and then measurement and performance study of the network
- Questions: is IPFS really decentralized in practice, and could IPFS replace the default way of serving web content
- IPFS builds off a DHT: distributed hash table (been around for decades)
    - Ring from $0$ to $2^{256}$
    - IDs for content and nodes are hashed and stored in this ring
    - Looks up through this table halving the distance to the content by walking the DHT tree and asking someone closer to it each time. $O(\log(n))$ is the complexity of the walk
- Provider record: maps object ID/hash to peer ID/hash
- Peer record: maps peer IDs to the network address where you can contact the peer

## sodoh
- client has target's cert, with some RSA pubkey in it. They encrypt AES key with that


# Take Over the Whole Cluster: Attacking Kubernetes via Excessive Permissions of Third-party Applications
_Tue Mar 25_
- Third party kubernetes apps have DaemonSets that run in worker nodes in a cluster
- __if the attacker controls a worker node__, which is a significant threat model assumption, they can potentially steal admin info directly
- Also, they can exploit "critical components" of third party apps running in their own node or other nodes to escalate privileges and cause damage 
- DaemonSet: pods controlled by third-party apps that run on kube nodes (one per node) to do handling and monitoring of events related to whatever the third party app is interested in
- random components: do the actual logic of the third-party app

# Guarding Serverless Applications with Kalium
_Thu Mar 27_
- Defense against return-oriented-programming: Generate all the valid control flow graphs and make sure that control flow that happens is in that set of possible things
- That's basically what Kalium is. 
- Stable identifier for function is URL


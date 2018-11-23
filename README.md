# webminerpool 

**Complete sources** for a Monero (cryptonight/cryptonight-lite) webminer. **Hard fork ready**.


###
_The server_ is written in **C#**, **optionally calling C**-routines to check hashes calculated by the clients. It acts as a proxy server for common pools.


_The client_ runs in the browser using javascript and webassembly. 
**websockets** are used for the connection between the client and the server, **webassembly** to perform hash calculations, **web workers** for threads.

Thanks to [nierdz](https://github.com/notgiven688/webminerpool/pull/62) there is a **docker** file available. See below.

# Will the hardfork (October 2018) be supported?

Yes. Update to the current master branch and you are ready for the October 2018 hard fork.

# What is new?

- **September 27, 2018** 
	- Added cryptonight v2. Hard fork ready! (**client-side** / **server-side**).

- **June 15, 2018** 
	- Support for blocks with more than 2^8 transactions. (**client-side** / **server-side**).

- **May 21, 2018** 
	- Support for multiple open tabs. Only one tab is constantly mining if several tabs/browser windows are open. (**client-side**).

- **May 6, 2018** 
	- Check if webasm is available. Please update the script. (**client-side**).

- **May 5, 2018** 
	- Support for multiple websocket servers in the client script (load-distribution).

- **April 26, 2018** 
	- A further improvement to fully support the [extended stratum protocol](https://github.com/xmrig/xmrig-proxy/blob/dev/doc/STRATUM_EXT.md#mining-algorithm-negotiation)  (**server-side**).
	- A simple json config-file holding all available pools (**server-side**).

- **April 22, 2018** 
	- All cryptonight and cryptonight-light based coins are supported in a single miner. [Stratum extension](https://github.com/xmrig/xmrig-proxy/blob/dev/doc/STRATUM_EXT.md#mining-algorithm-negotiation) were implemented: The server now takes pool suggestions (algorithm and variant) into account. Defaults can be specified for each pool - that makes it possible to mine coins like Stellite, Turtlecoin,.. (**client/server-side**)
	- Client reconnect time gets larger with failed attempts. (**client-side**)

# Repository Content

### SDK

The SDK directory contains all client side mining scripts which allow mining in the browser.

#### Minimal working example

```html
<script src="webmr.js"></script>

<script>
	server = "ws://localhost:8181"
	startMining("minexmr.com","472uwSQRraXF6ELji3g3ToTTcjsG7HbPF8mLFeaN1Wr2eMxkDN3UteyHGARWEmMvd3fnKMb59cyfzWJSoAFWga6GBsrag2n"); 
</script>
```
webmr.js can be found under SDK/miner_compressed.

The startMining function can take additional arguments

```javascript
startMining(pool, address, password, numThreads, userid);
```

- pool, this has to be a pool registered at the server.
- address, a valid XMR address you want to mine to.
- password, password for your pool. Often not needed.
- numThreads, the number of threads the miner uses. Use "-1" for auto-config.
- userid, allows you to identify the number of hashes calculated by a user. Can be any string with a length < 200 characters.

To **throttle** the miner just use the global variable "throttleMiner", e.g. 

```javascript
startMining(..);
throttleMiner = 20;
```

If you set this value to 20, the cpu workload will be approx. 80% (for 1 thread / CPU). Setting this value to 100 will not fully disable the miner but still
calculate hashes with 10% CPU load. 

If you do not want to show the user your address or even the password you have to create  a *loginid*. With the *loginid* you can start mining with

```javascript
startMiningWithId(loginid)
```

or with optional input parameters:

```javascript
startMiningWithId(loginid, numThreads, userid)
```

Get a *loginid* by opening *register.html* in SDK/other. You also find a script which enumerates all available pools and a script which shows you the amount of hashes calculated by a *userid*. These files are quite self-explanatory.

#### What are all the *.js files?

SDK/miner_compressed/webmr.js simply combines 

 1. SDK/miner_raw/miner.js
 2. SDK/miner_raw/worker.js
 3. SDK/miner_raw/cn.js

Where *miner.js* handles the server-client connection, *worker.js* are web workers calculating cryptonight hashes using *cn.js* - a emscripten generated wrapped webassembly file. The webassembly file can also be compiled by you, see section hash_cn below.

### Server

The C# server. It acts as proxy between the clients (the browser miners) and the pool server. Whenever several clients use the same credentials (pool, address and password) they get "bundled" into a single pool connection, i.e. only a single connection is seen by the pool server. This measure helps to prevent overloading regular pool servers with many low-hash web miners.

The server uses asynchronous websockets provided by the
[FLECK](https://github.com/statianzo/Fleck) library. Smaller fixes were applied to keep memory usage low. The server code should be able to handle several thousand connections with modest resource usage.

The following compilation instructions apply for linux systems. Windows users have to use Visual Studio to compile the sources.

 To compile under linux (with mono and msbuild) use
 ```bash
./build
```
and follow the instructions. No additional libraries are needed.

```bash
mono server.exe
```

should run the server.

 Optionally you can compile the C-library **libhash**.so found in *hash_cn*. Place this library in the same folder as *server.exe*. If this library is present the server will make use of it and check hashes which gets submitted by the clients. If clients submit bad hashes ("low diff shares"), they get disconnected. The server occasionally writes ip-addresses to *ip_list*. These addresses should get (temporarily) banned on your server for example by adding them to [*iptables*](http://ipset.netfilter.org/iptables.man.html). The file can be deleted after the ban. See *Firewall.cs* for rules when a client is seen as malicious - submitting wrong hashes is one possibility.

 Without a **SSL certificate** the server will open a regular websocket (ws://0.0.0.0:8181). To use websocket secure (ws**s**://0.0.0.0:8181) you should place *certificate.pfx* (a  pkcs12 file) into the server directory. The default password which the server uses to load the certificate is "miner". To create a pkcs12 file from regular certificates, e.g. from [*Let's Encrypt*](https://letsencrypt.org/), use the command

```bash
openssl pkcs12 -export -out certificate.pfx -inkey privkey.pem -in cert.pem -certfile chain.pem
```

The server should autodetect the certificate on startup and create a secure websocket.

**Attention:** Most linux based systems have a (low) fixed limit of
available file-descriptors configured ("ulimit"). This can cause an
unwanted upper limit for the users who can connect (typical 1000). You
should change this limit if you want to have more connections.

### hash_cn

The cryptonight hashing functions in C-code. With simple Makefiles (use the "make" command to compile) for use with gcc and emcc - the [emscripten](https://github.com/kripken/emscripten) webassembly compiler. *libhash* should be compiled so that the server can check hashes calculated by the user.

# Dockerization

Find the original pull request with instructions by nierdz [here](https://github.com/notgiven688/webminerpool/pull/62).

Added Dockerfile and entrypoint.sh.
Inside entrypoint.sh, a certificate is installed so you need to provide a domain name during docker run. The certificate is automatically renewed using a cronjob.

```bash
cd webminerpool
docker build -t webminerpool .
```

To run it: 

```bash
docker run -d -p 80:80 -p 8181:8181 -e DOMAIN=mydomain.com webminerpool
```
You absolutely need to set a domain name.
The 80:80 bind is used to obtain a certificate.
The 8181:8181 bind is used for server itself.

If you want to bind these ports to a specific IP, you can do this:

```bash
docker run -d -p xx.xx.xx.xx:80:80 -p xx.xx.xx.xx:8181:8181 -e DOMAIN=mydomain.com webminerpool
```

Donations
---------
* ETN: `etnjxu3nh241XPEJMrKPVT8FJV37HvUGWVrnxn76sUoF77mV1yDCS9PVzC5jKHDmrFRXpv5RSNmPgJdF8kDMsjwG2sG4cKYQNK`
* XMR: `472uwSQRraXF6ELji3g3ToTTcjsG7HbPF8mLFeaN1Wr2eMxkDN3UteyHGARWEmMvd3fnKMb59cyfzWJSoAFWga6GBsrag2n`
* AEON: `Wmswvbsac7eZ7pZEbey9nmgoZPjtwBAwRh7Qgm1xHVwF6hcnH43r2vX3hLTKARSrvtH8g4wJtEXS9V3Axz1Y2m8P2uqXEZi51`
* Karbo: `KbEVYBQuApoMvRLpKGymFvA9wbdVDu9rtg69a5rBqKoAaagYhcaBYupf7bsDLyMQakD7M9jp23krPMYKQUy4GoqK7Q3njZ2`


```
███████╗███╗   ██╗██╗   ██╗ ██████╗ ██╗   ██╗
██╔════╝████╗  ██║██║   ██║██╔═══██╗╚██╗ ██╔╝
█████╗  ██╔██╗ ██║██║   ██║██║   ██║ ╚████╔╝
██╔══╝  ██║╚██╗██║╚██╗ ██╔╝██║   ██║  ╚██╔╝
███████╗██║ ╚████║ ╚████╔╝ ╚██████╔╝   ██║
╚══════╝╚═╝  ╚═══╝  ╚═══╝   ╚═════╝    ╚═╝

██████╗ ███████╗███████╗██████╗     ██████╗ ██╗██╗   ██╗███████╗
██╔══██╗██╔════╝██╔════╝██╔══██╗    ██╔══██╗██║██║   ██║██╔════╝
██║  ██║█████╗  █████╗  ██████╔╝    ██║  ██║██║██║   ██║█████╗
██║  ██║██╔══╝  ██╔══╝  ██╔═══╝     ██║  ██║██║╚██╗ ██╔╝██╔══╝
██████╔╝███████╗███████╗██║         ██████╔╝██║ ╚████╔╝ ███████╗
╚═════╝ ╚══════╝╚══════╝╚═╝         ╚═════╝ ╚═╝  ╚═══╝  ╚══════╝
```

# Envoy deep dive

WIP

This is part of my journey learning Envoy, I try to explain different areas of envoy by agregating/simplifying/extending existing information found in several different sources (all the sources are referenced in the "Sources" section). Treat it as my "personal" study notes :)

Please, send any comment to:

* email: jmprusi at keepalive dot io  / joaquim at redhat dot com
* twitter: https://twitter.com/jmprusi

## Index (WIP not up to date.)

1. What is envoy.
2. Architecture overview.
3. Building envoy.
4. Configuration.
5. Rate limiting.
6. Creating our first Control Plane in Go.
7. Writting a Lua Filter.
8. Writting a C++ Filter.
9. Sources

### 1. What is envoy

As defined by [What Is Envoy](https://www.envoyproxy.io/docs/envoy/latest/intro/what_is_envoy):

"**Envoy** is an **L7 proxy** and **communication bus** designed for large **modern service oriented architecture**"

One of the main objectives is to make the network transparent to our applications, that's why envoy is designed around the idea of being deployed together with our apps in a transparent way, by embracing the "**Out of process architecture**", where all the envoy proxys deployed form a communication mesh where applications can contact each other.

Envoy provides the following high level features:

* **L3/L4 filter architecture:** At its core, Envoy is an **L3/L4 network proxy**. A pluggable **filter chain mechanism** allows filters to be written to perform different **TCP proxy** tasks and inserted into the main server. Filters have already been written to support various tasks such as **raw TCP proxy**, **HTTP proxy**, **TLS client certificate authentication**, etc.

* **Front/edge proxy support:** Envoy includes enough features to make it usable as an **edge proxy** for most modern web application use case. This includes TLS termination, HTTP/1.1 and HTTP/2 support, as well as HTTP L7 routing.

* **First class HTTP/2 support:** When operating in HTTP mode, Envoy supports both HTTP/1.1 and HTTP/2. Envoy can operate as a transparent HTTP/1.1 to HTTP/2 proxy in both directions. This means that any combination of HTTP/1.1 and HTTP/2 clients and target servers can be bridged.

* **HTTP L7 filter architecture:** Envoy supports an additional HTTP L7 filter layer. HTTP filters can be plugged into the HTTP connection management subsystem that perform different tasks such as **buffering**, **rate limiting**, **routing/forwarding**,**Sniffing Amazon’s DynamoDB**, etc.

* **HTTP L7 routing:** When operating in HTTP mode, Envoy supports a routing subsystem that is capable of routing and redirecting requests based on path, authority, content type, runtime values, etc.

* **gRPC support:** gRPC is an RPC framework from Google that uses HTTP/2 as the underlying multiplexed transport. Envoy supports all of the HTTP/2 features required to be used as the routing and load balancing substrate for gRPC requests and responses.

* **Service discovery and dynamic configuration:** Envoy optionally consumes a layered set of dynamic configuration APIs for centralized management. The layers provide an Envoy with dynamic updates about: hosts within a backend cluster, the backend clusters themselves, HTTP routing, listening sockets, and cryptographic material. For a simpler deployment, backend host discovery can be done through DNS resolution (or even skipped entirely), with the further layers replaced by static config files.

* **Health checking:** Envoy includes a health checking subsystem which can optionally perform **active health checking** of upstream service clusters and also **passive health checking** via an outlier detection subsystem.

* **Advanced load balancing:** Currently Envoy includes support for **automatic retries**, **circuit breaking**, **global rate limiting** via an external rate limiting service, **request shadowing**, and **outlier detection**.

* **Observability:** Envoy includes **robust statistics** support for all subsystems. **statsd** (and compatible providers) is the currently supported statistics sink. Statistics are also viewable via the administration port. Envoy also **supports distributed tracing** via thirdparty providers.

(I tried to resume a little bit the origicanl documentation, if you want to read the full, unaltered version in the oficial documentation page of envoy, go there: [What Is Envoy](https://www.envoyproxy.io/docs/envoy/latest/intro/what_is_envoy))

### 2. Architecture overview

### 2.1 Threading Model

Envoy uses different types of threads:


```
┌──────────────────────┐  ┌───────────────────────┐   ┌───────────────────┐
│┌───────────────────┐ │  │┌───────────────────┐  │   │┌──────────────────┴┐   
││                   │ │  ││┌──────────────────┴┐ │   ││┌──────────────────┴┐
││                   │ │  │││┌──────────────────┴┐│   │││                   │
││    Main Thread    │ │  ││││                   ││   │││    File Flush     │
││                   │ │  ││││      Worker       ││   │││     Thread(s)     │
││                   │ │  ││││     thread(s)     ││   └┤│                   │
│└───────────────────┘ │  │└┤│                   ││    └┤                   │
│                      │  │ └┤                   ││     └───────────────────┘
│ ┌──────────────────┐ │  │  └───────────────────┘│
│ │                  │ │  │                       │
│ │       xDS        │ │  │  ┌──────────────────┐ │
│ │                  │ │  │  │                  │ │
│ └──────────────────┘ │  │  │    Listeners     │ │
│ ┌──────────────────┐ │  │  │                  │ │
│ │                  │ │  │  └──────────────────┘ │
│ │     Runtime      │ │  │  ┌──────────────────┐ │
│ │                  │ │  │  │                  │ │
│ └──────────────────┘ │  │  │   Connections    │ │
│ ┌──────────────────┐ │  │  │                  │ │
│ │                  │ │  │  └──────────────────┘ │
│ │    Stat flush    │ │  └───────────────────────┘
│ │                  │ │
│ └──────────────────┘ │
│ ┌──────────────────┐ │
│ │                  │ │
│ │      Admin       │ │
│ │                  │ │
│ └──────────────────┘ │
│ ┌──────────────────┐ │
│ │     Process      │ │
│ │    Management    │ │
│ │                  │ │
│ └──────────────────┘ │
└──────────────────────┘
```


* **Main:** This thread owns server startup and shutdown, **all xDS API handling** (including DNS, health checking, and general cluster management), **runtime**, **stat flushing**, **admin**, and **general process management** (signals, hot restart, etc.). Everything that happens on this thread is **asynchronous** and **“non-blocking.”**

* **Worker:** By default, Envoy spawns a worker thread for every hardware thread in the system. Each worker thread runs a **“non-blocking” event loop** that is responsible for** listening on every listener**, **accepting new connections**, **instantiating a filter stack for the connection**, and **processing all IO for the lifetime of the connection**.

* **File flusher:** Every file that Envoy writes (primarily access logs) currently has an independent blocking flush thread. When worker threads need to write to a file, the data is actually moved into an in-memory buffer, where it is eventually flushed via the File flush thread

Because Envoy separates main thread responsabilities from worker thread responsabilities, theres is a requirement that **complex processing is done at the main thread**, for example, timestamp for http headers. To push this calculated data from the main thread to the worker processes in a highly concurrent way, Envoy uses the **Thread Local Storage (TLS) System**:


```                                                                                            
                                                                                                
                                         Dispatcher Post()                                      
                             ┌────────────────────────────────────────┐                         
                             │                                        │                         
                             │                                        ▼                         
        ┌─────┬─────┬─────┬─────┬─────┬─────┐    ┌─────┬─────┬─────┬─────┬─────┬─────┐          
        │     │     │     │     │     │     │    │     │     │     │     │     │     │          
  Main  │  0  │  1  │  2  │  3  │  4  │  5  │    │  0  │  1  │  2  │  3  │  4  │  5  │ Worker 1 
        │     │     │     │     │     │     │    │     │     │     │     │     │     │          
        └─────┴─────┴─────┴─────┴─────┴─────┘    └─────┴─────┴─────┴─────┴─────┴─────┘          
         "Slots"             │                   ┌─────┬─────┬─────┬─────┬─────┬─────┐          
                             │                   │     │     │     │     │     │     │          
                             │                   │  0  │  1  │  2  │  3  │  4  │  5  │ Worker 2 
                             │                   │     │     │     │     │     │     │          
                             │                   └─────┴─────┴─────┴─────┴─────┴─────┘          
                             │                   ┌─────┬─────┬─────┬─────┬─────┬─────┐          
                             │                   │     │     │     │     │     │     │          
                             │                   │  0  │  1  │  2  │  3  │  4  │  5  │ Worker 3 
                             │                   │     │     │     │     │     │     │          
                             │                   └─────┴─────┴─────┴─────┴─────┴─────┘          
                             │                   ┌─────┬─────┬─────┬─────┬─────┬─────┐          
          ┌────────────────┐ │                   │     │     │     │     │     │     │          
          │                │ │                   │  0  │  1  │  2  │  3  │  4  │  5  │ Worker 4 
          │   TLS get()    │ │                   │     │     │     │     │     │     │          
          │                │ │                   └─────┴─────┴─────┴─────┴─────┴─────┘          
          └────────────────┘ │                                  │     ▲                         
                   ▲         │                                  │     │                         
                   │         └──────────────────────────────────┼─────┘                         
                   └────────────────────────────────────────────┘                               

```

* **Code running on the main thread can allocate a process-wide TLS slot**. Though abstracted, in practice, this is an index into a vector allowing O(1) access.

* **The main thread can set arbitrary data into its slot**. When this is done, the data is posted into each worker as a normal event loop event.

* **Worker threads can read from their TLS slot and will retrieve whatever thread local data is available there**.

This is an incredibly **powerful paradigm that is very similar to the [RCU](https://en.wikipedia.org/wiki/Read-copy-update) locking concept**. Worker threads never see any change to the data in the TLS slots while they are doing work. **Change only happens during the quiescent period between work events**. 

Althought all of the code is written assuming that nothing ever blocks. However, **this is not entirely true** as **Envoy does employ a few process wide locks that can have some effect in the overall performance**:

* If access logs are being written, all workers acquire the same lock before filling the in-memory access log buffer. Lock hold time should be very low, but it is possible for this lock to become contended at high concurrency and high throughput.

* Part of the **thread local stat handling** sometimes requires to acquire a lock to the central “stat store.” This lock should not ever be highly contended.

* The main thread periodically needs to coordinate with all worker threads. This is done by “posting” from the main thread to the worker threads. Posting requires taking a lock so the posted message can be put into a queue for later delivery. These locks should never be highly contended but they can still technically block.

* When Envoy logs itself to standard error, it acquires a process-wide lock. **Envoy local logging is assumed to be terrible for performance.**


### 2.2 Connection Handling

All worker threads listen on all listeners without any sharding. The kernel is used to intelligently dispatch accepted sockets to worker threads.

Once a connection is accepted on a worker, **it never leaves that worker**. All further handling of the connection is entirely processed within the worker thread, including any forwarding behaviour.

* **All connection pools in Envoy are per worker thread**. So, though HTTP/2 connection pools only make a single connection to each upstream host at a time, **if there are four workers, there will be four HTTP/2 connections per upstream host at steady state**.

* From a memory and connection pool efficiency standpoint, it is actually quite **important to tune the `--concurrency` option**. **Having more workers than is needed will waste memory, create more idle connections, and lead to a lower connection pool hit rate**.

### 2.3 Performance sensible areas

There are a few known areas that will need attention when it is used at very high concurrency and throughput:

* Currently all workers acquire a lock **when writing to an access log’s in-memory buffer**. At high concurrency and high throughput it will be required to do per-worker batching of access logs at the cost of out of order delivery when written to the final file.

* Although stats are very heavily optimized, at very high concurrency and throughput it is likely that there will be **atomic contention on individual stats**. The solution to this is per-worker counters with periodic Lushing to the central counters.

* The existing architecture will not work well if Envoy is deployed in a scenario in which there are very **few connections that require substantial resources to handle**. This is because there is no guarantee that connections will be evenly spread between workers. This can be solved by implementing worker connection balancing in which a worker is capable of forwarding a connection to another worker for handling.

### 2.4 xDS APIs

## 3. Building Envoy

### 3.1 Get the sources

### 3.2 Building it locally

### 3.3 Building it using Docker


## 4.Configuration

### 4.X Listeners

A listener is a named network location (currently it can be either a port or a unix domain socket), that can accept connections from downstream clients, **currently only TCP protocol**.

Listeners **can be declared statically or dynamically via the listener discovery service (LDS)**, and **one envoy instance can have several listeners**, each independently configured with a varying number of L3/L4 layer level filters that get instantiated once the listener receives a new connection.

Listeners can also be configured with **"Listener Filters"**, those filters are processed **before** the L3/L4 layer Filters.

Example Listener definition: 

```yaml
listeners:
  - address:                    
      socket_address:
        address: 0.0.0.0
        port_value: 80
    filter_chains:                                 
    - filters:                                      
      - name: envoy.http_connection_manager
        config:
          codec_type: auto
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
              - "*"
              routes:
              - match:
                  prefix: "/service/1"
                route:
                  cluster: service1
              - match:
                  prefix: "/service/2"
                route:
                  cluster: service2
          http_filters:
          - name: envoy.router
            config: {}
```


### 4.X Clusters

A Cluster defines **a set of upstreams hosts where Envoy will connect to**, can **discover members of the cluster in a dynamic way via service discovery or statically provided in the config file**. **Optionally determines the health of cluster members via active health checking.**

Example Cluster definition:

```yaml
clusters:
  - name: ipapi_co
    connect_timeout: 0.25s
    type: logical_dns
    dns_lookup_family: V4_ONLY
    lb_policy: round_robin
    tls_context: { sni: ipapi.co }
    upstream_connection_options:
      tcp_keepalive: {}
    load_assignment:
      cluster_name: ipapi_co
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: ipapi.co
                port_value: 443
```

### 4.X Admin


Example Admin section: 

```yaml
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 8001
```


### 4.X Bootstrap Configuration

### 4.X Static Configuration

### 4.X Dynamic Configuration

Envoy can use a set of APIs, using the xDS Protocol, for discovering different configuration resources: 

* **LDS**: Listener Discovery Service
* **RDS**: Route Discovery Service
* **VHDS**: Virtual Host Discovery Service
* **CDS**: Cluster Discovery Service
* **EDS**: Endpoint Discovery Service
* **SDS**: Secret Discovery Service

The xDS Protocol supports gRPC or REST. 

Additionally to reduce the number of streams and improve fault tolerance, Envoy provides two different endpoints that
can be used to fetch all the information at the same time:

* **Aggregated Discovery Services (ADS)**: 

ADS allow a single management server, via a single gRPC stream, to deliver all API updates. This provides the ability to carefully sequence updates to avoid traffic drop. With ADS, a single stream is used.

Basically this endpoints aggregates all the configuration, and on each change, recalculates the whole configuration file.

* **Incremental Discovery Services (Incremental xDS)**: 

Allows the protocol to communicate on the wire in terms of resource/resource name deltas ("Delta xDS"). This supports the goal of scalability of xDS resources. Rather than deliver all 100k clusters when a single cluster is modified, the management server only needs to deliver the single cluster that changed.
Allows the Envoy to on-demand / lazily request additional resources. For example, requesting a cluster only when a request for that cluster arrives.


## 5. Filters

### 5.X Listener Filters

### 5.X L3/L4 Filters

## 5.X.X Rate limiting

### 5.X.X HTTP Filters

### 5.X.X The HTTP Connection Manager

### 5.X.X The HTTP Lua Filter

This HTTP Filter allows us to create logic by using LUA

### 5.X.X Lua Filter Support Overview

The current LUA support uses LuaJIT as the runtime, code should be loaded inline the configuration (although loading from filesystem can be implemented by the user easily).

* **All Lua environments are per worker thread** that implies that **there is no truly global data.**.
* **All scripts are run as coroutine:** All network/async processing is performed by Envoy via a set of APIs. Envoy will yield the script as appropriate and resume it when async tasks are complete.
* **Do not perform blocking operations from scripts**: It is critical for performance that Envoy APIs are used for all IO.

### 5.X.X What we can do with Lua Filters.

The LUA http Filter is meant to be used for **simple and safe scripts**, **anything that's complex or requires high performance should be written using the native C++ filter API**.

* **Inspection of headers, body, and trailers** while streaming in either the **request flow, response flow**, or both.
* **Modification of headers and trailers.**
* **Blocking and buffering the full request/response body for inspection.**
* **Performing an outbound async HTTP call to an upstream host.** Such a call can be performed while buffering body data so that when the call completes upstream headers can be modified.
* **Performing a direct response and skipping further filter iteration.** For example: a script could make an upstream HTTP call for authentication, and then directly respond with a 403 response code.

To do so, we should use the **"Stream Handle API"**:

```lua
function envoy_on_request(request_handle)
end

function envoy_on_response(response_handle)
end
```

Each Lua script should implement at least one of the previous global functions. Envoy will run the `envoy_on_request` function as a corouting during the `request` phase, and the `envoy_on_response` during the response phase.

More info about this API: [Stream Handle API docs](https://www.envoyproxy.io/docs/envoy/latest/configuration/http_filters/lua_filter#stream-handle-api)

### 5.X.X Available LUA APIs

* **Header object API:** Allows us to `get()`, `add()`, `remove()`, `replace()` current headers, or you can iterate throught the key, value pair with `__pairs()`.

* **Buffer API:** Allows us to interact with the current Buffer, provides the `length()` method to get the size of the buffer, and `getBytes()` to get bytes from the buffer.

* **Metadata object API:** Gives us access to the metada objects via `get()` and `__pairs()` to iterate through every metadata entry. 

* **Stream info object API:** Provides a way to get the `protocol()` of the current request (HTTP/1.0, HTTP/1.1 and HTTP/2) and getting all the `dynamicMetadata()` with populated info from the current connection.

* **Dynamic metadata object API:** Allows us to interact with the dynamic metada object, with `get()`, `set()`, and `__pairs()`, like a key/value store between filters.

* **Connection object API:** Gives us the `ssl()` method to know if a connection is ssl or plain.

### 5.X.X Setting up a testing environment

There's a ready to use example in the envoy repo, so first, pull the envoy repository from github:

```bash
git clone https://github.com/envoyproxy/envoy
cd envoy/examples/lua
```

Pull, build and run the images with docker-compose:

```bash
docker-compose pull
docker-compose up --build
```

Test the environment connectivity:

```bash
curl -v localhost:8000
```

The curl should return something like this: 

```
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8000 (#0)
> GET / HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 200 OK
< x-powered-by: Express
< content-type: application/json; charset=utf-8
< content-length: 544
< etag: W/"220-jKSHp64FKaJ/Uq1aR8BfGXuzPJM"
< date: Tue, 23 Apr 2019 14:00:00 GMT
< x-envoy-upstream-service-time: 2
< response-body-size: 544 <----------- LUA response phase code
< server: envoy
<
{
  "path": "/",
  "headers": {
    "host": "localhost:8000",
    "user-agent": "curl/7.64.1",
    "accept": "*/*",
    "x-forwarded-proto": "http",
    "x-request-id": "3427f10f-4783-4b0f-bcb9-b70e2d169acb",
    "foo": "bar",   <----------- LUA request phase code
    "x-envoy-expected-rq-timeout-ms": "15000",
    "content-length": "0"
  },
  "method": "GET",
  "body": "",
  "fresh": false,
  "hostname": "localhost",
  "ip": "::ffff:172.18.0.3",
  "ips": [],
  "protocol": "http",
  "query": {},
  "subdomains": [],
  "xhr": false,
  "os": {
    "hostname": "f429de3b2aff"
  }
* Connection #0 to host localhost left intact
}* Closing connection 0
```

Let's see how it works.

### 5.X.X Reviewing the example Lua filter

If you check the file `examples/lua/envoy.yaml` [(Github Link)](https://github.com/envoyproxy/envoy/blob/master/examples/lua/envoy.yaml), focusing only on the `http_filters` part: 

```yaml
(..)
  http_filters:
  - name: envoy.lua
    typed_config:
      "@type": type.googleapis.com/envoy.config.filter.http.lua.v2.Lua
      inline_code: |
        function envoy_on_request(request_handle)
          request_handle:headers():add("foo", "bar")
        end
        function envoy_on_response(response_handle)
          body_size = response_handle:body():length()
          response_handle:headers():add("response-body-size", tostring(body_size))
        end
  - name: envoy.router
    typed_config: {}
(..)
```

As you see the lua code is inline:

```lua
function envoy_on_request(request_handle)
  request_handle:headers():add("foo", "bar")
end
function envoy_on_response(response_handle)
  body_size = response_handle:body():length()
  response_handle:headers():add("response-body-size", tostring(body_size))
end
```

Here we can see that the filter is acting on both, request, and response:

* **Request phase**

```lua
function envoy_on_request(request_handle)
  request_handle:headers():add("foo", "bar")
end
```

During this phase, the example makes use of the headers api, **to add a new header called `foo` with value `bar`** 

* **Response phase**

```lua
function envoy_on_response(response_handle)
 body_size = response_handle:body():length()
 response_handle:headers():add("response-body-size", tostring(body_size))
end
```

In this phase, the example lua code **extract the actual size of the body, and adds it to the response header as `"response-body-size"`**, converting the value to a string. 

### 5.X.X Writting a more "advanced" Lua filter

Let's write a example lua filter that adds the continent code of the server as a header to the request and response, using `ipapi.co` services.

We need to: 

1. Contact the ipapi.co servers, so we should add a new cluster to the configuration.
2. Make an http call from lua to the server using: `https://ipapi.co/continent_code`. This will return the country code, for example: `EU` and we should add it as a header to the request/response.

Let's do it: 

1. Adding a new cluster so we can contact those servers: 

```yaml
 clusters:
  - name: web_service
          
          (..)

  - name: ipapi_co
    connect_timeout: 0.25s
    type: logical_dns
    dns_lookup_family: V4_ONLY
    lb_policy: round_robin
    tls_context: { sni: ipapi.co }
    upstream_connection_options:
      tcp_keepalive: {}
    load_assignment:
      cluster_name: ipapi_co
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: ipapi.co
                port_value: 443
```

2. Adding the code to make an http call from lua, reusing the previous lua file:


```lua
function envoy_on_request(request_handle)
  local headers, body = request_handle:httpCall("ipapi_co",{
  [":method"] = "GET",
  [":path"] = "/continent_code",
  [":authority"] = "ipapi.co",
  },"",5000)
  request_handle:headers():add("continent_code", body)
end
function envoy_on_response(response_handle)
  local headers, body = response_handle:httpCall("ipapi_co",{
  [":method"] = "GET",
  [":path"] = "/continent_code",
  [":authority"] = "ipapi.co",
  },"",5000)
  response_handle:headers():add("continent_code", body)
end
```

While this will work, we are doing 2 requests each time... By using the `DynamicMetada()` api, we can improve that: 

```lua
function envoy_on_request(request_handle)
  local headers, body = request_handle:httpCall("ipapi_co",{
  [":method"] = "GET",
  [":path"] = "/continent_code",
  [":authority"] = "ipapi.co",
  },"",5000)
  
  request_handle:streamInfo():dynamicMetadata():set("continentCode","continentCode", body)
  request_handle:headers():add("continent-code", body)
end

function envoy_on_response(response_handle)
  continent_code = response_handle:streamInfo():dynamicMetadata():get("continentCode")["continentCode"]
  response_handle:headers():add("continent-code", continent_code)
end
```

As you can see, we store the result of the first request in the `continentCode` key of the `dynamicMetada`, then we retrieve it on the response and reuse it.

Let's test it! 

```
~ curl -v localhost:8000
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8000 (#0)
> GET / HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 200 OK
< x-powered-by: Express
< content-type: application/json; charset=utf-8
< content-length: 554
< etag: W/"22a-LFeWARb/eiWVVHrEiNelcHiIAwo"
< date: Tue, 23 Apr 2019 16:16:54 GMT
< x-envoy-upstream-service-time: 2
< continent-code: EU  <------------ Our continent-code header in the response
< server: envoy
<
{
  "path": "/",
  "headers": {
    "host": "localhost:8000",
    "user-agent": "curl/7.64.1",
    "accept": "*/*",
    "x-forwarded-proto": "http",
    "x-request-id": "c71d92a9-0028-4cd5-8221-08bd0ea896d5",
    "continent-code": "EU",  <------------ Our continent-code header in the request
    "x-envoy-expected-rq-timeout-ms": "15000",
    "content-length": "0"
  },
  "method": "GET",
  "body": "",
  "fresh": false,
  "hostname": "localhost",
  "ip": "::ffff:172.18.0.3",
  "ips": [],
  "protocol": "http",
  "query": {},
  "subdomains": [],
  "xhr": false,
  "os": {
    "hostname": "f429de3b2aff"
  }
* Connection #0 to host localhost left intact
}* Closing connection 0
```

Of course this needs better error handling... :) 


Also, with a Lua Filter we can respond back to the user with an specific HTTP status code and stop the processing of the remaining filters, this is done with the `respond()` method of the request_handle: 


```lua
function envoy_on_request(request_handle)
  local headers, body = request_handle:httpCall("ipapi_co",{
  [":method"] = "GET",
  [":path"] = "/continent_code",
  [":authority"] = "ipapi.co",
  },"",5000)
  
  
  if body == "EU" then
    request_handle:respond({[":status"] = "403",},"nope")
  end
  
  request_handle:streamInfo():dynamicMetadata():set("continentCode","continentCode", body)
  request_handle:headers():add("continent-code", body)
end

function envoy_on_response(response_handle)
  continent_code = response_handle:streamInfo():dynamicMetadata():get("continentCode")["continentCode"]
  response_handle:headers():add("continent-code", continent_code)
end
```

Now if you run the test request again: 

```
~ curl -v localhost:8000
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 8000 (#0)
> GET / HTTP/1.1
> Host: localhost:8000
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 403 Forbidden
< content-length: 4
< date: Wed, 24 Apr 2019 09:55:42 GMT
< server: envoy
<
* Connection #0 to host localhost left intact
nope* Closing connection 0
```

And voila! got a 403 forbidden. 

## 8. Writting a C++ Filter


### 8.1 Writting a L3/L4 Filter in C++

Explain and extend https://github.com/envoyproxy/envoy-filter-example


### 8.2 Writting a HTTP Filter in C++

Explain and extend https://github.com/envoyproxy/envoy-filter-example/tree/master/http-filter-example

## 6. Creating our first Control Plane in Go




## Sources

* xDS Protocol: https://github.com/envoyproxy/data-plane-api/blob/master/XDS_PROTOCOL.md
* Envoy Threading Model: https://blog.envoyproxy.io/envoy-threading-model-a8d44b922310
* Slides from Envoy Internals Deep Dive: https://static.sched.com/hosted_files/kccnceu18/75/Kubecon_EU_18_Draft.pdf
* Threading Model: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/threading_model
* Stream Handle API: https://www.envoyproxy.io/docs/envoy/latest/configuration/http_filters/lua_filter#stream-handle-api
* Lua example environment: https://github.com/envoyproxy/envoy/tree/master/examples/lua
* Green Memory Study notes blog post: https://blog.gmem.cc/envoy-study-note










































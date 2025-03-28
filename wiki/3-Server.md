# Getting started

```clj
(ns my-ns
  (:require [org.httpkit.server :as hk-server]))
```

http-kit server uses an **event-driven, non-blocking I/O model** that makes it lightweight and scalable.

It's written to conform to the standard Clojure web server [Ring spec](https://github.com/ring-clojure/ring), with asynchronous and WebSocket support. http-kit is an (almost) drop-in replacement for [ring-jetty-adapter](https://clojars.org/ring/ring-jetty-adapter).

## Start server

`run-server` starts a Ring-compatible HTTP server.

```clj
(defn app [req]
  {:status  200
   :headers {"Content-Type" "text/html"}
   :body    "hello HTTP!"})

(def my-server (hk-server/run-server app {:port 8080})) ; Start server
```

See [API docs](http://http-kit.github.io/http-kit/org.httpkit.server.html#var-run-server) for detailed info on server options

## Stop server

The `run-server` call above returns a stop function that you can call like so:

```clj
(my-server) ; Immediate shutdown (breaks existing reqs)

;; or

;; Graceful shutdown (wait <=100 msecs for existing reqs to complete):
(my-server :timeout 100)
```

## Production environments

http-kit server runs alone happily, which is handy for development and quick deployment.

But for production environments, it's **strongly recommended** to run http-kit behind a well-configured and battle-hardened reverse proxy like [nginx](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/), [Caddy](https://caddyserver.com/docs/quick-starts/reverse-proxy), [HAProxy](https://www.haproxy.org/), [AWS ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html), etc.

Your proxy can then provide:

1. **HTTPS** support (which http-kit lacks)
2. Load balancing
3. Static file serving performance
4. Improved security against un/intentional bad requests, DDoS attempts, etc.

http-kit's philosophy is to **focus on running Clojure code**, while leaving as much as possible to the rest of your stack.

###  Sample Nginx configuration

```
upstream my-pool {
	ip_hash;
	server localhost:8080;
	# Can add extra servers here
	keepalive 32;
}

server {
    location /static/ {  # static content
        alias   /var/www/public/;
    }
    location / {
    	proxy_http_version                 1.1;
    	proxy_pass                         http://my-pool;
    	proxy_pass_request_headers         on;
    	proxy_set_header Host              $host;
    	proxy_set_header X-Real-IP         $remote_addr;
    	proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    	proxy_set_header X-Forwarded-Proto $scheme;
    	proxy_set_header Upgrade           $http_upgrade;       # For WebSockets
    	proxy_set_header Connection        $connection_upgrade; # For WebSockets
        access_log  /var/log/nginx/access.log;
    }
}
```

## Websockets

There are 2 ways to handle WebSockets with `http-kit`:
 1. Use `http-kit`'s own unified API for WebSocket and HTTP long-polling
 2. Use [Ring's WebSocket API](https://github.com/ring-clojure/ring/wiki/WebSockets)

### Using http-kit's unified API

```clj
(ns ws-example.unified-api
  (:require
    [org.httpkit.server :as hk-server]))

(def channels (atom #{}))

(defn on-open    [ch]             (swap! channels conj ch))
(defn on-close   [ch status-code] (swap! channels disj ch))
(defn on-receive [ch message]
  (doseq [ch @channels]
    (hk-server/send! ch (str "Broadcasting: " message))))

(defn app [ring-req]
  (if-not (:websocket? ring-req)
    {:status 200 :headers {"content-type" "text/html"} :body "Connect WebSockets to this URL."}
    (hk-server/as-channel ring-req
      {:on-open    on-open
       :on-receive on-receive
       :on-close   on-close})))

(def server (hk-server/run-server app {:port 8080}))
```

### Using Ring's WebSocket API

```clj
(ns ws-example.ring-api
  (:require
    [org.httpkit.server :as hk-server]
    [ring.websocket :as ws]))

(def sockets (atom #{}))

(defn on-open    [ch]                    (swap! sockets conj ch))
(defn on-close   [ch status-code reason] (swap! sockets disj ch))
(defn on-message [ch message]
  (doseq [ch @sockets]
    (ws/send ch (str "Broadcasting: " message))))

(defn app [ring-req]
  (if-not (:websocket? ring-req)
    {:status 200 :headers {"content-type" "text/html"} :body "Connect WebSockets to this URL."}
    {::ws/listener
     {:on-open    on-open
      :on-message on-message
      :on-close   on-close}}))

(def server (hk-server/run-server app {:port 8080}))
```

These look the same on the surface, but let's dive deeper:

- With the unified API you can use the exact same code for both WebSockets and HTTP long-polling (e.g. as a fallback). With the Ring WebSocket API you'd need to write separate code to handle HTTP long-polling.
  
- With the unified API you detect send success or failure by checking the `send!` return value. With the Ring WebSocket API you detect send success or failure by providing callback functions.
  
- The unified API provides no `reason` on WebSocket close. The Ring WebSocket API does, though in practice the reason is often empty (as of Jun 2024). Both APIs provide a `status-code` on close.

# Advanced topics

## Custom request queues

http-kit server's worker pool can be easily customised for arbitrary control and monitoring, e.g.:

```clojure
(require '[org.httpkit.server :as hk-server])

(defn new-http-kit-pool
  [& {:keys [queue-capacity min-thread-count max-thread-count]
      :or   {queue-capacity 20480
             min-thread-count 4
             max-thread-count 8}}]

  (let [ptf   (org.httpkit.PrefixThreadFactory. "worker-")
        queue (java.util.concurrent.ArrayBlockingQueue. queue-capacity)
        pool  (java.util.concurrent.ThreadPoolExecutor.
                (long min-thread-count) (long max-thread-count)
                0 java.util.concurrent.TimeUnit/MILLISECONDS
                queue ptf)]

    {:queue queue
     :pool  pool}))

(defonce my-http-kit-pool (new-http-kit-pool))

(hk-server/run-server
  (fn my-handler [ring-req] {:body "hello"})
  {:worker-pool (:pool my-http-kit-pool)
   :port 11000})

(.size (:queue my-http-kit-pool)) ; Get current queue size
```

NB see also [`utils/new-worker`](https://cljdoc.org/d/http-kit/http-kit/CURRENT/api/org.httpkit.utils#new-worker) for an easy custom pool constructor.
### Java 19+ virtual threads

As above, but using [`java.util.concurrent.Executors/newVirtualThreadPerTaskExecutor`](https://docs.oracle.com/en/java/javase/19/docs/api/java.base/java/util/concurrent/Executors.html#newVirtualThreadPerTaskExecutor()):

```clojure
(require '[org.httpkit.server :as hk-server])

(hk-server/run-server
  (fn my-handler [ring-req] {:body "Hello"})
  {:worker-pool (java.util.concurrent.Executors/newVirtualThreadPerTaskExecutor)
   :port 11000})
```

NB see also [`utils/new-worker`](https://cljdoc.org/d/http-kit/http-kit/CURRENT/api/org.httpkit.utils#new-worker) for an easy custom pool constructor.

## Unix Domain Sockets (UDS)

http-kit 2.7+ supports server+client UDS on Java 16+.  
To use, plug in appropriate `java.net.SocketAddress` and `java.nio.channels.SocketChannel` constructor fns:

```clojure
(require '[org.httpkit.server :as hk-server])

(let [my-uds-path "/tmp/test.sock"
      my-server
      (hk-server/run-server my-routes
        {:address-finder  (fn []         (UnixDomainSocketAddress/of my-uds-path))
         :channel-factory (fn [_address] (ServerSocketChannel/open StandardProtocolFamily/UNIX))})]
  <...>
  )
```

See [`run-server`](http://http-kit.github.io/http-kit/org.httpkit.server.html#var-run-server) for more info.

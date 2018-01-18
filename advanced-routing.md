# Advanced Routing

In this workshop we'll deploy a linkerd-to-linkerd configuration and explore
advanced routing using transformers.

## Deploy hello and world

```
curl -Lo python.tgz 'https://github.com/adleong/linkerd-training/blob/master/python.tgz?raw=true'
tar xvzf python.tgz
sudo pip install --no-index --find-links=python/wheel -r python/requirements.txt
http_proxy=localhost:4140 python python/hello.py &
http_proxy=localhost:4140 python python/world.py &
```

## Deploy Linkerd

Write a Linkerd config

```
namers: []

routers:
- protocol: http
  label: outgoing
  servers:
  - port: 4140
  interpreter:
    kind: io.l5d.mesh
    experimental: true
    dst: /$/inet/localhost/4321
    root: /default
    transformers:
    - kind: io.l5d.port
      port: 4141

- protocol: http
  label: incoming
  servers:
  - port: 4141
  interpreter:
    kind: io.l5d.mesh
    experimental: true
    dst: /$/inet/localhost/4321
    root: /default
    transformers:
    - kind: io.l5d.localhost

telemetry:
- kind: io.l5d.recentRequests
  sampleRate: 1.0
```

Deploy Linkerd:

```
./linkerd-1.3.4-exec linkerd.yml
```

Send some requests:

```
http_proxy=localhost:4140 curl http://hello
```

## Transparent TLS

We'll start by generating a server certificate, key, and ca-chain using the
eg-ca tool:

```
git clone https://github.com/olix0r/eg-ca.git
cd eg-ca
./init.sh
./mksvc.sh linkerd
openssl pkcs8 -topk8 -nocrypt -in private.pem -out private.pk8
cp linkerd.tls/* ../linkerd-training
```

Now modify the Linkerd config to use these credentials:

```
namers: []

routers:
- protocol: http
  label: outgoing
  servers:
  - port: 4140
  interpreter:
    kind: io.l5d.mesh
    experimental: true
    dst: /$/inet/localhost/4321
    root: /default
    transformers:
    - kind: io.l5d.port
      port: 4141
  client:
    tls:
      trustCerts:
      - ca-chain.cert.pem
      commonName: linkerd

- protocol: http
  label: incoming
  servers:
  - port: 4141
    tls:
      keyPath: private.pk8
      certPath: cert.pem
  interpreter:
    kind: io.l5d.mesh
    experimental: true
    dst: /$/inet/localhost/4321
    root: /default
    transformers:
    - kind: io.l5d.localhost

telemetry:
- kind: io.l5d.recentRequests
  sampleRate: 1.0
```

Sending plain requests to Outgoing works:

```
curl localhost:4140 -H "Host: cat"
```

And requests to Incoming require TLS:

```
curl localhost:4141 -H "Host: cat"
```

## Egress

We will update our dtab to allow egress.  Move the transformers off of the
interpreters and directly onto the namers and update the dtab.

Namerd:

```
namers:
- kind: io.l5d.fs
  prefix: /io.l5d.fs.outgoing
  rootDir: disco
  transformers:
  - kind: io.l5d.port
    port: 4141
- kind: io.l5d.fs
  prefix: /io.l5d.fs.incoming
  rootDir: disco
  transformers:
  - kind: io.l5d.localhost

storage:
  kind: io.l5d.inMemory
  namespaces:
    outgoing: |
      /ph => /$/io.buoyant.rinet ;
      /svc => /ph/443 ;
      /svc => /$/io.buoyant.porthostPfx/ph ;
      /svc => /#/io.l5d.fs.outgoing ;
    incoming: |
      /svc => /#/io.l5d.fs.incoming ;

interfaces:
- kind: io.l5d.mesh
- kind: io.l5d.httpController
```

Linkerd:

```
namers: []

routers:
- protocol: http
  label: outgoing
  servers:
  - port: 4140
  interpreter:
    kind: io.l5d.mesh
    experimental: true
    dst: /$/inet/localhost/4321
    root: /outgoing
  client:
    kind: io.l5d.static
    configs:
    - prefix: /$/io.buoyant.rinet/443/{host}
      tls:
        commonName: "{host}"
    - prefix: /%/io.l5d.port/4141/
      tls:
        trustCerts:
        - ca-chain.cert.pem
        commonName: linkerd

- protocol: http
  label: incoming
  servers:
  - port: 4141
    tls:
      keyPath: private.pk8
      certPath: cert.pem
  interpreter:
    kind: io.l5d.mesh
    experimental: true
    dst: /$/inet/localhost/4321
    root: /incoming

telemetry:
- kind: io.l5d.recentRequests
  sampleRate: 1.0
```

Now it is possible to make both service-to-service calls and egress calls:

```
curl localhost:4140 -H "Host: cat"
curl localhost:4140 -H "Host: www.google.com"
```

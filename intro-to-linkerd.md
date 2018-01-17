# Intro to Linkerd

In this workshop we'll get started with Linkerd by deploying it locally and
using it to send traffic to some local python services.

## Set Up Local Linkerd

Start by creating two python servers

```
mkdir cat dog
cd cat
echo cat > index.html
python3 -m http.server 7777
(new tab)
cd dog
echo dog > index.html
python3 -m http.server 7778
```

Download linkerd

```
curl -LO https://github.com/linkerd/linkerd/releases/download/1.3.4/linkerd-1.3.4-exec
chmod +x linkerd-0.9.0-exec
```

Write linkerd config

```
namers:
- kind: io.l5d.fs
  rootDir: disco

routers:
- protocol: http
  dtab: /svc => /#/io.l5d.fs
  servers:
  - port: 4140

telemetry:
- kind: io.l5d.recentRequests
  sampleRate: 1.0
```

Create disco directory

```
mkdir disco
cat << EOF > disco/animal
> 127.1 7777
> 127.1 7778
> EOF
```

## Send Traffic

Send request

```
http_proxy=localhost:4140 curl http://animal
```

Send many requests

```
while true; do http_proxy=localhost:4140 curl http://animal; done
```

Visit admin dashboard

```
open http://localhost:9990
```

## Traffic Control

Create two service discovery entries:

```
echo "127.1 7777" > disco/cat
echo "127.1 7778" > disco/dog
```

Try some requests

```
http_proxy=localhost:4140 curl cat
http_proxy=localhost:4140 curl dog
```

Add per-request overrides

```
http_proxy=localhost:4140 curl http://cat -H "l5d-dtab: /svc/cat => /#/io.l5d.fs/dog"
```

These overrides will propagate through the call tree (if apps forward l5d-ctx-*
headers).  This means that they can be specified at ingress and can apply
to services deep in the stack.  Useful for doing staging or for inserting
debug proxies.

Add namerd into the mix.  Download namerd:

```
curl -LO https://github.com/linkerd/linkerd/releases/download/1.3.4/namerd-1.3.4-exec
chmod +x namerd-1.3.4-exec
```

Write namerd config:

```
namers:
- kind: io.l5d.fs
  rootDir: disco

interfaces:
- kind: io.l5d.httpController
- kind: io.l5d.mesh

storage:
  kind: io.l5d.inMemory
  namespaces:
    default: |
      /svc => /#/io.l5d.fs
```

Update linkerd config

```
namers: []
routers:
- protocol: http
  interpreter:
    kind: io.l5d.mesh
    dst: /$/inet/localhost/4100
    root: /default
  servers:
  - port: 4140

telemetry:
- kind: io.l5d.recentRequests
  sampleRate: 1.0
```

Open the Namerd UI:

```
open http://localhost:9991
```

Try some requests

```
http_proxy=localhost:4140 curl cat
http_proxy=localhost:4140 curl dog
```

Install namerctl

```
go get -u github.com/BuoyantIO/namerctl
go install github.com/BuoyantIO/namerctl
```

View dtab

```
namerctl --base-url=http://localhost:4180 dtab list
namerctl --base-url=http://localhost:4180 dtab get default
```

Make new dtab file:

```
/svc => /#/io.l5d.fs ;
/svc/cat => 1 * /#/io.l5d.fs/dog & 9 * /#/io.l5d.fs/cat ;
```

Update dtab:

```
namerctl --base-url=http://localhost:4180 dtab update default dtab
```

# multibinder, a docker experiment
The other day my friend @gviliarino pointed me to a great article on [how GitHub achieves 
zero-downtime HA Proxy reloads](http://githubengineering.com/glb-part-2-haproxy-zero-downtime-zero-delay-reloads-with-multibinder/).  

This is something I was looking for since day one, as we have the same problem in MURAL. But also 
as we use Docker containers and not bare metal machines, I wanted to take the solution for a spin. 

At this point, I made it work and it's great as it works on so many levels: 

- We no longer need fixed ports on the app container (as we user a docker network here)
- We can boot up a whole new version of our app in parallel (so no rolling deployment, no reduced capacity while deploying)
- No more "scheduled" deployments, as said above we won't be rolling, we will be deploying in parallel
- We can _dockerize_ HAProxy, and handle it's distribution and configuration as any other service.
- Our customers will be happier and we'll have an improved uptime

Let's get started

### Pre-requisites
We need a network and a volume to put the whole example to run. 

To create the network run:

```bash
docker network create test
```

To create the shared volume (used for the .sock file) run 

```bash 
docker volume create --name test-volume
```
### 1. Run multibinder 
This will run the `multibinder` container which will be used for proxying the HTTP 
connection to the HA Proxy servers.

First, let's build the container (while on the multibinder folder)

```bash 
docker build -t multibinder . 
```

And let's start it 

```bash 
docker run -it --rm -v test-volume:/run -p 80:80 multibinder
```

### 2. Run the first static container
This will run the backend server that HAProxy will connect to. This is tiny node container that only serves an index.html indicating the version.

First, let's build the container (while on the static folder)

```bash 
docker build -t static .
```

And now let's run it, as this is the v1 of our app, we'll make `v1.html` the index.html in this case. 

```bash
docker run -it --rm --name static-v1 -v $(PWD)/V1.html:/usr/src/index.html --network test static
```

### 3. Run the first HAProxy container
This will run the HAProxy container we'll connect throught he `multibinder`. This one will send all the incoming traffic to the server we previously created.

First, let's build the container (while on the haproxy folder)

```bash 
docker build -t haproxy .
```

And now let's run it, as this is the v1 of our app, we'll make `site.v1.cfg.erb` the `haproxy.cfg` template in this case.

```bash
docker run -it --rm --network test -v $(PWD)/site.v1.cfg.erb:/etc/haproxy/site.cfg.erb -v test-volume:/run haproxy
```

At this point if navigate to http://localhost you should get `V1` as the server response.

### 4. Run the test script 
This tiny bash script will `curl http://locahost` continuously and print the response on the screen.

```bash 
λ sh test.sh
V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1V1
```
You should get the output shown above.

### 5. Run the second version of the static container
Now that that we have the v1 of our app, we're going to publish a new version. We'll create a new container of the static image but with `V2.html` as the `index.html`.

As this is the same image we built before, we don't need to rebuild it.

```bash
docker run -it --rm --name static-v2 -v $(PWD)/V2.html:/usr/src/index.html --network test static
```

### 6. Run the second version (configuration) of the HAProxy.
This container based on the previous image we built but with `site.v2.cfg.erb` as the config will also hook up to the `.sock` file created by `multibinder`. 

```bash
docker run -it --rm --network test -v $(PWD)/site.v2.cfg.erb:/etc/haproxy/site.cfg.erb -v test-volume:/run haproxy
```

At this point you will see on the output test script that some V2 already: 

```bash
λ sh test.sh
V1V2V2V2V2V2V1V1V2V1V1V2V1V2V1V1V1V2
```
### 7. Kill the old version
At this point it will be safe to remove the V1 containers. Let's start with `haproxy` to prevent traffic hitting a server that is stopped (if we stop the static-v1 first, it will result on client errors).

```bash 
docker stop romantic_payne
```

NOTE: `romantic_payne` is the auto name that docker assigned to the HAProxy container v1 on my computer. Also calling docker stop will do a graceful shutdown to prevent traffic being lost.

At this point, our test script should be reporting only `v2`.

```bash
V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2V2
```

Now we can kill or stop the static-v1 container as it's no longer receiving traffic.

```bash
docker stop static-v1
```

At this point we have "our production" service update with new HAProxy without a single request lost, and without affecting our customers.

### Remarks
Here are some of the most important thoughts about this `docker` and `multibinder` experiment.

* The only exposed port is on the `multibinder` container that's were our traffic routed.
* HAProxy containers expose no ports to the outside, hence we can have multiple instances running on the same machine. 
* The static container could be anything that can receive HTTP Traffic, on this sample I used node and a static file just to illustrate the point. 
* `multibinder` image is completely agnostic, it's based on `ruby:2.3.1` and has no references to HA Proxy at all, so you can use it with your own service. 
* HAProxy image is based on Ubuntu because it was faster and easier to install `ruby` and `haproxy`, but I think that it could be the `haproxy` image with `ruby` added to it (as long as it's 1.6)

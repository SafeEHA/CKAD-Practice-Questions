# Container Management - Practice Questions

This section covers container building, registry operations, and container runtime management with Docker and Podman.

## Question 11 | Working with Containers

**Solve this question on instance:** `ssh ckad9043`

During the last monthly meeting you mentioned your strong expertise in container technology. Now the Build&Release team of department Sun is in need of your insight knowledge. There are files to build a container image located at /opt/course/11/image. The container will run a Golang application which outputs information to stdout. You're asked to perform the following tasks:

NOTE: Make sure to run all commands as user candidate, for docker use sudo docker

1. Change the Dockerfile. The value of the environment variable SUN_CIPHER_ID should be set to the hardcoded value 5b9c1065-e39d-4a43-a04a-e59bcea3e03f
2. Build the image using Docker, named registry.killer.sh:5000/sun-cipher, tagged as latest and v1-docker, push these to the registry
3. Build the image using Podman, named registry.killer.sh:5000/sun-cipher, tagged as v1-podman, push it to the registry
4. Run a container using Podman, which keeps running detached in the background, named sun-cipher using image registry.killer.sh:5000/sun-cipher:v1-podman. Run the container from candidate@ckad9043 and not root@ckad9043
5. Write the logs your container sun-cipher produced into /opt/course/11/logs. Then write a list of all running Podman containers into /opt/course/11/containers on ckad9043

<details>
<summary>Answer</summary>

- Dockerfile: list of commands from which an Image can be build
- Image: binary file which includes all data/requirements to be run as a Container
- Container: running instance of an Image
- Registry: place where we can push/pull Images to/from

**1.**

First we need to change the Dockerfile to:

```dockerfile
# build container stage 1
FROM docker.io/library/golang:1.15.15-alpine3.14
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o bin/app .

# app container stage 2
FROM docker.io/library/alpine:3.12.4
COPY --from=0 /src/bin/app app
# CHANGE NEXT LINE
ENV SUN_CIPHER_ID=5b9c1065-e39d-4a43-a04a-e59bcea3e03f
CMD ["./app"]
```

**2.**

Then we build the image using Docker:

```bash
➜ cd /opt/course/11/image

➜ sudo docker build -t registry.killer.sh:5000/sun-cipher:latest -t registry.killer.sh:5000/sun-cipher:v1-docker .
...
Successfully built 409fde3c5bf9
Successfully tagged registry.killer.sh:5000/sun-cipher:latest
Successfully tagged registry.killer.sh:5000/sun-cipher:v1-docker

➜ sudo docker image ls
REPOSITORY                           TAG         IMAGE ID       CREATED              SIZE
registry.killer.sh:5000/sun-cipher   latest      409fde3c5bf9   24 seconds ago       7.76MB
registry.killer.sh:5000/sun-cipher   v1-docker   409fde3c5bf9   24 seconds ago       7.76MB
...

➜ sudo docker push registry.killer.sh:5000/sun-cipher:latest
The push refers to repository [registry.killer.sh:5000/sun-cipher]
c947fb5eba52: Pushed 
33e8713114f8: Pushed 
latest: digest: sha256:d216b4136a5b232b738698e826e7d12fccba9921d163b63777be23572250f23d size: 739

➜ sudo docker push registry.killer.sh:5000/sun-cipher:v1-docker
The push refers to repository [registry.killer.sh:5000/sun-cipher]
c947fb5eba52: Layer already exists 
33e8713114f8: Layer already exists 
v1-docker: digest: sha256:d216b4136a5b232b738698e826e7d12fccba9921d163b63777be23572250f23d size: 739
```

There we go, built and pushed.

**3.**

Next we build the image using Podman. Here it's only required to create one tag. The usage of Podman is very similar (for most cases even identical) to Docker:

```bash
➜ cd /opt/course/11/image

➜ podman build -t registry.killer.sh:5000/sun-cipher:v1-podman .
...
--> 38adc53bd92
Successfully tagged registry.killer.sh:5000/sun-cipher:v1-podman
38adc53bd92881d91981c4b537f4f1b64f8de1de1b32eacc8479883170cee537

➜ podman image ls
REPOSITORY                          TAG         IMAGE ID      CREATED        SIZE
registry.killer.sh:5000/sun-cipher  v1-podman   38adc53bd928  2 minutes ago  8.03 MB
...

➜ podman push registry.killer.sh:5000/sun-cipher:v1-podman
Getting image source signatures
Copying blob 4d0d60db9eb6 done  
Copying blob 33e8713114f8 done  
Copying config bfa1a225f8 done  
Writing manifest to image destination
Storing signatures
```

Built and pushed using Podman.

**4.**

We'll create a container from the perviously created image, using Podman, which keeps running in the background:

```bash
➜ podman run -d --name sun-cipher registry.killer.sh:5000/sun-cipher:v1-podman
f8199cba792f9fd2d1bd4decc9b7a9c0acfb975d95eda35f5f583c9efbf95589
```

**5.**

Finally we need to collect some information into files:

```bash
➜ podman ps
CONTAINER ID  IMAGE                                         COMMAND     ...       
f8199cba792f  registry.killer.sh:5000/sun-cipher:v1-podman  ./app       ...

➜ podman ps > /opt/course/11/containers

➜ podman logs sun-cipher
2077/03/13 06:50:34 random number for 5b9c1065-e39d-4a43-a04a-e59bcea3e03f is 8081
2077/03/13 06:50:34 random number for 5b9c1065-e39d-4a43-a04a-e59bcea3e03f is 7887
2077/03/13 06:50:34 random number for 5b9c1065-e39d-4a43-a04a-e59bcea3e03f is 1847
2077/03/13 06:50:34 random number for 5b9c1065-e39d-4a43-a04a-e59bcea3e03f is 4059
2077/03/13 06:50:34 random number for 5b9c1065-e39d-4a43-a04a-e59bcea3e03f is 2081
2077/03/13 06:50:34 random number for 5b9c1065-e39d-4a43-a04a-e59bcea3e03f is 1318
2077/03/13 06:50:34 random number for 5b9c1065-e39d-4a43-a04a-e59bcea3e03f is 4425
2077/03/13 06:50:34 random number for 5b9c1065-e39d-4a43-a04a-e59bcea3e03f is 2540
2077/03/13 06:50:34 random number for 5b9c1065-e39d-4a43-a04a-e59bcea3e03f is 456
2077/03/13 06:50:34 random number for 5b9c1065-e39d-4a43-a04a-e59bcea3e03f is 3300
2077/03/13 06:50:34 random number for 5b9c1065-e39d-4a43-a04a-e59bcea3e03f is 694
2077/03/13 06:50:34 random number for 5b9c1065-e39d-4a43-a04a-e59bcea3e03f is 8511
2077/03/13 06:50:44 random number for 5b9c1065-e39d-4a43-a04a-e59bcea3e03f is 8162
2077/03/13 06:50:54 random number for 5b9c1065-e39d-4a43-a04a-e59bcea3e03f is 5089

➜ podman logs sun-cipher > /opt/course/11/logs
```

This is looking not too bad at all. Our container skills are back in town!

</details>
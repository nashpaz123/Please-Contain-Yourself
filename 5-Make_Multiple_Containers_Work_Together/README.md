# Module 5 - Make Multiple Containers Work Together


- [ ] Check boxes are steps that need to be completed.

![dokernetwork](https://github.com/dylanlrrb/P-C-Y-Assets/blob/master/5/dockernetwork.png?raw=true)

## Preface: Intro to Microservices

Now, you might be wondering: **"Why not just layer everything I need into one MONOLITHIC container? I could just create one massive container with everything I need and, BAM, I got an app"**

**Overall, using microservice architecture allows you to build software that is adaptable, composable, autonomous, and fault tolerant.**
So, **"How on earth do I get my containers to communicate with each other?"**

---
## Docker Networks - let's learn about em'!

---


Module 5 comes with yet another app, and in the spirit of using microservice architecture, it is split into 3 services: a 'survey server', a 'results server', and a MongoDB database.

- The survey server has two jobs, rendering a form that lets a user create a database entry and saving that entry to the database via an API endpoint.

---
>Normally, microservices expose their interfaces with a standard protocol, such as a REST-ful API, and they can be consumed and reused by other services and applications without direct coupling through language bindings or shared libraries.
>
>Services exist as independent deployment artifacts and can be scaled independently of other services.


---

- The results server also has two jobs, retrieving the saved entry data from the database and then rendering that data on a webpage via an API endpoint

- The MongoDB database is... a MongoDB database. It stores stuff you tell it to store.

Now, assuming that you've got MongoDB, node.js, and npm installed on your computer this app would work flawlessly. **HOWEVER!** The beauty of containers is that you can wrap your app up and run it anywhere, regardless of if the host has the needed dependencies installed. (So, for the sake of this Module, even if you do have all the right dependencies installed, let's pretend you don't).

Docker is the only thing you need from now on! So let's get started spinning the different parts of our app into containerized microservices!


**First:** Let's build those images:

- [ ] Build your survey server image and tag it with the name 'survey' by running `docker build -t survey .`

**REMEMBER the dot is a relative file path to where your Dockerfile lives (one is included for each server), so MAKE SURE you are cd'd into the correct directory before running this command**

- [ ] Build your results server image and tag it with the name 'results' by running `docker build -t results .`

- [ ] Pull down a MongoDB image by running `docker pull mongo:latest`

This saves us a lot of time - since we straight up just need a MongoDB database and don't need to configure it in any way we'll just pull the official image from Dockerhub. As with every other time you used the `docker pull` command it grabs the image from Dockerhub and caches it on your machine. This time is no different. However, the way you spin up the official MongoDB image is a little different than we are used to; let's check it out ->

**Second:** Run the containers:

- [ ] First, we'll spin up the MongoDB container. Run `docker run -d --name mongo mongo:latest`

You might be wondering, **'Where's the -p port mapping option that we've used EVERY OTHER TIME?!?!?'**

'Good catch.', I would respond. But I would ask you back, 'Why would we need it to be mapped to a port on the host?' **The container only needs to talk to other containers and therefore it's exposed port can remain 'virtual'**

Go ahead and run `docker logs mongo`. You'll see something like this at the very end of the log:

```sh
I NETWORK  [thread1] waiting for connections on port 27017
```
"This image includes `EXPOSE 27017` (the mongo port), so standard container networking will make it automatically available to containers on the same network" -- [Official MongoDB image](https://hub.docker.com/_/mongo/)

- [ ] Next let's spin up the other two server images we made, survey and results

Be sure to also mount volumes in the respective directories just in case, i don't know, we want to make quick changes to the source code. *cough cough* (we will)

- [ ] cd into '/survey_server' and run:

`docker run -d -p 8080:8080 -v $(pwd):/src/app --name survey_container survey`

- [ ] cd into '/results_server' and run:

`docker run -d -p 3000:3000 -v $(pwd):/src/app --name results_container results`

- [ ] Now pull up the app by navigating to `localhost:8080` and `localhost:3000` in your browser! You should definitely have your hopes up because I totally didn't give you broken code on purpose!

**(Just kidding, I gave you broken code on purpose)**

- [ ]  As you can see, the app is completly broken. Fortunatly, to get it working, we only have to tweak one line of code in each server. But which one? Run the command `docker logs survey_container` to look for clues inside the container's logs...

You should see that the app has crashed and, examining the error stack, you should notice that the distal cause for the crash is described with the log:

```sh
events.js:161
      throw er; // Unhandled 'error' event
      ^
MongoError: failed to connect to server [localhost:27017] on first connect
```

Looks like the survey_container server crashed because it couldn't connect to the mongo database container. (Running `docker logs` on the results_container will yeild the same result) Here are some observations:

- On line 8 in index.js of both the survey and results server are trying to connect to the 'mongodb://localhost:27017/docker_test'

- Our mongo container is not mapped to any port on 'localhost', so that ain't gonna work.

- The mongo container has port 27017 exposed

To talk about how to solve this communication problem, I first need to formally introduce Docker Networking. I like to think of Docker Networks as a kind of 'internet' that only docker containers can use. But there's a cool little catch - you can create many isolated networks so that containers connected to a network can all talk to each other but not containers on other networks! This is super useful for controlling the flow of possibly sensitive data between containers.

- [ ] Docker comes installed with 3 ready-to-go networks, to check them out run the command `docker network ls`

You should see something like this:

```sh
NETWORK ID          NAME                DRIVER              SCOPE
b2062552b0c7        bridge              bridge              local
e9fbd2d053dc        host                host                local
b4a8c47fad5c        none                null                local
```

There are 3 networks listed here: bridge, host, and none. We're only going to go in-depth into the the bridge network but if you want more information about the others, [check out this resource](https://docs.docker.com/engine/userguide/networking/).

You might be wondering, **"Hey, when I spun up all the containers for this module I never specified what network to connect them to! Are they just floating in space not connected to anything?"**

But that would be silly. When you spin up a container without specifying a network to connect it to (**Something we will learn how to do in little bit**) it is AUTOMATICALLY connected to the 'bridge' network that comes with Docker by default.

- [ ] Good to know that they're not floating out in space somewhere, but I'd feel better if we had some visual proof of where they are connected. Run `docker network inspect bridge`

The `docker network inspect` command gives you an overview of the network whose name you specify in the last argument of the command. In this case we are inspecting the 'bridge' network and you should see something like this logged:

```sh
[
    {
        "Name": "bridge",
        "Id": "b2062552b0c7f2d189b6f62d7aabcde8aa5aa1ab564239227106a8033ac846f1",
        "Created": "2017-03-07T23:17:59.147774884Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "9549b4c14baf6702a9132d746bad8326b538daf0edb3af206724d43754eadb4e": {
                "Name": "mongo",
                "EndpointID": "fcd9382a73994369f22555b1510569d6baba88a4710296d100885e7b7c54049b",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "b51b76fc34fdaf212ada4f5f61a11b1aa20767a8f1814a777adae888c24fe949": {
                "Name": "survey_container",
                "EndpointID": "afc28c0e9e541c6d8d97ed429b32f94cf9524c3535b9879a1d1bcbe9eb781308",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            },
            "fcac3b3d18b69c3e59f89b7a6466c87ea8a2689b1fcefd72dae755e4f175065d": {
                "Name": "results_container",
                "EndpointID": "1311856d866aed29f40711d0076bf956680a677a64bfd941ee5c101ee9ed99c6",
                "MacAddress": "02:42:ac:11:00:04",
                "IPv4Address": "172.17.0.4/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

The one thing that I would like you to notice is the "Containers": property on this [JSON object](https://www.w3schools.com/js/js_json_objects.asp) that was logged. Here, you can see listed all the containers that are currently running and attached to the 'bridge' network that we're inspecting.

**Let's take a step back and think about what we're trying to accomplish here. We need our survey and results server containers to be able to connect to the mongo container that we're using to persist data.**

- To connect to the mongo container, we need to know where it is.

- We now know that it is connected to the 'bridge' network

- The survey and results containers are also running on the 'bridge' network

- Since all the containers are running on the same network they SHOULD be able to communicate

- **But what address goes in the string on line 8 in 'index.js' that would allow containers to send information to each other??**

The answer to all your questions is in that `docker network inspect` log. Since everyone needs to connect to the mongo container, check out the mongo container's information a little more closely

```sh
"9549b4c14baf6702a9132d746bad8326b538daf0edb3af206724d43754eadb4e": {
                "Name": "mongo",
                "EndpointID": "fcd9382a73994369f22555b1510569d6baba88a4710296d100885e7b7c54049b",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
```

Remember how I said that Docker Networks are kind of like an 'internet' within Docker? Well, turns out that containers have their own IP addresses that you need to use in any sort of network request between containers. And that little bit of critical information is listed right under the property "IPv4Address". The address you can find the mongo container with in this case is:

```sh
172.17.0.2
```

- [ ] Open up the 'index.js' for both the survey_server and the results_server

- [ ] On line 8 of both files, change the address string that the server is trying to connect to the database with. Inside the string, change the address 'localhost' to the IPv4Address of the mongo container

**This address may be different for you, so make sure you inspect the network on your own machine to find it!**

Because you mounted volumes with the two server containers that you spun up, this change in the source code should be immediately  reflected in the containers (**once you save, of course**).

- [ ] Navigate to `localhost:8080` and `localhost:3000` in separate tabs to check out the app!

- [ ] Fill out the form provided on `localhost:8080` and hit submit to create an entry. Then at `localhost:3000` refresh the page to show all the entries listed!

- [ ] **After you are done with these containers, be sure to remove them (or at least stop them). Nodemon's file watching feature can sometimes take up a chunk of CPU power** Run:

`docker rm -f survey_container results_container`

Congratulations!!! By splitting up the app into different containers and having them communicate, you’ve successfully implemented a microservices architecture!

**If something went wrong, try debugging your containers with `docker logs`, it is possible that you failed to mount your volumes properly or typed in the wrong address to connect to.**

---
>**REMEMBER:** The image still has the broken code in it, so if you want to spin up a container without needing a volume mounted, you will need to build a new image with the working code.
Also, there would be no need to watch for file changes in the final container, so you need change your dockerfile accordingly.

Wouldn't it be amazing if there were some way to automatically coordinate all these containers so you didn't have to build all the images and run all the containers separately? What if I told you there's a way to start up a multi-container app with one command??

If you would like to be enlightened by these ancient and powerful secrets, continue to [Module 6 - Docker Compose for Multi-Container Apps](https://github.com/nashpaz123/Please-Contain-Yourself/tree/master/6-Docker_Compose_For_Multi-Container_Apps)

---

## Extra Credit

- [Clean up after Docker!](http://blog.yohanliyanage.com/2015/05/docker-clean-up-after-yourself/)
- [Running commands inside a Docker container](https://docs.docker.com/engine/reference/commandline/exec/)
- [User-defined Docker Networks](https://docs.docker.com/engine/userguide/networking/#user-defined-networks)

---
#### Things we've learned:

- Microservices architecture
- Docker networks
- Default docker networks
- `docker network ls`
- `docker network inspect <network-name>`
- IPv4 Addressing


#### Resources
- [MuleSoft: Best Practices for Microservices](https://github.com/dylanlrrb/P-C-Y-Assets/blob/master/5/best.pdf)
- [Microservices Cheatsheet](https://github.com/cehsu/Lightningtalk/blob/master/Microservices.md)
- [Docker Container Networking](https://docs.docker.com/engine/userguide/networking)
- [Docker Network Command](https://docs.docker.com/engine/reference/commandline/network/#usage)
- [Persist data outside containers](https://docs.docker.com/engine/tutorials/dockervolumes/)

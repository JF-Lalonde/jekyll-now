---
layout: post
title: How to host a Rails app on a Raspberry Pi cluster with Docker Swarm
published: true
date: '2017-11-06 08:19'

---

<figure>
  <img alt="" src="/images/2017-11-06-how-to-host-a-rails-app-on-a-raspberry-pi-cluster-with-docker-swarm/1*QYJi0Ra4kAKwU6q_K3p4fQ.jpeg" title="" />
  <figcaption>Cluster comprised of 3 x Raspberry Pi 3 Model&nbsp;B</figcaption>
</figure>

Lately I’ve been hearing a lot of talk about containers (Docker, Rkt) and container orchestration (Kubernetes, Docker Swarm, Marathon). After doing some research into just what exactly all these terms actually mean ([https://docs.docker.com/engine/docker-overview/](https://docs.docker.com/engine/docker-overview/) , [https://kubernetes.io/docs/setup/](https://kubernetes.io/docs/setup/)) I realized that I was intrigued.

How do these containers actually host an app? And how does orchestration allow these apps to scale when under load?

I decided that the best way for me to understand more about containers was to play with them! Although I could have created a cluster using virtual machines, I am a visual person and the idea of playing with some hardware to solidify concepts is what drew me to the Raspberry Pi.

I was fortunate enough to borrow my way into three Raspberry Pi 3s, which are the models that have onboard wifi chips. This means that they can be connected into a cluster without the need for ethernet cables or a switch.

So, armed with my cluster of mini-computers, I set out to create my goal: a network comprised of three nodes, each running Docker. I would have one manager and two worker nodes, and they would all be overseen by Docker Swarm (I have future plans of switching to Kubernetes, but at the time of this article the K8s dashboard was giving me issues).

With this goal in mind, let’s get started with everything you need to know to host an app on a bare metal cluster.

---

## Step 1: Setting up each Raspberry Pi

Acquire however many Raspberry Pis you can get your hands on. I recommend using at least 2 with Docker Swarm, and if you’re looking to implement Kubernetes 3 would be better, as you cannot use the K8s master node to host a container. Just be sure to get the model 3 Pi, as this tutorial will be leveraging their wifi capabilities.

You will also need one microSD card for each Pi. I recommend at least 16gb cards just to ensure you have plenty of room. This tutorial will be using a Macbook Pro to flash the OS, but there are many resources available online if you have a different setup.

We will be installing Raspbian Stretch (with Desktop), which you can download [here](https://www.raspberrypi.org/downloads/raspbian/). Once downloaded, insert your microSD card via an SD card adapter into your machine. Run Disk Utility and make note of your card’s device name (ex. “disk2”). You’ll want to format your card before you begin. I’ve found [SD Card Formatter](https://www.sdcard.org/downloads/formatter_4/) to work well and quickly. Just unmount your card in Disk Utility before running the formatter.

Once your card is formatted, use the terminal and navigate to the directory that contains your Raspbian image.

You’ll want to use the following to flash the image to your disk:

```html
sudo dd bs=1m if=<image-name>.img of=/dev/r<device-name>
```

Which in my case looked like this:

```js
sudo dd bs=1m if=2017-09-07-raspbian-stretch.img of=/dev/rdisk2
```

**This should take ~5–10 min per card.**

After flashing the OS on your cards we’ll want to enable SSH so that we can control each Pi from our machine, eliminating the need to plug each one into a monitor.

**To enable SSH:**

Navigate to your new disk, and in the /boot/ directory create a file called SSH:

```
cd /Volumes/boot/
touch SSH
```

You’ll need to do this on each SD card.

**Then, enable WiFi:**

There are two ways to do this:

-   You could plug each Pi into a monitor (after inserting the SD cards into them) and use the Desktop client to select your network and fill in the password

OR

-   You can navigate to /boot/ and create a wpa\_supplicant.conf file

```
cd /Volumes/boot/
touch wpa_supplicant.conf
```

Inside the wpa\_supplicant.conf type the following:

```js
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
network={
    ssid="YOUR_WIFI_NETWORK_NAME"
    psk="YOUR_WIFI_PASSWORD"
    key_mgmt=WPA-PSK
}
```

You should now have SSH and WiFi enabled! Go ahead and insert your microSD cards into each Raspberry Pi.

We will now want to find the IP address of each Pi. I found installing and using [Nmap](https://nmap.org/book/inst-macosx.html) to be the most convenient. Find your computer’s IP address and then scan the subnet range to gather all of your device IPs by using:

```html
nmap -sn <Computer IP>.0/24
```

If your computer’s IP address is 192.168.0.4 swap out the last “4” with “0/24”, for example.

This should return a list of all devices in that range. Search for all the devices named “Raspberry Pi Foundation” and mark their IP addresses down.

You can now SSH into each Pi by using

```css
ssh pi@<IP ADDRESS>
```

This will prompt you for a username/password which are by default pi and raspberry, respectively.

**You can now SSH into each Pi from your machine!**

---

## Step 2: Setting up Docker

Now that you can SSH into each Pi, go ahead and run the following on each of them:

```js
ssh pi@<ADDRESS> curl -fsSL get.docker.com -o get-docker.sh
ssh pi@<ADDRESS> sudo sh get-docker.sh
```

Ref: [https://github.com/docker/docker-install](https://github.com/docker/docker-install)

This will install docker on each Pi. To make sure that docker is installed, you can SSH into each Pi and run

```
sudo docker version
```

Which should return something like this:

```sql
pi@masternode:~ $ sudo docker version
Client:
 Version:      17.10.0-ce
 API version:  1.33
 Go version:   go1.8.3
 Git commit:   f4ffd25
 Built:        Tue Oct 17 19:13:44 2017
 OS/Arch:      linux/arm

Server:
 Version:      17.10.0-ce
 API version:  1.33 (minimum version 1.12)
 Go version:   go1.8.3
 Git commit:   f4ffd25
 Built:        Tue Oct 17 19:06:18 2017
 OS/Arch:      linux/arm
 Experimental: false
```

If you’ve got docker up and running on each Pi you’re ready for the next step!

---

## Step 3: Run Docker Swarm

Docker Swarm comes included with Docker as of version 1.12.0 and beyond.

To start a swarm, choose one node to be your manager node. SSH into that node and run

```html
sudo docker swarm init --advertise-addr <IP ADDRESS>
```

You will be using that node’s IP address to broadcast to other nodes so they can join the cluster. If you want you can run multiple manager nodes, but this tutorial assumes one manager and two worker nodes.

The result of the previous command returns something like the following:

```css
docker swarm join --token SWMTKN-1-23zfgr7dr50a5jgum9d7lkt2013bsgidun9cm5246xsofeq4h-1x7pfnixwkasv0byz9qhuivy 193.165.0.1:2377
```

This is what will be used to allow worker nodes to join the cluster.

SSH into each worker node and paste the above line into them. If everything worked the nodes will return a confirmation that they joined successfully.

SSH back into the manager node and run

```
sudo docker node ls
```

Which should return all the nodes: (IDs will be much longer than this)

```rb
pi@masternode:~ $ sudo docker node ls
ID  HOSTNAME        STATUS      AVAILABILITY        MANAGER STATUS
9     firstworker   Ready         Active
vo *  masternode    Ready         Active              Leader
v17   secondworker  Ready         Active
```

**Congrats, you’ve got a cluster of Raspberry Pis running with Docker Swarm!**

---

## Step 4: Run visualization service

Before we dockerize an app, let’s set up a way to visually keep track of where our app will be hosted at any given time.

The Docker Swarm Visualizer is a great way of seeing what nodes are active, their respective roles, and what containers are being run on what nodes.

Ref: [https://github.com/dockersamples/docker-swarm-visualizer](https://github.com/dockersamples/docker-swarm-visualizer)

Follow the steps to install the Visualizer on ARM architecture (The Raspberry Pi version 3 is built using ARMv7)

On manager node:

```sql
sudo docker service create \
        --name viz \
        --publish 8080:8080/tcp \
        --constraint node.role==manager \
        --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
        alexellis2/visualizer-arm:latest
```

**This will take up to 15 min.**

To check if the service was created and is running, type

```
sudo docker service ls
```

Which should show 1/1 under the **REPLICAS** column

You can now view the visualizer in your browser by going to the IP address of your manager node and adding :8080 to the end (8080 is the port number)

You should see something like this:

<figure>
  <img alt="" src="/images/2017-11-06-how-to-host-a-rails-app-on-a-raspberry-pi-cluster-with-docker-swarm/1*md0C92eawm-hTv8nnGsKSQ.png" title="" />
  <figcaption>Initial state of the swarm with the visualizer service&nbsp;running</figcaption>
</figure>

---

## Step 5: Run an app as a docker image

Now that we can see what is happening in our cluster, let’s host an app. I am using a basic Rails CRUD app called [Climbing World](https://climbing-world.herokuapp.com/).

First, we will need to create a Docker image of our Rails app in order for the containers to host it.

Start by creating a repo on [Docker Hub](https://hub.docker.com/) where you can push and pull your Docker images. As you update an app you can push changes, and when Swarm creates new instances of your app it can pull down from the repo.

Now that you have an account, we will focus on creating the Docker image.

Since I had an existing Rails app on my machine, but I wanted to create an image on my manager node, I had to copy my app over to the Raspberry Pi.

I ended up using secure copy (scp) to transfer everything over, but that included many unnecessary files like all previous logs, commits, and cache files. If you chose to go this route, this is how to do it:

```rb
scp -r /path/to/local/dir user@remotehost:/path/to/remote/dir
```

**This took ~20 min.**

I used Docker-Compose in order for Nginx, Postgres, and Rails to all live in separate container tied together. Using Docker-Compose necessitated installation. The most straightforward way is to install “pip” first:

```sql
apt-get -y install python-pip

pip install docker-compose
```

Once Docker-Compose is installed, go to the root of your Rails project.

We’ll create some setup files following Chris Stump’s excellent guide: [http://chrisstump.online/2016/02/20/docker-existing-rails-application/](http://chrisstump.online/2016/02/20/docker-existing-rails-application/)

<figure>
  <img alt="" src="/images/2017-11-06-how-to-host-a-rails-app-on-a-raspberry-pi-cluster-with-docker-swarm/1*U7nARPt_FDAvkeu2lPviMA.png" title="" />
  <figcaption>First container getting ready with Postgres&nbsp;Database</figcaption>
</figure>

**Once you’re done following these instructions you should have a web app deployed using Docker!**

---

## Step 6: Use Docker Swarm to scale and load-balance your app

Now that we have an app running on Docker, let’s leverage Docker Swarm to create two instances of our app.

To make use of our Swarm, let’s create a network for our services:

```sql
sudo docker network create climb_net --driver overlay
```

The overlay driver allows Swarm to access the network.

Let’s create two replicas of our app to see where they go:

```sql
sudo docker service create --replicas 2 \
--env POSTGRES_PASSWORD=blank --name climbingworld_app \
--network climb_net postgres:9.5
```

You can chose to specify a Postgres password but setting it to blank works by default.

We can see that our app was created twice, and is running on different nodes. Note that Docker Swarm allows the manager node to run docker images too.

![](/images/2017-11-06-how-to-host-a-rails-app-on-a-raspberry-pi-cluster-with-docker-swarm/1*yR0AhYAxbAzuxjTi9hEOxw.png)

Let’s see how this looks when taken to a higher level:

```sql
--replicas 20
```

<figure>
  <img alt="" src="/images/2017-11-06-how-to-host-a-rails-app-on-a-raspberry-pi-cluster-with-docker-swarm/1*AQgpDfoWvJjS13w-13RfuA.png" title="" />
  <figcaption>Had to cut off image but more containers are running&nbsp;below.</figcaption>
</figure>

We can see that the application instances are running on each node and are distributed equally, since they are being used equally at the moment.

Another example of the load balancer in action is when I cut the power to one of the worker nodes, all containers are moved to the other worker node

<figure>
  <img alt="" src="/images/2017-11-06-how-to-host-a-rails-app-on-a-raspberry-pi-cluster-with-docker-swarm/1*2l9nBkzJDtVdQEvXgYd8ew.png" title="" />
  <figcaption>In this example, there were two containers running on “firstworker”. After cutting power to the node, all containers were moved to “secondworker”</figcaption>
</figure>

You can also observe the load on CPU and Memory usage using the command:

```js
sudo docker stats $(docker inspect -f {{.Name}} $(docker ps -q))
```

Which returns the following:

![](/images/2017-11-06-how-to-host-a-rails-app-on-a-raspberry-pi-cluster-with-docker-swarm/1*gH5xiWfcSH82gxpj57KmCg.png)

**Now you have an app orchestrated by Docker Swarm and scaling at your demand!**

---

## In the Future:

There are things that I wanted to do in this project that I wasn’t able to for various reasons.

I tried implementing stress testing to demonstrate how containers would move from node to node, or how new containers could automatically be created when needed. However the stress testing software I found was difficult to work with out of the four that I tried, only jfusterm’s [Stress](https://hub.docker.com/r/jfusterm/stress/) worked (and to a limited extent at that)

The only stress test I was able to run was by starting containers that consume 100% of the server’s CPU. However they were all shutdown immediately. This led me to rethink my assumptions.

In the future I hope to run Kubernetes and use the [Kubernetes Dashboard](https://github.com/kubernetes/dashboard) to better show realtime values like CPU and Memory usage.

<figure>
  <img alt="" src="/images/2017-11-06-how-to-host-a-rails-app-on-a-raspberry-pi-cluster-with-docker-swarm/1*qnrjh9zzPDmZVIcR-L4K_A.png" title="" />
  <figcaption>K8s dashboard can show realtime CPU and Memory&nbsp;load</figcaption>
</figure>

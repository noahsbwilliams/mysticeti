# ⚠️ **PWNED!** Rotate secrets! ⚠️

---

# [alebedev87](https://github.com/alebedev87)'s cool solution for containous/jobs

> Andrey did a fantastic job at this, and I learned some new Docker ps arguments from his work.

This is Andrey's solution for the funny puzzle used by Containous company.  
The puzzle is available as a Docker image: `containous/jobs`.

## Previous Containous puzzle
See [this article](https://containo.us/blog/solving-containous-jobs-b4a5cae04f3/) for the previous Containous' job puzzle, it can be helpful to kickoff with their new one.

## First run
```bash
$ docker run containous/jobs
Mysticeti, where are you ? :'(
```
Mysticeti?! What's that? It's a name for [Ballen whale](https://en.wikipedia.org/wiki/Baleen_whale).      
Why are they looking for a whale?! Well, Docker's logo is a whale too, hm, a coincidence? (I don't think so).

## Image internals
Let's take a look at how the image was created:
```bash
$ docker history containous/jobs
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
870d3fa8d182        2 years ago         /bin/sh -c #(nop)  ENTRYPOINT ["/mysticeti"]    0B                  
<missing>           2 years ago         /bin/sh -c #(nop) COPY file:f23f255c68e5e5ac…   4.67MB 
```
Some file was copied into the image and I suppose it was the entrypoint: `mysticeti`.   
Let's get the image filesystem:
```bash
$ docker run containous/jobs
Mysticeti, where are you ? :'(
$ docker ps -aq -l
da1808a9ab1a
$ docker export da1808a9ab1a > containous-jobs.tar
$ mkdir containous-jobs
$ tar -xvf containous-jobs.tar -C containous-jobs
.dockerenv
dev/
dev/console
dev/pts/
dev/shm/
etc/
etc/hostname
etc/hosts
etc/mtab
etc/resolv.conf
mysticeti
proc/
sys/
```
Well, indeed, `mysticeti` is there.

## Run the binary
Out of curiosity, let's run this binary outside the container:
```bash
$ cd containous-jobs
$ ./mysticeti
I have to tell you something...
Something that nobody should know.
However, everyone could see it.
It's not even hidden. Look at containo.us !
Come back when you know more.
But remember, it's a secret !
```
Wow, it gave a different output, so, `mysticeti` needed a Docker host, that's why it was saying `Mysticeti, where are you ? :'(`.

The same output can be seen if the socket of your Docker's daemon is mounted to the image:
```bash
$ docker run -v /var/run/docker.sock:/var/run/docker.sock containous/jobs
I have to tell you something...
Something that nobody should know.
However, everyone could see it.
It's not even hidden. Look at containo.us !
Come back when you know more.
But remember, it's a secret !
```

## Not hidden content
But what the binary is trying to tell us? It suggests to look at `containo.us`  
Don't try to crack the word `containo.us` or to scroll through the website or to search their main page as raw HTML, you'll find nothing.   
The hidden content is inside the header of HTTP response you get from `https://containo.us`:
```bash
$ curl -v https://containo.us/ 2> verbose
$ grep mysticeti verbose
< x-mysticeti: eC5DRo9sohLoSIudKcwSqYcvdcsvEcHj
```
Well, this header doesn't look like a standard one :)

## Swarm secret
Alright, now we have the hidden content but what we can do with it?    
This part took me a while to understand how to proceed. The key phrase here was `But remember, it's a secret !`.    
When I looked at the logs of my docker daemon, I found a weird error:
```bash
$ sudo journalctl -u docker --no-pager | grep -m1 mysticeti
dockerd[2436]: time="2020-08-12T10:50:35.696344089+02:00" level=error msg="Handler for GET /v1.24/secrets/mysticeti returned error: This node is not a swarm manager. Use \"docker swarm init\" or \"docker swarm join\" to connect this node to swarm and try again."
```

So, someone tries to access a secret named `mysticeti` on my Docker host.   
Let's try to find out who, isn't it our magic binary?
```bash
$ strace -f ./mysticeti 2>&1 | grep '/v1.24/secrets/mysticeti'
write(3, "GET /v1.24/secrets/mysticeti HTT"..., 90) = 90
```
Indeed! Let's follow the docker daemon's instructions and init the swarm:
```bash
$ docker swarm init
Swarm initialized: current node (04j1ymr554ayandkqleykbmxr) is now a manager.
```
And let's create that secret which our binary is looking for:
```bash
$ docker secret create mysticeti - <<<"something"
8xkbbwg7z79of2pbtc2f20q4e
```

## Last mile
Looks like we managed to make some progress but the binary doesn't show anything new:
```bash
$ ./mysticeti
I have to tell you something...
Something that nobody should know.
However, everyone could see it.
It's not even hidden. Look at containo.us !
Come back when you know more.
But remember, it's a secret !
```

Hm, I didn't have many ideas at this step, so, I tried the only utility which could tell me something about what is going on in this binary: `strace`   
And I found some syscalls which didn't appear before:
```bash
$ strace -f ./mysticeti  2>&1 | grep mysticeti
[pid  7230] openat(AT_FDCWD, "/run/secrets/mysticeti", O_RDONLY|O_CLOEXEC <unfinished ...>
```
Yes, that must be it! By the way, note `-f` flag which shows the child processes too.   
Then I created `/run/secrets/mysticeti` and ran the binary again:
```bash
$ sudo touch /run/secrets/mysticeti
$ ./mysticeti
Come on !  is not even a secret !
```
Ha ha, new output! I'm definitely on the right way, let's put the hidden content inside this file:
```bash
$ cat /run/secrets/mysticeti
eC5DRo9sohLoSIudKcwSqYcvdcsvEcHj
```

# Bingo
If it's not the end I may consider to quit :)
```bash
$ ./mysticeti
I'm listening on 8080...
```
Bingo!  
The binary started a http server which has the application from. To access the form - type `http://localhost:8080` in your web browser.    
    
The same result can be achieved from the docker container:
```bash
$ cat secret
eC5DRo9sohLoSIudKcwSqYcvdcsvEcHj
$ docker run -v ${PWD}/secret:/run/secrets/mysticeti -v /var/run/docker.sock:/var/run/docker.sock containous/jobs
I'm listening on 8080...
```

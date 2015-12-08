+++
date = "2015-12-07T08:36:54-07:00"
draft = false
title = "Integration testing in Go using docker"
slug = "integration_testing"
tags = ["golang", "docker", "testing"]

+++

*Note: this post was originally written for the [Go Advent 2015](https://blog.gopheracademy.com/advent-2015/introduction/) series, but I discovered that a post with almost exactly the same subject (and even similar code!) already planned :) That's amazing.*

Golang is often used for writing microservices and various backends. Often these type of software do some computation, read/write data on external storage and expose it's API via http handlers. All this functionality is remarkably easy to implement in Go and, especially if you're creating [12factor](http://12factor.net)-compatible app, Go is your friend here.

This functionality is also easy to test using built-in Go testing tooling. But here's the catch - unit testing or *small tests* doesn't guarantee that your service is working correctly. Even if you simply want to test your HTTP response codes, you have to inject dependencies first and connect your code to the external resources or storage. At this point you'll probably realize you need to write a proper integration test, which include not only your code but all dependent resources as well.

But, how to do this without inventing your own scripts and harness code for mocking and starting services? How to make it as easy to use as a normal 'go test' workflow? How to deal with setting up migrations and schemas for you databases? Finally, how to make it cross-platform, so you can easily run those tests on your Macbook as well as in your CI node?

Let me show one of the possible solutions I use for a number of services for quite a long time. It leverages the power of [Docker](https://www.docker.com) isolation and comfort of go test tooling, and thus very easy to use and, with little efforts, gives you truly cross-platform integration testing.

As an example I'll take simple go-based webservice, which is often may be sufficient for REST-backends:

 - REST-service based on [gin](https://github.com/gin-gonic/gin) framework
 - data storage - external MySQL database
 - [goose](https://bitbucket.org/liamstask/goose/) tool for migrations

## Docker

So, yes, we will use [Docker](https://www.docker.com) to handle all external dependencies (MySQL database in our case), and that's exactly the case where Docker shines. Nowadays internet is [full](http://ctankersley.com/2014/09/30/docker-a-misunderstood-tool/) of [articles](http://www.rkn.io/2014/09/26/no-silver-bullets/) and [talks](https://speakerdeck.com/rjschwei/docker-not-a-silver-bullet) telling that Docker is not a 'silver bullet', and [putting](https://valdhaus.co/writings/docker-misconceptions/) a [lot of criticism](http://sirupsen.com/production-docker/) on many docker use cases. Of course, they're absolutely right and many of their points are valid, but in this particular case it's exactly the case where you should use Docker. It gives us everything we need - repeatability, isolation, speed, and portability.

Let's start by creating [Dockerfile](http://docs.docker.com/engine/reference/builder/) for our dependency service - MySQL database. Normally you would use official mysql docker image, but we have to wind up migrations with goose, so we'd better off creating our custom MySQL debian image:
<pre><code class="dockerfile">FROM debian

ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update
RUN apt-get install -y mysql-server

RUN sed -i -e«s/^bind-address\s*=\s*127.0.0.1/bind-address = 0.0.0.0/» /etc/mysql/my.cnf

RUN apt-get install -y golang git ca-certificates gcc
ENV GOPATH /root
RUN go get bitbucket.org/liamstask/goose/cmd/goose

ADD. /db
RUN \
service mysql start && \
sleep 10 && \
while true; do mysql -e «SELECT 1» &> /dev/null; [ $? -eq 0 ] && break; echo -n "."; sleep 1; done && \
mysql -e «GRANT ALL ON *.* to 'root'@'%'; FLUSH PRIVILEGES;» && \
mysql -e «CREATE DATABASE mydb DEFAULT COLLATE utf8_general_ci;» && \
/root/bin/goose -env=production up && \
service mysql stop

EXPOSE 3306
CMD [«mysqld_safe»]</code></pre>
 
Then we build our image with `docker build -t mydb_test .` command and run it with `docker run -p 3306:3306 mydb_test`. The resulting container will have a fresh actual database instance with the latest migrations applied. Once the image is built it takes less than a second to start this container.

The actual name of container and database is not important here, so we use `mydb` and `mydb_test` - simply a convention.

## Go tests

Now, it's time to write some Go code. Remember, we want our test to be portable and issued with `go test` command only. Let's start our service_test.go:
<pre><code class="go">// +build integration

package main

import (
    "testing"
)</code></pre>
  
We place build tag `integration` here to make sure this test will run only when explicitly asked with `--tags=integration` flag. Yes, the test itself is fast, but still requires an external tool (docker), so we'd better separate integration tests and unit tests.

By the way, we could protect in with [testing.Short](https://golang.org/pkg/testing/#Short) flag, but the behavior is opposite in this case - long tests run by default.

<pre><code class="go">
if testing.Short() {
        t.Skip("skipping test in short mode.")
}</code></pre>

### Running docker container

Before running our tests, we need to start our dependencies. There are a few packages to work with [Docker Remote API](https://docs.docker.com/engine/reference/api/docker_remote_api/) for Go, I will use the [one from fsouza](http://github.com/fsouza/go-dockerclient), which I successfully using for quite a long time. Install it with:
   
<pre><code class="sh">go get -u github.com/fsouza/go-dockerclient</code></pre>

 To start the container, we have to write following code:

<pre><code class="go">client, err := docker.NewClientFromEnv()
if err != nil {
    t.Fatalf("Cannot connect to Docker daemon: %s", err)
}
c, err := client.CreateContainer(createOptions("mydb_test"))
if err != nil {
    t.Fatalf("Cannot create Docker container: %s", err)
}
defer func() {
    if err := client.RemoveContainer(docker.RemoveContainerOptions{
        ID:    c.ID,
        Force: true,
    }); err != nil {
        t.Fatalf("cannot remove container: %s", err)
    }
}()

err = client.StartContainer(c.ID, &docker.HostConfig{})
if err != nil {
    t.Fatalf("Cannot start Docker container: %s", err)
}</code></pre>
 
createOptions() is a helper function returning struct with container creating options. We pass our docker container name to that function.

<pre><code class="go">func сreateOptions(dbname string) docker.CreateContainerOptions {
    ports := make(map[docker.Port]struct{})
    ports["3306"] = struct{}{}
    opts := docker.CreateContainerOptions{
        Config: &docker.Config{
            Image:        dbname,
            ExposedPorts: ports,
        },
    }

    return opts
}</code></pre>
 
After that we need to write code which will wait for DB to start, extract IP address for connection, form DSN for database/sql driver and open the actual connection:
<pre><code class="go">// wait for container to wake up
if err := waitStarted(client, c.ID, 5*time.Second); err != nil {
    t.Fatalf("Couldn't reach MySQL server for testing, aborting.")
}
c, err = client.InspectContainer(c.ID)
if err != nil {
    t.Fatalf("Couldn't inspect container: %s", err)
}

// determine IP address for MySQL
ip = strings.TrimSpace(c.NetworkSettings.IPAddress)

// wait MySQL to wake up
if err := waitReachable(ip+":3306", 5*time.Second); err != nil {
    t.Fatalf("Couldn't reach MySQL server for testing, aborting.")
}</code></pre>
Here we wait for two actions to happen: first is to get network inside container up, so we can obtain it's IP address, and second, is MySQL service being actually started. Waiting functions are a bit tricky, so here they are:
<pre><code class="go">// waitReachable waits for hostport to became reachable for the maxWait time.
func waitReachable(hostport string, maxWait time.Duration) error {
	done := time.Now().Add(maxWait)
	for time.Now().Before(done) {
		c, err := net.Dial("tcp", hostport)
		if err == nil {
			c.Close()
			return nil
		}
		time.Sleep(100 * time.Millisecond)
	}
	return fmt.Errorf("cannot connect %v for %v", hostport, maxWait)
}

// waitStarted waits for a container to start for the maxWait time.
func waitStarted(client *docker.Client, id string, maxWait time.Duration) error {
	done := time.Now().Add(maxWait)
	for time.Now().Before(done) {
		c, err := client.InspectContainer(id)
		if err != nil {
			break
		}
		if c.State.Running {
			return nil
		}
		time.Sleep(100 * time.Millisecond)
	}
	return fmt.Errorf("cannot start container %s for %v", id, maxWait)
}</code></pre>

Basically, it's enough to work with our container, but here is another issue comes in - if you run MacOS X or Windows, you use Docker via the proxy virtual machine with tiny linux, `docker-machine` (or its predecessor, `boot2docker`). It means you should use docker-machine's IP address and not real container IP, which is not exposed outside of the docker-host linux VM.
    
### Tuning for portability

Again, let's just write code to accomplish that, as it's quite trivial:

<pre><code class="go">// DockerMachineIP returns IP of docker-machine or boot2docker VM instance.
//
// If docker-machine or boot2docker is running and has IP, it will be used to
// connect to dockerized services (MySQL, etc).
//
// Basically, it adds support for MacOS X and Windows.
func DockerMachineIP() string {
	// Docker-machine is a modern solution for docker in MacOS X.
	// Try to detect it, with fallback to boot2docker
	var dockerMachine bool
	machine := os.Getenv("DOCKER_MACHINE_NAME")
	if machine != "" {
		dockerMachine = true
	}

	var buf bytes.Buffer

	var cmd *exec.Cmd
	if dockerMachine {
		cmd = exec.Command("docker-machine", "ip", machine)
	} else {
		cmd = exec.Command("boot2docker", "ip")
	}
	cmd.Stdout = &buf

	if err := cmd.Run(); err != nil {
		// ignore error, as it's perfectly OK on Linux
		return ""
	}

	return buf.String()
}</code></pre>
    
For working with docker-machine we will also need to pass port forwarding configuration in CreateContainerOptions.

At this point, the amount of supporting code becomes quite notable, and it's better to move all docker related code into separate a subpackage, perhaps in internal/ directory. Let's name it `internal/dockertest`. The source of this package can be [found here](http://pastebin.com/faUUN0M1).

### Running from tests

Now, all we need is to import our `internal/dockertest` subpackage and start MySQL with a single line:

<pre><code class="go">// start db in docker container
dsn, deferFn, err := dockertest.StartMysql()
if err != nil {
	t.Fatalf("cannot start mysql in container for testing: %s", err)
}
defer deferFn()</code></pre>

Pass `dsn` to sql.Open() or your own service init function, and your code will connect to the database inside the container.
Note, that StartMysql() returns also a defer function, which will properly stop and remove container. Our test code knows nothing about underlying mechanisms. It just works as if it was a normal MySQL resource.

### Testing http endpoints

Next step is to test http-endpoints. We may want to test response codes, proper error messages, expected headers or data format and so on. And, following our desire to not depend on any external testing scripts, we want to run all the tests within the Go code. And Go allows us to do so using net/http/httptest package.

Honestly, `httptest` was one of the most surprising things in Go, when I first saw it. net/http design was quite unusual and elegant for me, but httptest looked like a killer feature for testing http services. It leverages the power of interfaces in Go, and particularly, the http.ResponseWriter interface to achieve in-memory round-trip of http requests. We don't need to ask OS to open ports, deal with permissions and busy ports - it's all in memory.

And as soon as gin framework implements http.Handler interface, which looks like this:

<pre><code class="go">type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}</code></pre>
    
we can use it transparently with httptest. I will also use amazing GoConvey testing framework, which implements behaviour-driven testing for Go, and fully compatible with the default `go test` workflow.

<pre><code class="go">func NewServer(db *sql.DB) *gin.Engine {
	r := gin.Default()
	r.Use(cors.Middleware(cors.Options{}))
	// more middlewares ...

	// Health check
	r.GET("/ping", ping)

	// CRUD resources
	usersRes := &UsersResource{db: db}

	// Define routes
	api := r.Group("/api")
	{
		v1 := api.Group("/v1")
		{
			rest.CRUD(v1, "/users", usersRes)
		}
	}

	return r
}
...
r := NewServer(db)
Convey("Users endpoints should respond correctly", t, func() {
	Convey("User should return empty list", func() {
		// it's safe to ignore error here, because we're manually entering URL
		req, _ := http.NewRequest("GET", "http://localhost/api/v1/users", nil)
		w := httptest.NewRecorder()
		r.ServeHTTP(w, req)

		So(w.Code, ShouldEqual, http.StatusOK)
		body := strings.TrimSpace(w.Body.String())
		So(body, ShouldEqual, "[]")
	})
})</code></pre>

GoConvey has also an astonishing web UI, I guarantee you will start writing more tests just to see that nice blinking "PASS" message! :)

And now, after you get the idea, we can add more tests for testing basic CRUD functionality for our simple service:

<pre><code class="go">Convey("Create should return ID of a newly created user", func() {
	user := &User{Name: "Test user"}
	data, err := json.Marshal(user)
	So(err, ShouldBeNil)
	buf := bytes.NewBuffer(data)
	req, err := http.NewRequest("POST", "http://localhost/api/v1/users", buf)
	So(err, ShouldBeNil)
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)

	So(w.Code, ShouldEqual, http.StatusOK)
	body := strings.TrimSpace(w.Body.String())
	So(body, ShouldEqual, "1")
})
Convey("List should return one user with name 'Test user'", func() {
	req, _ := http.NewRequest("GET", "http://localhost/api/v1/users", nil)
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)

	So(w.Code, ShouldEqual, http.StatusOK)
	body := w.Body.Bytes()
	var users []*User
	err := json.Unmarshal(body, &users)
	So(err, ShouldBeNil)
	user := &User{
		ID: 1,
		Name: "Test user",
	}
	So(len(users), ShouldEqual, 1)
	So(users[0], ShouldResemble, user)
})</code></pre>

# Conclusion

As you may see, Go not only make testing a lot easiers but also make use of BDD and TDD methodologies very easy to follow and opens new possibilities for cross-platform integration- and acceptance- testing.

This example provided here is simplified on purpose, but it's based on the real production code which is being tested in this way for more than 1.5 years and survived a number of refactorings and migrations' updates. On my Macbook Air, the whole test, from start to end (compile code, run docker container in docker-machine and test ~35 http requests, shut down the container) it takes about 3 seconds. On native Linux system it's obviously a lot faster.

One may ask why not publish this code as a separate library, and make the whole task (and article) even shorter. But the point here is that for every different service there may be a different set of service connections, different usage patterns and so on. And what is really important is that with Go it's so easy to write this harness code for your needs, that you don't have an excuse not to do this. Whether you need many similar containers in parallel (probably, you'll need to randomize exposed ports), or you have to interconnect some services before starting them - you just write in Go, hiding all the complexity from the actual testing code.

And always write tests! There is not excuse not to write them anymore.

UPD: After writing the article, discovered the package [dockertest](https://github.com/ory-am/dockertest) by Aeneas Rekkas ([@_aeneasr](https://twitter.com/_aeneasr)), which does almost exactly the same as a code in this article, and looks pretty solid. Don't miss it out!

+++
Tags = ["golang"]
date = "2016-06-01T19:28:46+02:00"
title = "go get for private repos in docker"
slug = "go_get_private"

+++

As Go community slowly moving towards established and well understood patterns and practices of dependency management, there are still some confusing moments. One of them is automating repeatable build process using containers along with using dependencies in private repositories. 

Private repositories on Github are often is a [source of confusion](https://github.com/golang/go/issues/9697) when using `go get`, but it has easy workaround by adding two lines to your `.gitconfig`:

```
[url "git@github.com:"]
	insteadOf = https://github.com/
```
or as a oneliner:

```
git config --global url."git@github.com:".insteadOf "https://github.com/"
```

But the most confusing part is trying to make the whole build process work inside the container. I will use Docker as an example, as it's most popular container at the moment.

## The problem
Imagine, you have two packages `github.com/company/foo` and `github.com/company/bar`, where `foo` imports `bar`:

```
foo.go:
import "github.com/company/bar"
```

In normal workflow, you setup GOPATH, you SSH keys, gitconfig and you're done - simple `go get github.com/company/foo` will work and download both packages:

```
$ go get -v github.com/company/foo
github.com/company/foo (download)
github.com/company/bar (download)
```

But now, you want to make the build process reproducible on any machine, even on CI instance, so you pack everything in Docker container. You will probably use simple Dockerfile based on official `golang`:

**Dockerfile**:

```
FROM golang:1.6

ADD . /go/src/github.com/company/foo
CMD cd /go/src/github.com/company/foo; go get github.com/company/bar && go build -o /go/bin/foo
```

**Build script**

```
docker build -t foo-build . 				# build image
docker run --name=foo-build foo-build		# compile binary
docker cp foo-build:/foo foo				# copy binary to fs
docker rm -f foo-build						# remove container
docker rmi -f foo-build						# remove image
```

This setup **will not** work because Docker container used for building (foo-build) doesn't container `bar` dependency, SSH keys and proper gitconfig. And, apparently, it's not trivial simply to add the keys - you have to deal with a bunch of obstacles, mainly on the SSH side. So, let's go through quickly and setup working solution.

## The solution
### ssh vs https
First of all, on the building stage (`docker run ...`) you will encounter the following error message:

```
# cd .; git clone https://github.com/company/bar /go/src/github.com/company/bar
Cloning into '/go/src/github.com/company/bar'...
fatal: could not read Username for 'https://github.com': No such device or address
package github.com/company/bar: exit status 128
```
What it does mean, is that your access to github is granted using SSH keys, but `git` command, which is invoked by `go get`, is trying to clone repository using HTTPS form and you don't have credentials set up.

Workaround for this is easy, and described in the beginning of this post, so we just have to add this to our Dockerfile right before calling `go get`:

```
RUN echo "[url \"git@github.com:\"]\n\tinsteadOf = https://github.com/" >> /root/.gitconfig
```

### Keys
The next error you'll see is host key verification error:

```
# cd .; git clone https://github.com/company/bar /go/src/github.com/company/bar
Cloning into '/go/src/github.com/company/bar'...
Host key verification failed.
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
package github.com/company/bar: exit status 128
```
Well, it's simply because our Docker container doesn't have SSH keys yet. And the right approach is not trivial, so let's go through it.

First of all, we want every developer or CI to use it's own keys for accessing private repo. If person has access to `foo`, she's definitely has an access to `bar` and the keys are usually in `~/.ssh/id_rsa` file.

**You can't copy keys into container, though.** Dockerfile's ADD and COPY commands can copy files from the current directory only, so you can't just add `ADD ~/.ssh/ /root/ssh` to your Dockerfile. One of the solution is to write wrapper script that will copy private key to local directory and then to the container, but it's still not very safe and elegant solution.

What we can do, is to mount volume using docker's `-v` command line flag. The first approach will be probably to mount the whole `~/.ssh` directory, but it's tricky

`docker run --name=foo-build -v ~/.ssh:/root/.ssh foo-build`

This command will work as expected on MacOS X (using latest Docker Beta, at least), but not in Linux box. The reason is the files ownership for `~/.ssh/config` file. The `ssh` (which is invoked by `git`, which is invoked by `go get`) expects this file to have the same user ownership as a running user. Inside the container the user is `root`, but the mounted directory most probably has ovnership of your normal Linux user, say, `developer` and inside the container it looks like:

```
$ ls ~/.ssh/config
-rw-r--r--  1 1000  1000  147 Jun  1 19:20 /root/.ssh/config
```
making SSH to complain and abort:

```
Bad owner or permissions on ~/.ssh/config
```

The solution is to mount only the key and workaround host checking later.

```
docker run --name=foo-build -v ~/.ssh/id_rsa:/root/.ssh/id_rsa foo-build
```

Error will be the same, though, but rerunning it with `-t` option, we'll see the reason:

```
$ docker run --name=foo-build -v ~/.ssh/id_rsa:/root/.ssh/id_rsa -t foo-build
The authenticity of host 'github.com (192.30.252.128)' can't be established.
RSA key fingerprint is 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.
Are you sure you want to continue connecting (yes/no)?
```

Of course, we don't want to interact manually with ssh prompt, so we have to find a way to force it. There is an SSH client option for that, called StrictHostChecking.

### StrictHostChecking
Typically, you have file named `~/.ssh/known_hosts`, which holds information about, well, known hosts. But in our container, there is no such file, so we have to use the client option to supress those checks. The easiest way to do this is to put this option into the `~/.ssh/config` file - yes, the one we had ownership problems with.

But, we only need one option, so it's ok to create this file on the fly inside the container. Add to Dockerfile:

```
RUN mkdir /root/.ssh && echo "StrictHostKeyChecking no " > /root/.ssh/config
```

Rerun the ```docker run``` step and you'll finally have success!

### Conclusion

The final Dockerfile:

```
FROM golang:1.6

RUN echo "[url \"git@github.com:\"]\n\tinsteadOf = https://github.com/" >> /root/.gitconfig
RUN mkdir /root/.ssh && echo "StrictHostKeyChecking no " > /root/.ssh/config
ADD .  /go/src/github.com/company/foo
CMD cd /go/src/github.com/company/foo && go get github.com/company/bar && go build -o /foo
```

and the build steps:

```
docker build -t foo-build .
docker run --name=foo-build -v ~/.ssh/id_rsa:/root/.ssh/id_rsa foo-build
docker cp foo-build:/foo foo
docker rm -f foo-build
docker rmi -f foo-build	
```

You may put those steps to the Makefile or custom build script, and can safely use it locally or in CI or whatever.

Private SSH key is copied once into the temporary container, used for building, which is removed immediately. Nice and safe solution.

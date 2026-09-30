# 1. Getting set up
This guide assumes you have a repo previously created without an existing Rails infrastructure. These steps should slide between steps #2 and #3 of the in-class assignment from 9/30/26.

Do the following: 
1 Clone the existing repo
2 Create a Dockerfile (yes, that's the name of the file with no file extension) which contains the following. The Dockerfile should be saved in the top-level directory of the repo.
```
FROM ruby:3.3.8

RUN gem install rails -v '<8.0'
RUN gem install bundler

WORKDIR /app

# Default to the interactive bash shell
CMD ["/bin/bash"]
```
# 2. Start Docker Desktop
* Ensure that Docker Desktop is running on your local machine. Then do the following commands to build your container and run it on your (local) machine. For more on Docker containers, 
see [this reference](https://www.geeksforgeeks.org/devops/introduction-to-docker/). The way we're running the container will
  *  permit us shell access to the container (`-it` to be able to execute commands in the container),
  *  make the server, once running, accessible to the local laptop (`-p 3000:3000`), and
  *  mount the local working directory in the container, so that local file changes (such as the result of running a command) will be saved outside of the container in the local
filesystem (`-v "$(pwd)":/app`)

```bash
docker build -t ourrailsapp:latest .
docker run -v "$(pwd)":/app -p 3000:3000 -it ourrailsapp:latest
```
Once you start the container, you should see a command prompt such as:
```bash
root@aed39cf54eae:/app#
```

You can now use this shell into the container to do the rest of the work, such as:

* Create the rails framework for your app (there's a specific command to be run to do so)
* Create your resources (also a rails command)
* List the routes for your application (also a rails command)
* Install the required gem files into the container. From the container's shell, run `bundle install`.
* Start the server, but be sure to bind it to the IP address of 0.0.0.0, otherwise you will not "see" it from your local machine.

Once you start the server and use a broswer to test the server (go visit `https://localhost:3000`), you should see an error. No investigate that, and resolve the issue with
another rails command. You likely will need to stop the server (Ctrl+C), issue a rails command in the Docker container's shell to fix the issue, and the restart the server
again with the same command you used earlier (above).

When you are done, stop the server (see above) and terminate the Docker shell (CTRL+D).  You can then close down Docker Desktop.

# Jenkins

https://www.linkedin.com/learning/devops-foundations-your-first-project/

## Installation on Docker

Fortunately, installing Jenkins on Docker is pretty straightforward, it didn't used to be, but fortunately today it is. We are going to start with the Docker file and we're going to name this `jenkins.dockerfile`. So, like all Docker images, we start with a FROM line, and a MAINTAINER line. FROM line is the line that we use to source the parent image for this Docker image. Docker images are a layered file system, so they build on top of each other and the union of them is what comprises the final image. So you need to start from somewhere, and normally this would be an image that you can find on the Docker hub, or it can be from scratch, if we were starting completely from scratch. You can think of this like a clean Linux installation. Remember, Docker virtualizes the operating system host kernel, not the actual hardware, which is what virtual machines do. But we're not going to start from scratch 'cause that's a lot of work. Instead, what we're going to do is we're going to start from a preexisting image on the Docker hub.

Instead of going to the internet browser we could just:

```
$ docker search jenkins
NAME                                   DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
jenkins                                Official Jenkins Docker image                   4645                [OK]                
jenkins/jenkins                        The leading open source automation server       1906
```

The first result is the official image from Jenkins but it is deprecated now in favor of `jenkins/jenkins:lts`. There are multiple tags (at least on the web page for that image) and let's pick Alpine because its small and fast Linux distro. `FROM jenkins/jenkins:alpine`

One of the strengths that Jenkins brings with it is its exhaustive network of plugins. The disadvantage with this though is that it doesn't come with any plugins out of the box, which can make using Jenkins from the first instance quite difficult, so what we need to do here is we need to tell this Docker image how to install plugins. Fortunately, this Docker image has a really convenient way of doing that, and the way that it does that is by copying a file called plugins.txt into this directory: `COPY plugins.txt /usr/share/jenkins/plugins.txt`. Inside of the Jenkins Docker image, it looks for this file, and then if it finds it, it installs the plugins based on what is in that file.

If you've never used Jenkins before, this is a pretty daunting place to start because your first question is probably what plugins do I need? Well, fortunately, there is a repository for Jenkins plugins and that repository is located at [plugins.jenkins.io](https://plugins.jenkins.io/).

These are the plugins that we are going to use:
- **workflow-aggregator**: jenkins pipelines as code
- **seed**: when you create a Jenkins file, you need to have a job that looks at all of the projects, all of the freestyle projects that exist, and finds Jenkins files within those projects and imports them. That job is called the seed job, and the seed plugin enables you to do that.
- git

`plugins.txt`:

```
workflow-aggregator
seed
git
```

Next, we need to install these plugins and Jenkins image comes with `install-plugins.sh` script pre-installed, you don't have to worry about it, just provide the `plugins.txt` file in that location and you are good to go: `RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/plugins.txt`.

The next command that we're going to write in our Docker file is a documentation command called EXPOSE. And EXPOSE doesn't do anything except let any maintainers or authors, or anyone that's going to modify this Jenkins file know what port they need to expose in order for this service to be made available. So, Jenkins by default, runs on port 8080, and so I would like people using this image to know to expose 8080 when they run this Docker container: `EXPOSE 8080`.

`jenkins.Dockerfile`:

```Dockerfile
FROM jenkins/jenkins:alpine
MAINTAINER Sigman <sigman@sigman.pl>

COPY plugins.txt /usr/share/jenkins/plugins.txt
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/plugins.txt

EXPOSE 8080
```

Now we will create the `docker-compose.yml` file.

What we're going to do is we are going to mount some volumes. The way that Jenkins works, at least this Docker image works, is that it creates a directory called /var/jenkinshome, and in that lives all of the metadata about how the Jenkins server is set up, how it runs, how and also all of the jobs that it's going to be launching, so we're going to create that volume out here, and for the purposes of this project, we're going to mount that into our directory underneath Jenkins Home. So, to the left is the directory you want to use on our host. To the right, is the directory within the container, so var/jenkins_home: `$PWD/jenkins_home:/var/jenkins_home`.

The second volume that we are going to mount is our current directory inside of a folder called /app, that way our project can be made available to this Jenkins Docker container: `$PWD:/app`. If we were using something like GitHub or GitLab, or Bitbucket, or any other collaborative SAS based git product, we probably wouldn't need to include this line, because our project would be in git. And, we could just, when we spin up Jenkins, have it clone our project from GitHub and get on its way. However, because we're not using git for this example, we can just mount the current directory and pull in the project as if it were on the file system.

The next thing that we're going to do is we're going to expose the port, and so to do that, we're going to use the port argument here. And, we can mount 8080 to 8080. Now, the number to the left is the port on our Mac that we want to access our container from, and the port on the right is the port that's being served from the container, so like we said in our Docker file, we expose port 8080.

`docker-compose.yml`:

```yml
version: '3.7'
services:
  jenkins:
    build:
      context: .
      dockerfile: jenkins.Dockerfile
    volumes:
      - $PWD/jenkins_home:/var/jenkins_home
      - $PWD:/app
    ports:
      - 8182:8080
```

With this we can start jenkins with: `$ docker-compose up jenkins`.

Jenkins will install all the plugins but at the end it will also display the password to log in to: `localhost:8080`. The second place you can find it is in your Jenkins home, underneath `secrets/initialAdminPassword`. Now, if you're working on this project in a git repository, I would highly recommend not committing your Jenkins home. There's a lot of data that goes in there, including your own project when you start playing around with this, so I would recommend putting it in some other place, such as the temp directory on your Mac or Linux machine, just because, especially this initial password here, if you never change it, then anyone that runs this could know what your Jenkins password is, and that could be bad depending on how you decide to play with Jenkins in the future. So, once again I recommend do not committing your Jenkins home if you're working in a git repository.

`.gitignore`:

```.gitignore
jenkins_home/
```

## Jenkinsfile

There are two types of Jenkins pipelines that the pipeline plugin within Jenkins supports. There is the declarative pipeline and the scripted pipeline. The main differences between the two come down to syntax and flexibility. Scripted pipelines adhere more closely to Groovy syntax, which yields a lot of flexibility for people that already know Groovy. But it comes at the expense of readability over time. I've found that declarative pipelines are, while newer, more readable and easier to work with especially if this is the first time that you're using Jenkins. There's also more documentation on declarative pipelines.

Structure for `Jenkinsfile`:

```groovy
pipeline {
  agent any
  stages {
    stage('Build website') {
      steps {
        // for debugging
        sh "echo here is pwd: $PWD"
        sh "ls -lah"
        sh "ls -lah scripts/"

        sh "scripts/build.sh"
      }
    }
    stage('Run tests') {
      steps {
        sh "scripts/tests.sh"
      }
    }
    stage('Deploy website') {
      steps {
        sh "scripts/deploy.sh"
      }
    }
  }
}
```

- `pipeline`: Everything that you need to do within your pipeline happens within the pipeline block. You can have code outside of it for things like global variables or some configuration data that you need from Jenkins, but really all of the work that's being done is happening in here.
- `agent any`: The agent clause tells Jenkins what kind of Jenkins slaves to build this job on. So, there are two types of nodes in Jenkins. There are Jenkins masters and Jenkins slaves. When I log on to Jenkins, typically, you're logging into a Jenkins master. Jenkins slaves are the actual nodes that do the work or run the builds, and the master can also be a slave. In our demo, the master is going to be a slave because we only have one Docker container for Jenkins. But in your normal enterprise scenario, you might see one or more masters and several slaves for different types of things, which is where this agent-any comes in. So agent-any tells Jenkins that it can use any agent on any slave to build this job. However, if you have specific needs, such as if you wanted to run Docker within this job, or you needed a Windows slave to run some Windows builds or things like that, you might want to change agent-any to a different identifier that supports that.
- `stages` define the actual stages that need to run. And these run serially by default, so they run step-by-step. You can configure it to run in parallel. I would consult the documentation to learn more about that. But for this example, we're just going to use a serial pipeline. So normally when it comes to CI pipelines, there are three stages.
- `steps` can be anything. I typically write my steps to be just scripts, what this would do is when the Jenkins slave runs this, it's going to literally invoke a shell and run this script, which can be nice because you can have your script run locally on your laptop and also in CI, and they both run the same.

We don't want to use `sh "$PWD/scripts/build.sh"` in steps as the job will fail. Now PWD stands for present working directory, or current working directory. However, because when this job clones my repository into its workspace it's going to be set to the workspaces directory, which means when it clones my app directory, which you can see here, it's basically going to cd, or change directories, into my app directory here. So I don't actually need to specify the present working directory, because I'm already going to be inside of it.

Our demo scripts are simple:

```sh
echo "I am running: $0"
```

Remember to set right permissions to shell scripts on Linux to let Jenkins correctly run them.

More info: [https://jenkins.io/doc/book/pipeline/syntax/](https://jenkins.io/doc/book/pipeline/syntax/)

In Jenkins UI we will create new project, select "Multibranch Pipeline" and as "Branch Sources" choose "Git". For "Project Repository" we will pick type `file:///app` as we have mounted this dir as a volume to our dir with git repo on the host. Normally we would put a URL to github repo like `https://github.com/wsierakowski/someproject` and configure webhooks so that updates on github would trigger jenkins build. No credentials etc, we will leave everything else as default. "Build Configuration" Script Path `Jenkinsfile` is fine.

Every commit to the local git will make Jenkins start the build after we click "Scan Multibranch Pipeline Now".

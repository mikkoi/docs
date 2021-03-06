---
title: IronWorker Getting Started 3-Step Docker Workflow
layout: default
section: worker
---

<img style= "display: block; width: 250px; margin: 0 auto;"  src="/images/docker_logo.png" style=""/>

<p class="subtitle">Offload your tasks to the parallel-processing power of the elastic cloud. Write your code, then queue tasks against it&mdash;no servers to manage, no scaling to worry about.</p>

<div class="flow-steps">
    <div class="step">
        <a class="number" href="#write">1</a>
        <a class="title" href="#write">Write and Test your Worker</a>
    </div>
        <i class="icon-long-arrow-right icon-2x"></i>

    <div class="step">
        <a class="number" href="#upload">2</a>
        <a class="title" href="#queue">Package and Upload your Worker</a>
    </div>
        <i class="icon-long-arrow-right icon-2x"></i>

    <div class="step last">
        <a class="number" href="#queue">3</a>
        <a class="title" href="#inspect">Queue Tasks for your Worker</a>
    </div>
</div>

## Setup

Before starting, you'll need to setup a couple of things. You only need to do this once.

1. [Install the CLI tool](/worker/cli/)
1. [Setup your Iron.io credentials](/worker/reference/configuration/)
1. [Install Docker](https://docs.docker.com/installation/#installation)

## Hello World Worker

This is a very simple hello world example worker in Ruby. You don't even need Ruby installed to try this example so give it a go!
All languages follow the same process so you'll get an idea of how things work regardless.

Create a file called `helloworld.rb` containing:

{% highlight ruby %}
puts "Hello World!"
{% endhighlight %}

Now let's run it in one of the Iron stack containers:

{% highlight bash %}
docker run --rm -v "$(pwd)":/worker -w /worker iron/ruby:2.2 sh -c 'ruby helloworld.rb'
{% endhighlight %}

The fact that it runs means it's all good to run on IronWorker, so lets upload it and queue up a task for it so it runs on
the IronWorker platform.

Let's package it up inside a Docker image. You should have account on <a href="https://hub.docker.com/" target="_blank">Docker Hub</a> for this.
Copy the <a href="https://github.com/iron-io/dockerworker/blob/master/ruby/Dockerfile" target="_blank">Dockerfile</a> from our repository and modify the ENTRYPOINT line to run your script. Build your docker image:

{% highlight bash %}
ENTRYPOINT ["ruby", "helloworld.rb"]
{% endhighlight %}

Build a docker image:
{% highlight bash %}
docker build -t USERNAME/hello:0.0.1 .
{% endhighlight %}

The `0.0.1` is the version which you can update whenever you make changes to your code.  
`USERNAME` is your username on Docker Hub.
Test your image, just to be sure you created it correctly:

{% highlight bash %}
docker run --rm -it USERNAME/hello:0.0.1
{% endhighlight %}

Push it to Docker Hub and register your image with Iron:

{% highlight bash %}
docker push USERNAME/hello:0.0.1
iron register USERNAME/hello:0.0.1
{% endhighlight %}

And finally queue up a job for it!

{% highlight bash %}
iron worker queue --wait USERNAME/hello
{% endhighlight %}

The `--wait` parameter waits for the job to finish, then prints the output.
You will also see a link to [HUD](http://hud.iron.io) where you can see all the rest of the task details along with the log output.

That's it, you've ran a worker on the IronWorker cloud platform!

Now let's get into more detail.

<h2 id="write">1. Write and Test your Worker</h2>

IronWorker's <a href="/worker/reference/environment">environment</a> is a Linux Docker container that your task is executed in. Anything you write that runs inside of our published <a href="https://hub.docker.com/r/iron" target="_blank">Docker images</a> should run just the same as on IronWorker. The key here is getting it to run with the Docker commands below and sample payloads.

The primary Docker command is:

{% highlight bash %}
docker run --rm -v "$(pwd)":/worker -w /worker IMAGE[:TAG] sh -c 'MY_EXECUTABLE -payload MY_PAYLOAD.json'
{% endhighlight %}

* Replace IMAGE with the name of the image you want your code to be executed in. For example, if your worker is a Ruby script, you can replace IMAGE with `iron/ruby`. Also you may need to specify a TAG (version) of the image you want to use: `iron/ruby:2.2`
* Replace MY\_COMMAND with what you'd like to execute. For instance, if your worker was a Ruby script called `myworker.rb`, you'd
replace MY\_COMMAND with `ruby myworker.rb`. If it was a Go program, you'd change it to `./myworker`.
* Replace MY_PAYLOAD with the name of an example payload file to test with. This payload file is the format that you'll use
to queue up jobs/tasks for your worker after it's uploaded.

This command may seem simple at first glance, but the main thing is that it will force you to vendor all your dependencies
along with your worker. You'll find links to an example repository showing how to do this for various languages.

<h2 id="upload">2. Package your Worker</h2>

Let's package it up inside a Docker image and upload it to a Docker Registry. Copy the Dockerfile from appropriate directory (depending on used programming language) of this [repository](https://github.com/iron-io/dockerworker) and modify the ENTRYPOINT line to run your script. Build your docker image:

{% highlight bash %}
docker build -t USERNAME/IMAGENAME:0.0.1 .
{% endhighlight %}

That's just a standard `docker build` command. The 0.0.1 is the version which you can update
whenever you make changes to your code. Change `USERNAME` to your Docker Hub username and change `IMAGENAME` to your docker image name.

Test your image, just to be sure you created it correctly:

{% highlight bash %}
docker run --rm -it -e "PAYLOAD_FILE=MY_PAYLOAD.json" -e "YOUR_ENV_VAR=ANYTHING" USERNAME/IMAGENAME:0.0.1
{% endhighlight %}

<h2 id="push">3. Push it to Docker Hub</h2>

Push it to Docker Hub:

{% highlight bash %}
docker push USERNAME/IMAGENAME:0.0.1
{% endhighlight %}

<h2 id="register">4. Register your image with Iron</h2>

Ok, we're ready to run this on Iron now, but first we have to let Iron know about the
image you just pushed to Docker Hub. Also, you can optionally register environment variables here that will be passed into your container at runtime.

{% highlight bash %}
iron register -e "YOUR_ENV_VAR=ANYTHING" USERNAME/IMAGENAME:0.0.1
{% endhighlight %}

<h2 id="queue">5. Queue / Schedule jobs for your image</h2>

Now you can start queuing jobs or schedule recurring jobs for your image.

{% highlight bash %}
iron worker queue --payload-file MY_PAYLOAD.json --wait USERNAME/IMAGENAME
{% endhighlight %}

Notice we don't use the image tag when queuing, this is so you can change versions without having to update all your code that's queuing up jobs for the image.

The --wait parameter waits for the job to finish, then prints the output. You will also see a link to HUD where you can see all the rest of the task details along with the log output.

Of course, in practice you'll be [queuing up jobs via the API](/worker/reference/api/#queue_a_task), most likely using one of our [client libraries](/worker/libraries).

<h2 id="private">Private images</h2>

If you want to keep your code private and use a [private Docker repository](https://docs.docker.com/docker-hub/repos/#private-repositories), you just need to let Iron know how to access your private images:

{% highlight bash %}
iron docker login -e YOUR_DOCKER_HUB_EMAIL -u YOUR_DOCKER_HUB_USERNAME -p YOUR_DOCKER_HUB_PASSWORD
{% endhighlight %}

Then just do everything the same as above.

<h2 id="zip_packaging">If you don't want to package your code using Docker</h2>

You can package and send your code to Iron directly with the instructions below. Start with step 1 above, then continue at step 2 here.

<h2 id="upload">2. Package and Upload your Worker</h2>

Packing is pretty straightforward knowing that you got it working with the `docker run` command above, just zip it up.

{% highlight bash %}
zip -r myworker.zip .
{% endhighlight %}

Then upload the zip you just created:

{% highlight bash %}
iron worker upload [--zip myworker.zip] --name myworker DOCKER_IMAGE [COMMAND]
{% endhighlight %}

<h2 id="queue">3. Queue Tasks for your Worker</h2>

Now you get to queue up tasks/jobs for your Worker!

{% highlight bash %}
iron worker queue --wait myworker
{% endhighlight %}

Typically you'd use the [IronWorker API](/worker/reference/api/#queue_a_task) to actually queue up tasks from other systems.
The cli queue command above is primarily for testing purposes.


<br/><br/><br/>

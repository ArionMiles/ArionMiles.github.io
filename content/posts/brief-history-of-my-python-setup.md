---
title: A brief history of my Python setup
date: 2021-05-18
type: "post"
description: "My experiences with different python development tools"
tags:
  - python
---

**Note:** This isn’t meant to be a comprehensive article comparing the pros and cons of each tool used here. It’s a simple log of my experience with these tools. I talk about what worked, what didn’t, and why I moved away from certain tools.

## Some background

I started out using python for writing small automation scripts. Then I spent a lot of my time in college writing and maintaining a Telegram bot for my college’s web portal. It used an API wrapper for the Telegram API, an SQLite DB, and a web scraper written using Scrapy. It grew from a small to a midsize project pretty fast.
I’ve also used it for writing Django applications, both personal and work-related. Most recently I’ve used Python for a college project and it was a huge one, using Machine Learning and web libraries.
Lately, I’ve stopped working on big projects which use Python, as I’ve started working with JavaScript. I’m using this opportunity to offer some perspective on 4 years of using python as a student and professional and how my tools evolved.

## First foray into 🐍 land

The first time I touched Python was back in 2015 when I was intrigued by InfoSec stuff and spent some time learning about and looking into keyloggers, RATs, and other such fun apps made in Python (Mr. Robot had just come out, don’t judge me). I was still using Python2 at the time, since a lot of apps of that variety were still written in it, for some reason. I used the plain old MSI installer to get it up and running on my machine. Wasn’t even aware of what functions are, much less care about how dependencies mess up environments.
It wasn’t until two years later in March 2017 when I got back to Python. This is when I started learning more, eventually moved to Py3 after a month or so. I can’t remember if it was some person or some blog/article or some Reddit thread where I learned about the importance of virtual environments. Then I spent a small amount of time learning about them. While writing this blog, I went back to my earlier projects and found my first documented use of virtual environments was in Nov 2017, for my Telegram bot project, and I didn’t start out using the built-in venv module, but [virtualenv](https://pypi.org/project/virtualenv/).

## Environment Isolation Tools

### virtualenv

This is the first tool I used for isolating my development environments. I guess I just wasn’t aware that py3 had a built-in module for virtual environments (`venv`). Or maybe it was because I transitioned from py2 to py3 and might have tried virtualenv in py2 first since it’s compatible across both versions.
This module is a drop-in replacement for the `venv` module in the py3 stdlib and offers more features as well, but I never used them as my needs were very limited.

### venv

I don’t even remember when I started using this. I guess one day I just started using the built-in functionality instead of having to install `virtualenv`, my needs weren’t complex, and both of them satisfied them adequately. I still use this to this day. Every project gets its separate virtual environment, generally, all of them are named `.venv`.

### pipenv

This is touted as the best possible solution for python’s bad tooling. But it has its issues. It offers the lockfile goodness, but pipenv just didn’t sit right with me. The reason I stopped using pipenv was that an upgrade to pip broke it once, and it left a bad taste in my mouth.
Today, pipenv is the official recommendation from Python for dependency and environment management. I might get around to trying it again someday, but not anytime soon.

### docker

The only reason I first started using docker was because a project I was working on used Redis, and since Redis doesn’t exactly work on Windows, I had to use containers. I had a vagrant box with Ubuntu 16.04 which had some version of docker in it.
Containers were fun, unless you were me, back in 2018, on a slow internet connection. Then it sucked waiting for all the different base images to be done downloading, and then waiting for my container image to be built. If I updated my `requirements.txt` file, I’d have to recreate my image and you can’t take advantage of cached pip packages in this scenario, so it’d download all of them all over again. Not to mention, starting up and shutting down the vagrant box took its own sweet time.
I continued to use them for my Django projects since it was easier to deploy them as a docker-compose app than install all dependencies in a fresh Linux server every time. During this time I even changed the setup for my telegram bot project to be deployed using Docker.
Using docker does have the advantage of developing in an environment very close to the production, and managing different environments becomes as simple as having 2 different compose files.

### conda

I first came across conda through a workshop at PyCon. I never really gave it a chance back then, as I was comfortable with venv, and never really understood the importance of it until recently. Conda is the default package manager with Anaconda, which comes with a bunch of data science and machine learning packages pre-installed, so I’d always associate it with those use cases.
That said, I did attempt to use it at one point, but was confused why my conda environment wasn’t listing the packages I installed through pip. I thought maybe something was broken. It wasn’t until later that the distinction between conda and pip became clear to me, how conda doesn’t work together with pip but instead works as a completely independent package manager.
In my recent project, I did make extensive use of conda though and learned how much more it has to offer. It’s important to understand that conda exists outside of python. And so, it can be used for isolating environments for any programming language. The isolation it offers is on a whole different level than what pip or any other tools do. You can isolate system-level dependencies, e.g. the dependencies you install using the distro’s package manager. As a result, conda can be used to manage different versions of pythons as well. Conda environments can be easily exported to `environment.yml` files and then recreated exactly in a different machine.
Of course, this means you must install the packages through conda’s repositories (conda forge), and if they don’t exist there, you’ll have to either create your recipe and put it up there or deal with it. Though chances of not finding your package on there would be rare, so it’s a good bet. If your python packages aren’t available through conda forge, you can specify them in your `environment.yml` file as a [pip entry](https://conda.io/docs/user-guide/tasks/manage-environments.html#create-env-file-manually).

## Python version managers

These tools deserve their own category as they’re not meant to be used instead of the options mentioned above but complement them.

### py launcher

[This tool has been shipping inside Python installers since 2012](https://learning-python.com/py33-windows-launcher.html), and I’d been sleeping on it until a few months ago. When it came to running projects using different versions of python, I’d manually adjust the PATH environment variable all this time on Windows, and now I feel stupid.
It’s only available for Windows, and it does a pretty good job. You can install different versions of Python onto the same machine and then decide which version to use when running your python script. It also understands the python shebang, so you don’t have to be explicit every time you run the file.
You can also set a `PY_PYTHON` environment variable to specify a global version to be used by default.

### pyenv

This immediately became my favorite python tool ever! Python doesn’t get tooling right, so it’s a breath of fresh air when you find something that works like pyenv. It works the same as py launcher, but for Linux. You can set a global python version here as well.
You can also set python versions per project by running `pyenv local <versionNumber>` and a `.python-version` file is created in that directory. Then pyenv will automatically run all files in it using the version number specified in the file.
And it does more! You can install any implementation of python using pyenv. Wanna see how much more performant your code is in `pypy`? You can install a pypy implementation. Wanna run python in a JVM? `Jython` is available too! You can download the latest stable, dev, or release candidate versions of CPython and a lot more! You can even install anaconda or miniconda with specific python versions using pyenv.

## Code Editors & IDEs

I’ve used Notepad++, Sublime Text, and Atom, very briefly however. For the majority of my time writing python, I’ve relied on VSCode and PyCharm.

## PyCharm

I’ll wholeheartedly support PyCharm for Django projects, as its IntelliSense support for Django is better than VSCode’s. It can also interface with databases really well, and that’s a plus when you’re working on a Django project. They’ve also added support for recognizing WSL’s virtual environments.

## VSCode

However, VSCode has been my main editor for a long time now. With its remote extensions for containers and cloud VMs, it’s shaping up to be the ideal editor for small to moderate projects. The extensions marketplace also offers a wide variety of tools to improve your workflow.

## Jupyter Notebooks

I tried Jupyter as I dived into Data Science and found it cool, but only good enough for experimentation. I’ve always been a terminal guy, so my mindset for running python code has been either in the REPL for quick experimentation, or putting it in a file. Also, in the time it takes to launch a notebook and be able to do anything meaningful with it, I’d lose my train of thought. That said, I have used it plenty. Familiarizing myself with the shortcuts was the biggest productivity boost for working with notebooks. My main reason for opening a notebook has always been ML and DS libraries, and I can’t imagine working with `pandas` or `numpy` outside of a notebook.

## Things I haven’t tried out

### poetry

This is something I’m looking forward to trying someday. It’s supposed to be what pipenv should’ve been, or so I’ve heard.

### virtualenvwrapper

As the name suggests, it’s a wrapper around the `virtualenv` package. A friend used this extensively and recommended it. I never got around to trying it. It makes all environments in a common directory (home) and offers a good workflow for switching between environments. I don’t know much about this one apart from this and didn’t bother researching about it for this blog.

## Current setup

All my development work is done on Ubuntu 20.04 inside WSL2. After doing a fresh install of WSL2, I install pyenv and install the latest python version. I leave the system python as it is, and you should too.
The python which comes installed by default on Ubuntu is shit. Someone on the Ubuntu team decided it’d be nice to amputate the python stdlib into different packages. As a result, the default python installed on Ubuntu doesn’t come with a lot of development-related packages like `venv`, which need to be installed separately as `python-venv` and `python-dev`.
Instead, I’d say leave default python alone, install a python version with pyenv and set it as global.
Do not remove the default python, as many Linux tools may depend on it.
Every project gets a default python version set using pyenv, with a virtual environment created using venv. I also maintain the [requirements.txt](/posts/python-dependency-management/) file for the project. This simple pyenv + venv + requirements.txt setup works for a lot of use cases. If code depends on system dependencies that need to be isolated, I’ll use conda.

## Summary

If you’re reading this and have just started with Python, I’d say don’t get into the habit of frequently switching between different tools. This goes for any developer tool, really. The tools mentioned in this blog have been explored and used over a long period. That said, don’t get hung up on one tool, try new ones now and then. Be patient, but not stubborn.

---
title: Python dependency management with requirements.txt
date: 2020-04-10
type: "post"
description: "Creating and maintaining requirements.txt for python projects"
tags:
 - python
---

This article is a follow up to my [previous article](https://arionmiles.me/posts/using-virtual-environments-in-py3/) on using virtual environments in python3. This may become a series of articles for getting familiar with python developement for beginners.

In this one we'll be talking about a file called `requirements.txt`. If you've looked at open source python projects on GitHub or anywhere else, chances are you have come across this file. If you haven't, you have all the more reason to read this blog.


## What is requirements.txt?
It's a text file. I'm sorry please don't close this tab.

The name of the file is pretty much self-explanatory, but if you haven't caught up yet, the file contains a list of requirements, i.e. python modules/packages that you must have installed in order to run the project it belongs to.

Let's take a look at the `requirements.txt` from one of my projects, [MIS-Bot](https://github.com/ArionMiles/MIS-Bot). Looking at the contents, it contains the names of some packages and their versions.

```
python-telegram-bot==11.1.0
Scrapy==1.4.0
scrapy-splash==0.7.2
SQLAlchemy==1.3.0
sympy==1.1.1
requests==2.20.0
...
```

It contains package names and their versions (`major.minor.patch`) separated by `==`.
In essence, a `requirements.txt` file is a list of arguments to the `pip install` command.

## Why do I need this?
There are two main reasons for having a requirements.txt for your project.

1. Collaboration

    If you are collaborating with someone on a project, you and your partner need to have the same specific versions of packages installed or you may end up writing code which works on your machine and not on your partner's.

2. Consistency

    If you don't include the specific versions along with your module names, it's almost certain that your project will stop working in the future as your dependencies get updated to newer versions and they include breaking changes.

    Suppose you're working on a Django project, you pip install the latest version of django (say v2.2). You work on the project for some time, put it online on GitHub for everyone to try. More than a year later, some other developer decides to try out your project, but since you did not include versions of your dependencies, the other developer will install their latest available versions (say v3.x). Now, since this is a major version change, it will most certainly will throw deprecation warnings and errors when you try to run it.


### Creating requirements.txt
Keep in mind that the file should only contain names and versions of python packages, which can by fetched from PyPI (Python Package Index) or similar host which contains the python files. It shouldn't contain any external dependency like a binary or additional software not written in python. For such dependencies, specify their installation steps and download links in your README.

The usual way I generate the file for my projects is:

1. Create & activate a virtual environment for my project
2. Install the necessary packages during the course of development
3. Run `pip freeze > requirements.txt`

This above command (`pip freeze`) will list all installed packages in your environment along with their version number, we then pipe it to a file called `requirements.txt`.

#### Constrainted requirements
By default, when you use `pip freeze > requirements.txt`, it specifies a strict version using `==`, i.e. only that specific version of the library is needed. 

But you may have certain cases where you need a dependency to be above or below a certain version. 

For example:
- The code you've written uses a feature introduced in a certain version, and all future versions have included it.
- If you make use of a feature which has been removed after a specific version.
- If a very specific version of the module causes problems in your code, but any other version is fine.

To make this possible, pip provides certain version specifiers, a whole list of which is [available here](https://www.python.org/dev/peps/pep-0440/#version-specifiers).

Most common ones are:
```
SomeProject             # No strict version
SomeProject == 1.3      # Only v1.3
SomeProject >=1.2,<2.0  # Greater than or v1.2, and less than v2.0
SomeProject[foo, bar]   # Between versions foo & bar
SomeProject != 1.1      # Skip v1.1
```

Note that the comma (`,`) works as a logical AND operator, so you can chain these together.


You can also specify a library from a URL in your requirements. For example: I've specified a GitHub repo link of a python package/library in the [requirements.txt of MIS-Bot](https://github.com/ArionMiles/MIS-Bot/blob/master/requirements.txt#L9) since that package is unavailable on PyPI where `pip` looks for packages by default.

**NOTE**: The library you're attempting to install here must be structured as a proper [python package](https://packaging.python.org/tutorials/packaging-projects/) and have a setup.py file in order to be able to install from GitHub or similar VCS (Version Control Systems)

Simply run 
```
pip install https://github.com/{USERNAME}/{MYPROJECT}/archive/{BRANCH}.zip
```

So in my instance the URL was 
```
https://github.com/ArionMiles/securimage_solver/archive/pip_package.zip
```

`pip_package` is the name of the branch to install from.

You can simply put the URL in the `requirements.txt` to make it a part of your dependencies.

#### Environment markers
You can also specify an "environment marker" in your requirements file. As to what it is, it allows you to specify which dependencies are to be ignore or installed dependending on whether the system running the pip command meets the conditions.

For example, you could be writing your code for Linux and Windows systems, and windows requires certain package, while Linux doesn't, so you can specify using the `sys_platform` environment marker:
```
SomeProject==1.3; sys_platform=='win32'
```

A good usecase for this can be waitress and gunicorn, two WSGI libraries useful for working with web development frameworks such as Django/Flask. `gunicorn` is made for Linux only, while `waitress` is an alternative to gunicorn on Windows. So you could specify this in your requirements as:

```
gunicorn==20.0.4; sys_platform='linux'
waitress==1.4.3; sys_platform='win32'
```

This will ensure that if you run `pip install -r requirements.txt` from a windows machine, gunicorn will be ignored and waitress will be installed. Likewise, if ran from a linux environment, only gunicorn will be installed.

You can learn about more such useful environment markers in [PEP 508](https://www.python.org/dev/peps/pep-0508/#environment-markers).

#### Comments
A line beginning with `#` symbol will be treated as a comment. 

You can use it for:
- Ignoring certain packages. Useful for quick debugging.
- Add a quick note about why you're using a specific package. However, only do it if it's of note.
- Mark specific sections in the requirements file, though it will make no difference `pip`, but might be useful for humans.


### Installing from a requirements.txt
When you come across a project or inherit a codebase which already has the requirements file, you can quickly setup a virtual environment and install the dependencies using the below command:

```
pip install -r requirements.txt
```


## Multiple requirements.txt?
A good pattern that I've seen many developers adopt is the use of multiple requirement files.

The files are separated based on whether it's a development, production or testing environment.

- `base.txt` - contains core dependencies which every other file will utilize.
- `development.txt` - only contains packages required when working on development of the application. This may include packages like `django-debug-toolbar` if you're working on a Django project.
- `production.txt` - when you're deploying the project somewhere. This may include packages like `gunicorn`.
- `testing.txt` - for your testing environment. This may include testing frameworks, code coverage tools, etc.

A simpler division could be just `base.txt` and `development.txt`.

The `base.txt` will contain packages like:
```
python-telegram-bot==11.1.0
Scrapy==1.4.0
scrapy-splash==0.7.2
SQLAlchemy==1.3.0
...
```

and the `development.txt` file will contain:
```
-r base.txt
SomeDevelopementPackage==1.0.5
...
```

This means when you run `pip install -r development.txt`, it'll install all packages in `base.txt` and then install additional modules required for development.

## Conclusion
Even though it may feel as if it's not worth having a `requirements.txt` for your project, if you're putting it online for collaboration it's very much helpful to make it available and easier for others to use your code. Even if you're uploading the project for showcase, having the file can signal to your audience that you've actually put some thought towards making your work maintainable. Besides, creating the file only takes a moment of your time.

This article may look like a lot to take in, but I've tried to give a basic idea along with a few features that are available. If you feel overwhelmed, just start with a simple `pip freeze > requirements.txt`.

---
title: Using virtual environments in Python 3
date: 2020-04-03
type: "post"
description: "Detailing the importance of virtual environments in Python, yet again."
tags:
 - python
---

I'm writing this short post on the use of virtual environments after I've seen a lot of my classmates at college blindly pip install every python package under the sun in their system python installation, pollute their entire python installation and then wonder why their code won't run.


## What are virtual environments?
As the name suggests, a virtual environment (venv for short), is an isolated environment which mimics the same behaviour as your system Python installation (global Python installation). Any python package you install while your virtual environment is activated is isolated from your system python installation.


## Why do I need a venv?
Consider that you're working on 2 projects A and B which use Django (a very popular web development framework in Python).

As of writing this, Django has three major versions, `v1.x.x`, `v2.x.x`, and `v3.x.x`

Suppose project A uses Django `v1.11` and project B uses `v2.2`. If you install Django v1.11 and work on project A for a little while, then  after a while, you go to work on project B. Given project B requires v2.2, and it is a major version change from v1, you will get errors and deprecation warnings when you attempt to run your code, since a lot of things used in v1 have now been removed in v2.

Likewise, if you previously installed v2.2 to work on project B and then moved to work on A, you'll get errors saying certain modules/functions were not found, since the newer stuff introduced in v2 didn't exist in v1.

So to resolve this, you'll have to uninstall and reinstall the two Django versions as you switch between the projects.

Now you may say, this is not so painful or bothersome, since you can uninstall and reinstall these specific packages whenever you switch between projects. Sure, but chances are, Django is not the only dependency in your entire project. There may be other packages Django depends on, which aren't uninstalled when you uninstall Django, and unless you keep track of all of them, even those packages may cause conflicts as you upgrade/downgrade Django.


## Requirements
 - Python v3.3 and up
 - `venv` module (part of Python standard library)


## Creating and using virtual environments
Fire up your terminal, create an empty directory and inside it, run the following command:

```
$ python3 -m venv .venv
```

Notice that `.venv` will be the name of the folder inside which the virtual environment files will be placed. I've chosen `.venv` as the name since it's a very common name and ignored by default by most `.gitignore` files if you work with Python a lot.

You can also choose to name it `projectName` or `projectName-venv` so that you know you've activated the appropriate virtual environment when working with multiple projects.

### Activating the venv
The command for activating the virtual environment differs from platform to platform and also depends on the shell which you're using. Below is a table from [official Python documentation](https://docs.python.org/3/library/venv.html) which lists the activation commands for some common platforms and shells.

 Platform | Shell | Command to activate virtual environment 
 ------------- |:-------------:| :-----:
POSIX|bash/zsh|`$ source <venv>/bin/activate`
||fish|`$ . <venv>/bin/activate.fish`
||csh/tcsh|`$ source <venv>/bin/activate.csh`
||PowerShell Core | `$ <venv>/bin/Activate.ps1`
Windows | cmd.exe | `C:\> <venv>\Scripts\activate.bat`
||PowerShell|`PS C:\> <venv>\Scripts\Activate.ps1`

For most people, `bash` and `cmd.exe` commands are all they need if they're on Linux and Windows, respectively. 

Run the appropriate command, and your terminal will read something like this if you're on Linux:
```
(venv) $
```

or like this if you're on windows:
```
(venv) C:\>
```

### Deactivating the venv
After you're done with your work, you can deactivate the virtual environment to use the normal shell or perhaps if you wish to switch to another project's virtual environment.

Just run `deactivate` command irrespective of the platform or shell:
```
(venv) $ deactivate
$
```


## What do I use the global installation for?
As a rule of thumb, I never install any project related dependency in my global python installation. I only keep python utilities I use frequently (e.g `youtube-dl`). Reason being I can use it from anywhere without having to activate a virtual environment first.


## IDE & Editor support
Most modern editors and IDEs like [VS Code](https://code.visualstudio.com/) and [PyCharm](https://www.jetbrains.com/pycharm/) can readily make use of the virtual environments you create. With PyCharm going as far as allowing you to create a new project with a dedicated virtual environment right from the get go.

So look up if virtual environments are supported by your IDEs and you can get the best use out of your tools.


## Alternatives
Apart from the `venv` module included in the python standard library, there are other solutions for management of virtual environments. A couple of those include-

- [virtualenv](https://virtualenv.pypa.io/en/latest/) (More feature rich and backwards compatible with py2)
- [virtualenvwrapper](https://virtualenvwrapper.readthedocs.io/en/latest/)
- [Conda environments](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html) (If you installed python through Anaconda)

Personally I've felt that the stdlib `venv` package fulfills my needs so I never use the above mentioned packages. If you feel like the stdlib solution isn't enough, feel free to explore the above options and see if it's a good fit. 

As for me, I'd rather not complicate my environment setup anymore than I need to and focus on building my software.


## Conclusion
The use of virtual environments can prove to be a very useful tool as you grow in experience and start working with more than one project at a time.

For some, it may look like an extra unnecessary step you need to follow in order to get your project up, since you won't realize how much pain they're saving you as long as you use venvs. 

But it only takes one bad evening of trying to figure out why your code won't run when you realize, after hours of looking up StackOverflow threads that you're using an incompatible version of numpy since you installed it long ago and never bothered updating it to know the importance of virtual environments.

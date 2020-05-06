---
title: Creating AWS Lambda layers for ELF Binaries
date: 2020-05-04
type: "post"
description: "Using ELF binaries inside lambda applications"
tags:
 - linux
 - serverless
---

Last week a friend told me about a new app he was writing and it involved use of an application which was quite CPU heavy, and we wondered if we could save CPU costs by offloading that workload to an AWS Lambda function. I had never touched AWS Lambda prior to this, and I had a vague overview of what serverless applications were from a presentation I did for a college course, but I was very bored in lockdown and the prospect seemed fun, so I took it upon myself to make this happen.

TL;DR for serverless paradigm: A Lambda function is basically a stateless execution environment, which is only fired up on demand, so it saves costs of managing, hosting full fledged servers (also called serverful apps). So whenever you need something done, you trigger your function, it takes your input, carries out whatever operation you program it to, returns a response and shuts down until it is triggered again.

This write up contains my learnings and things of note I believe will be useful for people who are new to the serverless world, and also serve as a way to quickly refresh my memory if I work on more serverless stuff in the future.


## AWS Lambda Application Structure
The basic way to deploy your app using Lambda is to package the code and the dependencies into a zip file and just upload it directly, either from the AWS Lambda console or upload to an S3 bucket and reference the bucket and file in the Lambda console.

This approach is sufficient if your application is fairly simple, and has very little to no dependencies. But even then, I feel packing your dependencies along with the code has 2 drawbacks

1. You cannot swap versions of your dependencies without having to re-upload the code too.
2. To reuse the dependencies in another lambda function, you need to bundle it again with the code for that application.

Luckily for us, in 2018, AWS introduced "Layers" for Lambda. These layers are independent from your code. Your code is mounted in `/var/task/` while everything inside your layers is mounted in `/opt/`.

Lambda layers solve the above two pain points. Every successive update to the layer acts as a different version of the same layer, so if you feel that the newer layer for a dependency is insufficient or broken, you can easily switch to a previous working version. Layers are also not limited to one function, they can be freely referenced in unlimited number of functions.


## An impossible task
We'll build a simple lambda application as an exercise which relies on a linux binary to work. This walk-through can be done on any linux machine or if you're on Windows 10, you can use [WSL (Windows Subsystem for Linus)](https://docs.microsoft.com/en-us/windows/wsl/install-win10). This article has been made using WSL.

Let's take `rig`, it stands for Random Identity Generation, it's a classic linux utility that when called, prints a fake name, address and zip code.

If you don't have it pre-installed, you can download rig and zip (we'll be using it later) by:
```
$ sudo apt install -y rig zip
```

Get a fake identity by simply calling `rig` inside your shell.
```
$ rig
Daphne Nichols
7 Fairfield Rd
Detroit, MI  48233
(313) xxx-xxxx
```

The lamda application we'll be building will simply call this binary and return the text as a response.

Let's get started. Create a empty directory, and make a `lambda_function.py` file.
```
$ mkdir -p lambda-example/src/
$ cd lambda-example/src
$ echo > lambda_function.py
$ cd ..             # Come back to project root
```

and paste the below code into the file
```python
import subprocess

def lambda_handler(event, context):
    rig_command = "rig"
    random_identity = subprocess.getoutput(rig_command)
    return {
        "identity": random_identity,
    }

if __name__ == "__main__":
    # Test function
    print(lambda_handler("", ""))
```

## Packing `rig` for lambda
Now the issue is, the Linux image Lambda runs (Amazon Linux) doesn't come preinstalled with `rig`, and we cannot simply make a shell call to install the package on every invocation for 2 big reasons:

1. Expensive time cost, on every invocation we'd be installing this package and then giving the output. What if the package is unavailable for the package manager Amazon Linux uses?
2. The entire environment is read-only (except for `/tmp/`) so even if we _could_ do 1, we can't.

The way it works is quite simple, you put all binaries inside a `bin/` directory, and all Shared Object (`.so`) files inside a `lib/` directory, zip them up and upload them as a lambda layer.

Let's find out where the binary is:
```
$ which rig
/usr/bin/rig
```

Now we'll use `ldd` to list all dependencies (shared objects) it needs to properly function.
```
$ ldd $(which rig)
        linux-vdso.so.1 (0x00007fffc82f4000)
        libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007fdd275f0000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007fdd273d0000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fdd26fd0000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007fdd26c30000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fdd27c00000)
```

Now, you can choose to copy the binary and all these dependencies into `bin/` & `lib/` respectively, but it's an endless war and nothing but frustration, and unless you're into masochism, this is not worth your time.

Here's a few problems which you'll encounter even if you manage to put all this together manually (or even write fancy bash scripts if you're in a good mood):

1. Dependency-ception
    These dependencies might depend on some other libraries not listed by `ldd`.
2. All ELF executables have a hardcoded interpreter path which is used by the kernel to start the program.

>>> [Source](https://intoli.com/blog/transcoding-on-aws-lambda/)


## `exodus` from this mess
I was trying to create layers for my friend's application using the above mentioned strategy to no avail, encountering cryptic and mysterious errors like:
```
`<some-binary>: /lib64/libz.so.1: version ZLIB_1.2.9' not found (required by /opt/lib/libpng16.so.16)
```
I have mentioned the issue of "dependency-ception" above, this is exactly that. Looks like my binary's dependencies require zlib packaged as well.

I asked around and a friend suggested this magical tool called [exodus](https://github.com/intoli/exodus). This was a mind-blown moment for me. This was exactly I had been looking for all along!

Exodus makes everything simple! It takes care of everything! Just run one command and it'll create a tarball of any ELF binary you wish to port to another system. It was originally made with the aim of allowing any binary from system X run on system Y, Z and so on.

You can install it from pip by:
```
$ pip3 install --user exodus-bundler
```

Let's quickly make a tarball of our `rig` binary, and extract the contents:
```
$ exodus --tarball rig | tar -zx
```

You may see a warning saying...
```
WARNING: Installing either the musl or diet C libraries will result in more efficient launchers (currently using bash fallbacks instead)
```
If you have gcc and musl or dietlibc installed, exodus will identify them and try to create smaller, statically linked launchers which are faster, but it defaults to shell scripts if these additional dependencies aren't present, but they say this may carry significant overheads. 

So depending on how much performance you're trying to get out of your binary, you may choose to install the optional dependencies with exodus.


Moving on, we'll see a `exodus/` directory now. We'll zip the contents of this directory into `rig.zip` file.
```
$ cd exodus/
$ zip -r9 ../rig.zip *
$ cd ..
```
This will give us a zip file and we can start uploading our code.

**TIP:** Make sure every file which will be uploaded to lambda has 755 permissions. If it doesn't, change permissions using below command which changes permissions on all files inside `src` to 755:
```
$ chmod -R 755 src/
```

## Preparing for Deployment
AWS SAM (Serverless Application Model) is an application which allows you to easily deploy your functions in a declarative manner. SAM uses configurable YAML file which contains definition of your function, the permissions/policies it will require, events (or triggers) for your function, and any other AWS resource it may require.

You can install SAM through pip, along with AWS-CLI which you need since SAM picks up your AWS keys after you configure it in `aws-cli`

```
$ pip3 install --user awscli aws-sam-cli
$ aws configure
```

Run `aws configure` next and add your access keys and secret keys. You can [learn how to create those keys here](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html). Next select your default region and response format (json).

We'll use the below SAM template for our project. Save the contents of this gist to `template.yaml` and save it in your project root.

{{< gist ArionMiles f57ca40f83d0445d69edf02d92d9ad45 >}}

```
$ sam deploy --guided
```
Run the above command from the project root (where you have `template.yaml`) and follow the instructions. This will create a CloudFormation stack containing the function, permissions, relevant roles, and the layers. You can view it from the [Lambda Console](https://console.aws.amazon.com/lambda/) and if you wish to delete everything related to the lambda application, you can do so from [CloudFormation console](https://console.aws.amazon.com/cloudformation/).

Go to Lambda > Applications > Select your "rig" application (whatever you named the app during `sam deploy`)

{{< video webm="/videos/rig_demo.webm" >}}

Scroll to the bottom, under resources, select your Function. When the console opens, create a simple test (use the default test template provided) and test the function. You'll see this output:

```
{
  "identity": "/opt/bin/rig: line 9: /opt/bin/./linker-2d196bc8632e500316fa0e0c3e8f40d0e7da853ae940805080b3492ce03b7b51: No such file or directory"
}
```

So it looks like exodus didn't work. Or maybe we're doing something wrong. Let's check.
```
$ ls -la exodus/*
exodus/bin:
total 0
drwxrwxrwx 1 arion arion 4096 May  6 01:24 .
drwxrwxrwx 1 arion arion 4096 May  6 01:24 ..
lrwxrwxrwx 1 arion arion   87 May  6 01:24 rig -> ../bundles/9cf834c649ca952381d55810f61a4594cd8fb8d965b2018b5aa055c4bfc5cd34/usr/bin/rig
```
so we see that the binary `exodus/bin/rig` is a symlink (symbolic link) to another file in `bundles/`. When we zip them, these symlinks are not preserved, hence we get the above error.

Let's fix this. The `--symlinks` flag will instruct zip to preserve the symlinks. 
```
$ rm rig.zip        # remove pre-existing zip
$ cd exodus/
$ zip --symlinks -r9 ../rig.zip *
$ cd ..
```

Let's deploy it again.
```
$ sam deploy
```
On testing once again, we see a new error (cleaned up):
```
Unable to read file /usr/share/rig/locdata.idx

USAGE: rig [-f | -m ] [ -d datadir ] [ -c num ]
       datadir - Directory where data files can be found.
       If datadir is not specified, /usr/share/rig is used as the
       default directory.
       num - print num identities.
       -f and -m specify gender of generated identities.
```
It turns out `rig` utilizes pre defined data files to generate the random identities. They reside in `/usr/share/rig`

Exodus allows you to add additional files using the `--add` flag.
So we could do 
```
$ rm -r exodus/
$ exodus --tarball --add /usr/share/rig rig | tar -zx
$ cd exodus/
$ zip --symlinks -r9 ../rig.zip *
$ cd ..
```

I tried this, but turns out this additional file is mounted at `/opt/bundles/<hash>/usr/share/rig` and the PATH of the lambda environment contains these directories:
```
/var/lang/bin:/usr/local/bin:/usr/bin/:/bin:/opt/bin
```
So the `/opt/bundles/<hash>/user/share/rig` needs to be added to the path since we can't write to `/usr/share` on the lambda environment. Doing PATH shenanigans is something I'd advise against. It's messy.

Fortunately for us, as evident in the above error and usage message, `rig` allows us to specify a data file to generate name files from. So we can simply copy `/usr/share/rig` from our local system into `exodus/rig` and zip everything up.

```
$ rm rig.zip 
$ rm -r exodus/
$ exodus --tarball rig | tar -zx
$ cp -r /usr/share/rig exodus/rig
$ cd exodus/
$ zip --symlinks -r9 ../rig.zip *
$ cd ..
$ sam deploy
```

Also, change `rig_command` in `src/lambda_function.py` to tell rig where to pick up data files from:
```python
rig_command = "rig -d /opt/rig"
```
Now deploy once again, and sure enough, you'll see an output like this:
```
{
  "identity": "Harriet Martinez\n855 Cimenny Rd\nMiami, FL  33152\n(305) xxx-xxxx"
}
```
And that's it! We're able to use a binary not originally available inside the lambda environment. We can use the sam layer by specifying its ARN in any other function we want. 

Exodus has a bunch of nice features, I've already mentioned `--add` flag to bundle additional files, but you can also infer runtime dependencies using `strace` and pipe them to exodus. You can find more about their features in [this article](https://intoli.com/blog/exodus-2/)

With that said, it also has some limitations. Adding external files didn't work for me, but, it's still very new, so you can expect improvements. You can find more about its limitations [here](https://github.com/intoli/exodus#known-limitations).


## Lessons learnt
1. 
2. If your binary depends on additional data files which may be present in paths not included under the Lamda execution environment, it's better to find a way to configure the binary to load data files from some sort of configuration, like a flag (`-d` in our case).
3. Symlinks will ruin your life. I wasted a day until it was pointed out to me that I wasn't preserving the symlinks while zipping the files.

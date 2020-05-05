---
title: Creating AWS Lambda layers for ELF Binaries
date: 2020-05-04
type: "post"
description: "Using ELF binaries inside lambda applications"
tags:
 - linux
 - serverless
---

Last week a friend told me about a new app he was writing and it involved use of an application which was quite CPU heavy, and we wondered if we could save CPU costs by offloading that workload to a AWS Lambda function. I had never touched AWS Lambda prior to this, and I had a vague overview of what serverless applications were from a presentation I did for a college course, but I was very bored in lockdown and the prospect seemed fun, so I took upon myself to make this happen.

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
We'll build a simple lambda application as an exercise which relies on a linux binary to work.

Let's take `rig`, it stands for Random Identity Generation, it's a classic linux utility that when called, prints a fake name, address and zip code.

If you don't have it pre-installed, you can download it and a few more utilities we'll be using:
```
$ sudo apt install -y rig zip jq
```

Get a fake identity by simply calling `rig` inside your shell.
```
$ rig
Daphne Nichols
7 Fairfield Rd
Detroit, MI  48233
(313) xxx-xxxx
```

The lamda application we'll be building will simply call this binary and return the text as a json response.

Let's get started. Create a empty directory, and make a `lambda_function.py` file.
```
$ mkdir -p lambda-example/src/
$ cd lambda-example/src
$ echo > lambda_function.py
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


## `exodus` from this headache
I was trying to create layers using the above mentioned strategy to no avail, encountering cryptic and mysterious errors like:
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
```

Run `aws configure` next and add your access keys and secret keys. You can [learn how to create those keys here](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html). Next select your default region and response format (json).

We'll use the below SAM template for our project. Save the contents of this gist to `template.yaml` and save it in your project root.

{{< gist ArionMiles f57ca40f83d0445d69edf02d92d9ad45 >}}



>>>> symlink issueee
```
/var/task/exodus/bin/<some_binary>: line 9: /var/task/exodus/bin/./linker-2d196bc8632e500316fa0e0c3e8f40d0e7da853ae940805080b3492ce03b7b51: No such file or directory
```

```
$ cd exodus/
$ zip --symlinks -r9 ../rig.zip *
$ cd ..
```

>>>> /usr/share/rig/ issueeee


## Sources
1. https://intoli.com/blog/transcoding-on-aws-lambda/
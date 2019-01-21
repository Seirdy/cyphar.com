title: "umoci: a New Tool for OCI Images"
author: Aleksa Sarai
published: 2016-11-29 15:30:00
updated: 2016-12-16 18:15:00
short_description: >
  The lack of tooling for OCI container images is a real concern, especially
  given the slow progression to 1.0 without any implementations. I wrote one
  that I hope is generic enough that it can be used (and also gives the
  opportunity to do cool things with it).
description: >
  Very recently, I've been working on implementing the required tooling for
  creating and modifying [Open Container Initiative](https://www.opencontainers.org/)
  images without needing any external components. The tool I've written is
  called `umoci` and is probably one of the more exciting things I've worked on
  in the past couple of months. In particular, the applications of `umoci` when
  it comes to SUSE tooling like the [Open Build Service](http://openbuildservice.org/)
  or [KIWI](https://suse.github.io/kiwi/) is what really makes it exciting.
tags:
  - oci
  - containers
  - free software
  - kiwi
  - suse

*[`umoci`][umoci] is a free software command-line tool I've been working on that
allows someone to create and modify an [OCI][oci] image without needing to
understand any aspect of how OCI images are structured. Currently `umoci` is in
an alpha stage, with lots of features being actively developed. It will be
heading for [a `0.0.0` beta release][milestone-0.0.0] in the coming weeks.*

In recent months, the OCI [image][oci-image] and [runtime][oci-runtime]
specifications have been slowly moving towards `1.0` status. However, [a lot of
discussion][gh-image-users] has come up regarding users of the image
specification and how stable we can consider the specification if nobody has
really created new tooling around it or is using it in production.

To be fair, there is some work to get support for the format [inside
Docker][docker-oci], but that hasn't landed yet and it isn't clear if Docker
will ever fully switch to the OCI specification. There are also plans to add
OCI support to [`acbuild`][acbuild] but I haven't seen any actual code to that
effect.

So, I decided to stop waiting for someone else to implement what was necessary
and just decided to do it myself. In particular, I'm hoping that a lot of the
code I've written will be merged back into the OCI's [`image-tools`][oci-image-tools]
codebase so that it can be maintained by the same body of people that are
writing the spec. While I am a contributor to the OCI specifications, I don't
want to have to maintain a library for a specification that might drastically
change in the future.

The main reason why openSUSE and SUSE would be interested in this sort of
tooling is better explain in the [applications section](#applications). But the
short version is that it's related to integrating the automated building of OCI
and Docker images in [our build infrastructure][obs-opensuse] without needing
to run Docker inside our build infrastructure. This would be done by
integrating `umoci` into [KIWI][kiwi] and the [Open Build Service][obs].

[umoci]: https://github.com/cyphar/umoci
[oci]: https://www.opencontainers.org/
[milestone-0.0.0]: https://github.com/cyphar/umoci/milestone/2
[oci-image]: https://github.com/opencontainers/image-spec
[oci-runtime]: https://github.com/opencontainers/runtime-spec
[gh-image-users]: https://github.com/opencontainers/image-spec/issues/126
[docker-oci]: https://github.com/docker/docker/pull/26369
[acbuild]: https://github.com/containers/build
[oci-image-tools]: https://github.com/opencontainers/image-tools
[obs-opensuse]: https://build.opensuse.org/
[kiwi]: https://suse.github.io/kiwi/
[obs]: http://openbuildservice.org/

### Previous Work ###

While there has been a lot of discussion about various projects (such as
[`rkt`][rkt-oci] and others) implementing OCI specification support, as far as
I'm aware there's only two examples of "completed" tooling for OCI images:

* [`skopeo`][skopeo] which is a tool for translating between different image
  formats. `skopeo` supports translating to-and-from different image formats
  (thanks to the [`containers/image`][containers/image] library) including the
  Docker registry, Docker archive and OCI image formats. I'm personally very
  excited about `skopeo` because it means that we can use OCI-only tooling to
  create a Docker image without ever needing to run a Docker daemon in our
  build infrastructure!

* And of course the [official `oci-image-tools` tooling by the
  OCI][oci-image-tools]. The main issue with this tooling is that it isn't
  full-featured enough to be able to fully interact with and manipulate OCI
  images. There are [several][oci-pr5] [pull requests][oci-pr8] to implement
  functionality that would make the `image-tools` *more* useful but still would
  not make that tooling useful for anyone who is not intimately familiar with
  the OCI specification. In addition, those pull requests have been open for a
  long enough time that I'm convinced they won't be merged any time soon (the
  discussion has stalled).

So, all in all, there isn't really any OCI-native tooling that allows us to
modify an OCI image without understanding the internals of the spec. In
addition, because we want to add support for this tooling into KIWI, we need to
have a tool that doesn't rely on using containers to modify the contents of an
image (which is a restriction of Docker as well as most other tooling), because
KIWI already knows how to install packages with `zypper` and so on.

[rkt-oci]: https://github.com/coreos/rkt/projects/4
[skopeo]: https://github.com/projectatomic/skopeo
[containers/image]: https://github.com/containers/image
[oci-pr5]: https://github.com/opencontainers/image-tools/pull/5
[oci-pr8]: https://github.com/opencontainers/image-tools/pull/8

### Usage ###

Now, with all of that background out of the way, what does `umoci` actually
look like? Unlike most tools these days, the interface does **not** look like
`git` -- though the concepts are fairly similar because of how the OCI
specification works.

Looking at the help page, we can get an idea where to start from:

```language-bash
% umoci -h
NAME:
   umoci - umoci modifies Open Container images

USAGE:
   umoci [global options] command [command options] [arguments...]

VERSION:
   0.0.0~rc2

AUTHOR(S):
   Aleksa Sarai <asarai@suse.com>

COMMANDS:
     help, h  Shows a list of commands or help for one command

   image:
     config      modifies the image configuration of an OCI image
     unpack      unpacks a reference into an OCI runtime bundle
     repack      repacks an OCI runtime bundle into a reference
     new         creates a blank tagged OCI image
     tag         creates a new tag in an OCI image
     remove, rm  removes a tag from an OCI image
     stat        displays status information of an image manifest

   layout:
     gc        garbage-collects an OCI image's blobs
     init      create a new OCI layout
     list, ls  lists the set of tags in an OCI image

GLOBAL OPTIONS:
   --debug        set log level to debug
   --help, -h     show help
   --version, -v  print the version
```

Currently `umoci` doesn't have any man pages, though I am currently [working on
it][umoci-man], so I'll give some more description about what the general
workflow is of `umoci` and how we can use it to modify images.

#### Getting an Image ###

I would recommend getting an OCI image using [`skopeo`][skopeo] so that you can
mess around with an image that already has content. Here's how you can pull an
image using `skopeo` (if you're on openSUSE you can get `skopeo` from [this
repository maintained by us][obs-vc-skopeo]):

```language-bash
% # Download the latest openSUSE Leap 42.2 image from the Docker hub, convert
  # it to an OCI image and store the OCI image in a directory called "opensuse".
  # The tag of the image converted to the OCI image is "latest".
% skopeo copy docker://opensuse/amd64:42.2 oci:opensuse:latest
Getting image source manifest
Getting image source signatures
Getting image source configuration
Uploading blob sha256:16724059119c810dd03218fd597625067dcf14d661c687a4396413345ced91c4
 0 B / 1.73 KB [---------------------------------------------------------------]
Uploading blob sha256:467db25190688bc9dc1d5fd6dbced1ac56f55d93a942987f448e62ad2614e46e
 46.97 MB / 46.97 MB [=========================================================]
Uploading manifest to image destination
Storing signatures
% # You can copy more than one Docker image into a single OCI image.
% skopeo copy docker://opensuse/amd64:42.1 oci:opensuse:old
Getting image source manifest
Getting image source signatures
Getting image source configuration
Uploading blob sha256:f041be4a5bbe4e191ec9e323f30a7e82ec4a1781e3376a885e8e5218bce6400c
 0 B / 1.73 KB [---------------------------------------------------------------]
Uploading blob sha256:b5b3627caa3d91971b1d701feec88c16e621d9f28122facfe22dd6ab57476221
 37.03 MB / 37.03 MB [=========================================================]
Uploading manifest to image destination
Storing signatures
```

It is also possible to [create an image from scratch with `umoci new` and
`umoci init` ](#creating-new-images), but that's not as cool as modifying a
Docker image using OCI tooling.

Now that we have an OCI image in a particular directory, we can play around
with the image using `umoci`. Note that some of the modifications we do require
privileges, but that's due to the nature of having to extract an image that has
files owned by more than one user.

[umoci-man]: https://github.com/cyphar/umoci/issues/34
[obs-vc-skopeo]: https://build.opensuse.org/package/show/Virtualization:containers/skopeo

#### Interacting with Tags ####

OCI images have a list of "named references", which are the same as [tags in
Docker][docker-tags]. Essentially inside an OCI image bundle, there can be any
number of tagged images. An image inside an OCI image bundle is the combination
of a configuration file and the layers that make up the root filesystem of the
image.

With `umoci ls` we can take a look at what we just pulled.

```language-bash
% umoci ls --layout opensuse
latest
old
```

The very first thing to note is the `--layout opensuse` argument. Since every
OCI image is independent, we have to specify which one is the one we're
operating on. All `umoci` commands take either a `--layout` or `--image`
argument (in `umoci -h` you can see what commands fall into the respective
categories). `umoci` supports local, directory-based OCI images so `--layout`
(or `--image`) refers to a local directory (it's unclear if the OCI will in
the forseeable future define any remote-registry API).

We can also create, inspect or delete tags using `umoci tag`, `umoci stat` and
`umoci rm` respectively. A trivial example is swapping the `old` and `latest`:

```language-bash
% umoci stat --image opensuse:latest
LAYER                                                                   CREATED                        CREATED BY                                                                                        SIZE    COMMENT
<none>                                                                  2016-12-14T00:17:31.334478162Z /bin/sh -c #(nop)  MAINTAINER SUSE Containers Team <containers@suse.com>                          <none>
sha256:d05d5d3c35088ecc4cf83c172c881cedc58df38ed127a9b0dc4adda9b291afa3 2016-12-14T00:17:44.456345827Z /bin/sh -c #(nop) ADD file:379aeb5188443ed46af74119ff2eb532b9280e72d137b1e1865ad41911c13e58 in /  49.3 MB
% umoci stat --image opensuse:old
LAYER                                                                   CREATED                        CREATED BY                                                                                        SIZE     COMMENT
<none>                                                                  2016-12-14T00:13:28.318720201Z /bin/sh -c #(nop)  MAINTAINER SUSE Containers Team <containers@suse.com>                          <none>
sha256:8f0d2170f95bbad6858ad6432396bfebb990cd3396d841581d4b4c6b55c7d333 2016-12-14T00:13:37.494221488Z /bin/sh -c #(nop) ADD file:2f6306c949ad0e3316bf5afa7b1e9f598d88cedc31caa0abe616bfef470fade6 in /  38.85 MB
% # Preform the swap.
% umoci tag --image opensuse:latest tmp
% umoci tag --image opensuse:old latest
% umoci tag --image opensuse:tmp old
% # Delete the tmp tag.
% umoci rm --image opensuse:tmp
% # Verify that :old is now :latest.
% umoci stat --image opensuse:latest
LAYER                                                                   CREATED                        CREATED BY                                                                                        SIZE     COMMENT
<none>                                                                  2016-12-14T00:13:28.318720201Z /bin/sh -c #(nop)  MAINTAINER SUSE Containers Team <containers@suse.com>                          <none>
sha256:8f0d2170f95bbad6858ad6432396bfebb990cd3396d841581d4b4c6b55c7d333 2016-12-14T00:13:37.494221488Z /bin/sh -c #(nop) ADD file:2f6306c949ad0e3316bf5afa7b1e9f598d88cedc31caa0abe616bfef470fade6 in /  38.85 MB
```
As you can see, `umoci stat` is in the `image` category of commands and thus
takes an `--image` flag. The exact meaning of `--image` depends on the command,
but `--image` is always of the format `layout:tag`. If you don't specify a
`tag`, the default is `latest` (so I could've used `umoci stat --image
opensuse` in the above example if I wanted to save characters).

Please note that the output of `umoci stat` will change in future versions (use
`--json` if you want to script around `umoci stat`) but the general idea will
always be the same -- it gives you information about what a tag points to.

[docker-tags]: https://docs.docker.com/engine/tutorials/dockerimages/#/setting-tags-on-an-image

#### Unpacking ####

One of the most important features of `umoci` is the fact that it allows us to
unpack an image into an [OCI runtime bundle][oci-runtime-bundle], which
includes an extracted root filesystem for the container as well as the OCI
runtime configuration file (which is generated from the OCI image configuration
for the extracted image). This means, if we wanted, we could go from an OCI
image to a running container with `runc` in a few seconds:

```language-bash
% sudo umoci unpack --image opensuse:latest opensuse_bundle
INFO[0000] parsed mappings                    map.gid=[] map.uid=[]
INFO[0000] unpack manifest: unpacking layer sha256:467db25190688bc9dc1d5fd6dbced1ac56f55d93a942987f448e62ad2614e46e  diffid="sha256:33e694f8e290bc896a8da5718854e81845fe79579f34704cf249ea457500134f"
INFO[0001] unpack manifest: unpacking config  config="sha256:16724059119c810dd03218fd597625067dcf14d661c687a4396413345ced91c4"
% sudo runc run -b opensuse_bundle ctr
sh-4.3# cat /etc/os-release
NAME="openSUSE Leap"
VERSION="42.2"
ID=opensuse
ID_LIKE="suse"
VERSION_ID="42.2"
PRETTY_NAME="openSUSE Leap 42.2"
ANSI_COLOR="0;32"
CPE_NAME="cpe:/o:opensuse:leap:42.2"
BUG_REPORT_URL="https://bugs.opensuse.org"
HOME_URL="https://www.opensuse.org/"
sh-4.3# exit
```

Note that `umoci unpack` has other options related to mapping of user IDs (for
rootless and user namespaced containers). However, there's probably enough
information in the help page (and in the man pages) that I don't need to give
an example here.

It is true that this feature already exists within the [`oci-image-tools`][oci-image-tools]
project (under the `oci-create-runtime-bundle` command), but it will become
clear how `umoci` is different in the next section. Also, `umoci unpack` has
stronger guarantees about the reproducibility of extraction than
`oci-create-runtime-bundle` (`umoci` has integration tests that make sure that
the same image will extract to have precisely the same root filesystem, every
time the same image is extracted).

[oci-runtime-bundle]: https://github.com/opencontainers/runtime-spec/blob/v1.0.0-rc2/glossary.md#bundle

#### Repacking ####

While unpacking is all well and good, at the end of the day we want to have an
image that we can distribute (not a root filesystem that is essentially useless
to everyone else). The OCI image specification defines that a root filesystem
is made up of different [diff layers][oci-layers], which is what we extracted
when using `umoci unpack`. But how do we create our own layers once we've
edited our root filesystem to our heart's liking? The answer is, of course,
`umoci repack`:

```language-bash
% # Starting where we left off in the previous example, we still have an
  # unpacked openSUSE Leap 42.2 bundle at opensuse_bundle, which we have not
  # touched.
% # First, let's make some modifications inside a container. If the network
  # doesn't work for you, try removing the "network" namespace from
  # "linux.namespaces" and then add a bind-mount (or just copy) your host's
  # /etc/resolv.conf to the container. This shouldn't be necessary for most
  # people, but it's necessary on my machines (for some unholy reason).
% sudo runc -b opensuse_bundle ctr-build
sh-4.3# zypper lr
# | Alias          | Name           | Enabled | GPG Check | Refresh
--+----------------+----------------+---------+-----------+--------
1 | non-oss        | NON-OSS        | Yes     | ( p) Yes  | Yes
2 | oss            | OSS            | Yes     | ( p) Yes  | Yes
3 | oss-update     | OSS Update     | Yes     | ( p) Yes  | Yes
4 | update-non-oss | Update Non-Oss | Yes     | ( p) Yes  | Yes
sh-4.3# zypper rr 1 4
Removing repository 'NON-OSS' ...........................................[done]
Repository 'NON-OSS' has been removed.
Removing repository 'Update Non-Oss' ....................................[done]
Repository 'Update Non-Oss' has been removed.
sh-4.3# zypper ref
Retrieving repository 'OSS' metadata ....................................[done]
Building repository 'OSS' cache .........................................[done]
Retrieving repository 'OSS Update' metadata .............................[done]
Building repository 'OSS Update' cache ..................................[done]
All repositories have been refreshed.
sh-4.3# zypper in strace
Loading repository data...
Reading installed packages...
Resolving package dependencies...

The following 2 NEW packages are going to be installed:
  libunwind strace

2 new packages to install.
Overall download size: 217.7 KiB. Already cached: 0 B. After the operation, additional 709.7 KiB will be used.
Continue? [y/n/? shows all options] (y):
Retrieving package libunwind-1.1-12.5.x86_64  (1/2),  47.4 KiB (137.4 KiB unpacked)
Retrieving: libunwind-1.1-12.5.x86_64.rpm ...............................[done]
Retrieving package strace-4.10-3.2.x86_64     (2/2), 170.3 KiB (572.3 KiB unpacked)
Retrieving: strace-4.10-3.2.x86_64.rpm ..................................[done]
Checking for file conflicts: ............................................[done]
(1/2) Installing: libunwind-1.1-12.5.x86_64 .............................[done]
(2/2) Installing: strace-4.10-3.2.x86_64 ................................[done]
sh-4.3# strace -V
strace -- version 4.10
sh-4.3# exit
% # Now, let's edit the root filesystem manually, just for the fun of it.
% sudo touch opensuse_bundle/rootfs/a_new_file
% sudo rm -rf opensuse_bundle/rootfs/selinux
% # The fun part! Let's repack the image into a new tag.
% sudo umoci repack --image opensuse:new-latest opensuse_bundle
INFO[0000] parsed mappings    map.gid=[] map.uid=[]
INFO[0002] created new image  digest="sha256:e18f2438c89d9e6ae1e931448bcf525ba0ec4ebdc0888af836889e6ebcf70cd5" mediatype="application/vnd.oci.image.manifest.v1+json" size=586
```

Okay, so what just happened? Well, we first messed around with the root
filesystem of the image by creating a container (using [runc][runc], which you
can get from the official repositories if you're on openSUSE -- or from [this
repo][obs-vc-runc] if you want the bleeding-edge version) and doing some
updates and installing `strace`. Then we also went and manually modified the
root filesystem, without using a container. And finally we told `umoci` to
create a modified version of the `latest` image (tagged `new-latest`), with our
changes to `opensuse_bundle/rootfs` being converted into a diff layer and added
to the set of diff layers for the new image.

You might be wondering how `umoci repack` knew that the unpacked image came
from `opensuse:latest`. The answer is that `umoci unpack` stores some metadata
inside the bundle that allows it to infer what the extracted filesystem came
from.

Essentially `umoci repack` creates a **derivative** image from a base image,
using the changes made between an `unpack` and a `repack` as the set of
differences to package inside the new diff layer.

Note that `umoci repack` does not parse the `config.json` in the extracted
bundle and thus won't try to change the image configuration based on changes to
the runtime configuration. Use `umoci config` to make changes to the image
configuration.

We can now extract our new image and verify that our changes really were added
to the new image. Note that `umoci unpack` now extracts one more layer than it
did in the `umoci unpack` example.

```language-bash
% sudo umoci unpack --image opensuse:new-latest opensuse_bundle_updated
INFO[0000] parsed mappings                    map.gid=[] map.uid=[]
INFO[0000] unpack manifest: unpacking layer sha256:467db25190688bc9dc1d5fd6dbced1ac56f55d93a942987f448e62ad2614e46e  diffid="sha256:33e694f8e290bc896a8da5718854e81845fe79579f34704cf249ea457500134f"
INFO[0005] unpack manifest: unpacking layer sha256:9dab7d4b116241c5a3e30212f703a1f3ab786e05fae7555a4fbd63ab211df0ae  diffid="sha256:1a7f92d610b1a4ebb11a81a6d9dd011f82598afa92be7f9f9f517c4b77708f68"
INFO[0007] unpack manifest: unpacking config  config="sha256:f622b0688a120097a594222961a465cefe1b05543db1507b6036873e563c24d2"
% sudo runc run -b opensuse_bundle_updated ctr
sh-4.3# ls -la /selinux /a_new_file
ls: cannot access '/selinux': No such file or directory
-rw-r--r-- 1 root root 0 Nov 23 05:07 /a_new_file
sh-4.3# strace -V
strace -- version 4.10
sh-4.3# zypper lr
# | Alias      | Name       | Enabled | GPG Check | Refresh
--+------------+------------+---------+-----------+--------
1 | oss        | OSS        | Yes     | (r ) Yes  | Yes
2 | oss-update | OSS Update | Yes     | (r ) Yes  | Yes
sh-4.3# exit
```

While this might seem a bit magical, the way this is implemented is described
in more detail in [the implementation section](#implementation).

[oci-layers]: https://github.com/opencontainers/image-spec/blob/v1.0.0-rc2/layer.md
[runc]: https://github.com/opencontainers/runc
[obs-vc-runc]: https://build.opensuse.org/package/show/Virtualization:containers/runc

#### Image Configuration ####

The final major component of `umoci` is the ability to modify the configuration
of an image in a similar way to how `umoci repack` allows us to modify the set
of diff layers of an image. The interface is also fairly similar (though the
overall interface [will almost certainly be refined before `0.0.0` is
released][umoci-ux]). The options are self-explanatory, and will look familiar
if you've used [Dockerfiles][docker-dockerfiles] or have looked at the output
of [`docker inspect <some-image>`][docker-inspect].

It should be noted that changing the configuration of an image may cause the
runtime configuration generated by `umoci unpack` to change. We can use the
[`oci-runtime-tools generate` command][oci-runtime-tools-generate] to modify
the runtime configuration in order to avoid problems.

```language-bash
% umoci config --help
NAME:
   umoci config - modifies the image configuration of an OCI image

USAGE:
   umoci config [command options] --image <image-path>[:<tag>] [--tag <new-tag>]

Where "<image-path>" is the path to the OCI image, and "<tag>" is the name of
the tagged image from which the config modifications will be based (if not
specified, it defaults to "latest"). "<new-tag>" is the new reference name to
save the new image as, if this is not specified then umoci will replace the old
image.

CATEGORY:
   image

OPTIONS:
   --config.user value
   --config.memory.limit value  (default: 0)
   --config.memory.swap value   (default: 0)
   --config.cpu.shares value    (default: 0)
   --config.exposedports value
   --config.env value
   --config.entrypoint value
   --config.cmd value
   --config.volume value
   --config.label value
   --config.workingdir value
   --created value
   --author value
   --architecture value
   --os value
   --manifest.annotation value
   --clear value
   --tag value                  tag name
   --history.author value       author value for the history entry
   --history.comment value      comment for the history entry
   --history.created value      created value for the history entry
   --history.created_by value   created_by value for the history entry
   --image value                OCI image URI of the form 'path[:tag]'
```

The flags are documented in a much better fashion if you look at the man pages,
but the names should sound familiar to anyone who has used Docker (or ever
looked at the output of `docker inspect`). For example, here is an example of using
`umoci config` to change the default user and program that containers based on
this image will run:

```language-bash
% umoci config --image opensuse --config.user daemon:daemon --config.entrypoint="id" --config.cmd="-a"
INFO[0000] created new image  digest="sha256:7429ff32d9b769405620688d581d435a76608a187fe3cd3d8dcd11ed0c34379b" mediatype="application/vnd.oci.image.manifest.v1+json" size=426
% sudo umoci unpack --image opensuse opensuse_bundle_new
INFO[0000] parsed mappings                    map.gid=[] map.uid=[]
INFO[0000] unpack manifest: unpacking layer sha256:467db25190688bc9dc1d5fd6dbced1ac56f55d93a942987f448e62ad2614e46e  diffid="sha256:33e694f8e290bc896a8da5718854e81845fe79579f34704cf249ea457500134f"
INFO[0001] unpack manifest: unpacking config  config="sha256:1383884f41c46931acfb180609109848556d545a501bb4190468c1b8d4649d9a"
% sudo runc run -b opensuse_bundle_new ctr-build
uid=2(daemon) gid=2(daemon) groups=2(daemon)
```

Note that if we don't specify `--tag` then `umoci` will overwrite the old tag.

The `--clear` flag allows us to clear list-based configuration options.
Examples include the `--history`, `--config.volume`s, `--config.exposedports`,
and `--config.env`s.

[docker-dockerfiles]: https://docs.docker.com/engine/reference/builder/#/format
[docker-inspect]: https://docs.docker.com/engine/reference/api/docker_remote_api_v1.24/#/inspect-an-image
[oci-runtime-tools-generate]: https://github.com/opencontainers/runtime-tools#generating-an-oci-runtime-spec-configuration-files

#### Creating new Images ####

For completeness, `umoci init` and `umoci new` allow users to create an OCI
image from scratch. It allows us to create a brand-new OCI image layout
directory (without needing to pull a base image with `skopeo`). In addition, it
lets us create an "empty image" that we can use `umoci unpack` and `umoci
repack` on. This can be considered something like the Docker `scratch` image,
which contains no files or other metadata and is a blank slate for users to
craft an image from.

`umoci init` allows you to create a new OCI image layout (an image with no tags
or blobs inside it). `umoci new` allows you to create a new tagged image (which
has no layers and has a dummy configuration) that you can then modify to your
liking.

```language-bash
% umoci init --layout newimage
INFO[0000] created new OCI image layout  path=newimage
% umoci create --image newimage:new-tag
INFO[0000] creating new manifest  tag=new-tag
INFO[0000] created new image      digest="sha256:3b6ff1d61cd4d3ed25370607a48db9094c3bb1aa2de796e25fca51c08ef86a8b" mediatype="application/vnd.oci.image.manifest.v1+json" size=249
% sudo umoci unpack --image newimage:new-tag newbundle
INFO[0000] parsed mappings                    map.gid=[] map.uid=[]
INFO[0000] unpack manifest: unpacking config  config="sha256:a6d0e1ce7500a4b80cfed9779255a5ce2eccc8508de10ffc1825b4217f8588e1"
% ls -la newbundle/rootfs
total 0
drwxr-xr-x 1 root root   0 Jan  1  1970 .
drwxr-xr-x 1 root root 188 Nov 23 02:26 ..
```

From there, we can use `umoci repack` to create a new OCI image. You could even
imitate `docker import` by extracting a `tar` archive to the root filesystem
directory and then using `umoci repack`. The plan is that this is the first
step KIWI will use when creating an image.

#### Garbage Collection ####

And finally, for cleanliness reasons we have `umoci gc`. This command is very
similar to `git`'s `git gc` command. It removes all unused blobs within an OCI
image. It will not modify the contents or layers inside an OCI image, and is
perfectly safe (and **recommended**) to be run on an image before pushing the
image to production or to a registry.

The command-line is very simple, and we can see what I mean by "unused blobs"
if we delete a tag from the OCI image.

```language-bash
% umoci gc --layout opensuse
INFO[0000] GC: garbage collected 0 blobs
% umoci gc --layout opensuse
INFO[0000] GC: garbage collected 0 blobs
% umoci rm --image opensuse:old
% umoci gc --layout opensuse
INFO[0000] GC: garbage collecting blob  digest="sha256:51caf3bdd5f3e395f81e5454276b2dc2e15cf7a3047cc991f14b77e158356a3b"
INFO[0000] GC: garbage collecting blob  digest="sha256:b5b3627caa3d91971b1d701feec88c16e621d9f28122facfe22dd6ab57476221"
INFO[0000] GC: garbage collecting blob  digest="sha256:f041be4a5bbe4e191ec9e323f30a7e82ec4a1781e3376a885e8e5218bce6400c"
INFO[0000] GC: garbage collected 3 blobs
```

The reason a garbage collector is even required is because it is not safe for
`umoci` to start deleting blobs that we have replaced, because it is possible
for there to be multiple references to the same blob from different image tags.
To make life easy, rather than being clever about automatic garbage collection
the user just has to manually garbage collect blobs if they want to.

#### Converting Back to Docker ####

So, while all of the above is enough for OCI images, maybe you want to convert
your brand-new OCI image into a Docker image. `skopeo` to the rescue again! You
can push an OCI image to a Docker registry or to a local Docker daemon with the
following commands. You need to have a skopeo version later than
[`0.1.17`][skopeo-0.1.17] (which you can get from `Virtualization:containers`
if you're on openSUSE).

Also, note that pushing to a Docker registry requires you to have logged into
the registry with `docker login`. This is not intended (and [I've submitted a
bug report which is being fixed][skopeo-issue253]), but is necessary for the
moment.

```language-bash
% docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: cyphar
Password:
Login Succeeded
% # Push to the Docker Hub. You can also push to a local registry by setting
  # some other options. This requires a patched version of skopeo in order to
  # be able to use an OCI image as a source in a copy.
% skopeo copy oci:opensuse:new_latest docker://opensuse/amd64:latest
Getting image source signatures
Copying blob sha256:467db25190688bc9dc1d5fd6dbced1ac56f55d93a942987f448e62ad2614e46e
 0 B / 46.97 MB [--------------------------------------------------------------]
Copying blob sha256:eb9108dbdabef43ae694b955990153159c1cbd069c3e4ca5ac9ccdbdda0e4b09
 0 B / 23.48 MB [--------------------------------------------------------------]
Copying config sha256:a04dcb512d1d9fdd58667c4b2aaec916418f51130f9d8a9dd5b2dbed066e8c62
 0 B / 1003 B [----------------------------------------------------------------]
Writing manifest to image destination
Storing signatures
% # You can also push it to a local Docker daemon.
% skopeo copy oci:opensuse:new_latest docker-daemon:opensuse/amd64:latest
Getting image source signatures
Copying blob sha256:467db25190688bc9dc1d5fd6dbced1ac56f55d93a942987f448e62ad2614e46e
 17.16 MB / 46.97 MB [====================>------------------------------------]
Copying blob sha256:eb9108dbdabef43ae694b955990153159c1cbd069c3e4ca5ac9ccdbdda0e4b09
 46.97 MB / 46.97 MB [=========================================================]
Copying config sha256:a04dcb512d1d9fdd58667c4b2aaec916418f51130f9d8a9dd5b2dbed066e8c62
 0 B / 1003 B [----------------------------------------------------------------]
Writing manifest to image destination
Storing signatures
 1003 B / 1003 B [=============================================================]
```

Unfortunately, both of these methods require you to either have a Docker daemon
or Docker registry in your build infrastructure. The other option is to just
push to the Docker Hub, but that doesn't work as a generic build system. This
annoyed me too, so [I've started work on a PR][image-pr148] that will allow you
to convert an OCI image into a `docker load`-friendly Docker image (without
needing a daemon). This is also quite important for my "master plan" with [KIWI
integration](#applications), because we cannot run a Docker registry or Docker
daemon inside of the Open Build Service infrastructure. With that patch applied
(on top of the rest applied above), you can create `docker load`-friendly
archives.

```language-bash
% skopeo copy oci:opensuse:new docker-archive:opensuse.tar:opensuse/amd64:42.42
Getting image source signatures
Copying blob sha256:467db25190688bc9dc1d5fd6dbced1ac56f55d93a942987f448e62ad2614e46e
 37.88 MB / 46.97 MB [=============================================>-----------]
Copying blob sha256:eb9108dbdabef43ae694b955990153159c1cbd069c3e4ca5ac9ccdbdda0e4b09
 0 B / 23.48 MB [--------------------------------------------------------------]
Copying config sha256:a04dcb512d1d9fdd58667c4b2aaec916418f51130f9d8a9dd5b2dbed066e8c62
 0 B / 1003 B [----------------------------------------------------------------]
Writing manifest to image destination
Storing signatures
% docker load <opensuse.tar
Loaded image: opensuse/amd64:42.42
```

So, hopefully soon all of this code will be merged upstream so that you can
just grab a copy of `skopeo` from upstream to be able to do all of these
conversions from OCI images. I will patch the openSUSE version once these
patches get more review and are merged upstream.

[skopeo-0.1.17]: https://github.com/projectatomic/skopeo/releases/tag/v0.1.17
[image-pr148]: https://github.com/containers/image/pull/148
[skopeo-issue253]: https://github.com/projectatomic/skopeo/issues/253

### Implementation ###

There isn't much magic within `umoci`, but one of the cool features of `umoci`
is the that `umoci repack` knows what changes were made to the image root
filesystem after doing an `umoci unpack`. The way this works is using manifests.

If you look at the extracted bundle, you can see that there's an oddly-named
`.mtree` file in the bundle. And there is also an `umoci.json` file as well.

```language-bash
% ls -l opensuse_bundle
total 744
-rw-r--r-- 1 root root  24738 Dec 16 18:06 config.json
drwxr-xr-x 1 root root    128 Jan  1  1970 rootfs
-rw-r--r-- 1 root root 728474 Dec 16 18:06 sha256_80e063bbd4b40705b6d7d6e4d0b3f567376a9ac21ddf7aeb90ef9c7ca461c4e5.mtree
-rw-r--r-- 1 root root    324 Dec 16 18:06 umoci.json
```

`mtree` is [a fairly old FreeBSD utility][freebsd-mtree] that allows you to
generate a manifest for a directory hierarchy, so that you can verify that a
given directory matches the manifest. In particular, it means that we can see
what files have changed after the `.mtree` manifest was generated. However,
`mtree` is a FreeBSD utility and although it has [a port to
GNU/Linux][linux-mtree], the port is not very widely packaged and is not very
actively developed.

Luckily though, [Vincent Batts][vbatts] has been working on a [reimplementation
of `mtree` for GNU/Linux][go-mtree] written in Go and organised as a Go
library. I'd also like to thank him for telling me about this project, because
it's what inspired me to go solve this problem in the way I did. I've also been
contributing to `go-mtree` to [fix][go-mtree-pr81] [bugs][go-mtree-pr85], make
it [much more usable as a library][go-mtree-pr48], as well as adding [some cool
features][go-mtree-pr96] related to [rootless containers][rootless-containers].

So, when we call `umoci unpack` it will use `go-mtree` to figure out what files
need to be added to the diff layer of the image. You can see the changes
`go-mtree` will detect by using the command-line tool provided [by the
project][go-mtree] (which if you're on openSUSE can be installed [from my home
project on OBS][obs-home-cyphar]):

```language-bash
% sudo gomtree -p opensuse_bundle/rootfs -f opensuse_bundle/sha256_80e063bbd4b40705b6d7d6e4d0b3f567376a9ac21ddf7aeb90ef9c7ca461c4e5.mtree
"usr/bin": keyword "size": expected 5614; got 5682
"var/lib/rpm/Name": keyword "tar_time": expected 1479529762.000000000; got 1479826841.000000000
"var/log/zypper.log": keyword "size": expected 260934; got 549336
"var/tmp": keyword "tar_time": expected 1479529765.000000000; got 1479826841.000000000
"var/lib/rpm/Providename": keyword "tar_time": expected 1479529762.000000000; got 1479826841.000000000
"var/cache/zypp": keyword "size": expected 8; got 30
"var/cache/ldconfig": keyword "tar_time": expected 1479529762.000000000; got 1479826840.000000000
"tmp": keyword "tar_time": expected 1479529762.000000000; got 1479826829.000000000
"var/lib/zypp/AutoInstalled": keyword "size": expected 1400; got 1354
"var/lib/rpm/Packages": keyword "tar_time": expected 1479529762.000000000; got 1479826841.000000000
"lib64": keyword "tar_time": expected 1479529755.000000000; got 1479826840.000000000
"var/cache/zypp/solv": keyword "size": expected 14; got 40
"var/cache/zypp/solv/@System": keyword "tar_time": expected 1479529761.000000000; got 1479826841.000000000
"etc/ld.so.cache": keyword "size": expected 10989; got 11503
"etc/resolv.conf": keyword "size": expected 0; got 870
"var/lib/rpm/Sigmd5": keyword "tar_time": expected 1479529762.000000000; got 1479826841.000000000
"var/lib/zypp": keyword "size": expected 60; got 104
"var/log/zypp/history": keyword "tar_time": expected 1479529761.000000000; got 1479826841.000000000
"var/lib/rpm/Sha1header": keyword "tar_time": expected 1479529762.000000000; got 1479826841.000000000
"usr/lib64": keyword "size": expected 6024; got 6424
"var/lib/rpm/Requirename": keyword "tar_time": expected 1479529762.000000000; got 1479826841.000000000
"var/cache/zypp/solv/@System/cookie": keyword "tar_time": expected 1479529761.000000000; got 1479826841.000000000
"var/cache/zypp/solv/@System/solv.idx": keyword "size": expected 4097; got 3791
"var/lib/rpm/Basenames": keyword "tar_time": expected 1479529762.000000000; got 1479826841.000000000
"etc/zypp/repos.d": keyword "tar_time": expected 1447075872.000000000; got 1479826746.000000000
"run/zypp.pid": keyword "tar_time": expected 1479529761.000000000; got 1479826841.000000000
".": keyword "size": expected 128; got 122
"root/.bash_history": keyword "size": expected 0; got 142
"var/cache/zypp/solv/@System/solv": keyword "sha256digest": expected 6f15a4f936cb17364dad0d920874415b60541cc2ea1e8fecf18a103d044f43f7; got 6aef5835e85ae4257bcd71c7d5238d97ddb8759b52019b3e1bd82e37c37f76a6
"var/lib/rpm/Installtid": keyword "tar_time": expected 1479529762.000000000; got 1479826841.000000000
"etc": keyword "tar_time": expected 1479529767.000000000; got 1479826840.000000000
"var/lib/rpm/Obsoletename": keyword "tar_time": expected 1479529762.000000000; got 1479826841.000000000
"var/lib/rpm/Dirnames": keyword "tar_time": expected 1479529762.000000000; got 1479826841.000000000
"var/cache/ldconfig/aux-cache": keyword "size": expected 8851; got 9196
"var/lib/rpm/Group": keyword "tar_time": expected 1479529762.000000000; got 1479826841.000000000
% echo $?
1
```

`umoci` then uses this manifest comparison output to generate the root
filesystem diff layers. Then `umoci` modifies the image manifest and
configuration to include the new diff layer and creates the requested tag to
point to the new manifest.

In addition, `umoci.json` contains information about what `--image` flag was
used to extract the image initially (so `umoci repack` knows what image it has
to modify), as well as the `--uid-map`, `--gid-map` and `--rootless` flag
values. This is why `umoci repack` doesn't have any of those flags (they're all
read from `umoci.json` because `umoci repack` and `umoci unpack` would always
have be run with the same flags anyway).

[freebsd-mtree]: https://www.freebsd.org/cgi/man.cgi?mtree(8)
[linux-mtree]: https://github.com/archiecobbs/mtree-port
[vbatts]: https://twitter.com/vbatts
[go-mtree]: https://github.com/vbatts/go-mtree
[go-mtree-pr81]: https://github.com/vbatts/go-mtree/pull/81
[go-mtree-pr85]: https://github.com/vbatts/go-mtree/pull/85
[go-mtree-pr48]: https://github.com/vbatts/go-mtree/pull/48
[go-mtree-pr96]: https://github.com/vbatts/go-mtree/pull/96
[rootless-containers]: /blog/post/rootless-containers-with-runc
[obs-home-cyphar]: https://build.opensuse.org/package/show/home:cyphar/go-mtree

### Applications ###

All of this sounds well and good, but why bother? If we have `skopeo` and
Docker can already create images using `docker build`, why spend time
implementing an OCI-native tool? There are two main answers to this question.

First of all, it's quite important to the container community that we have
independent implementations of the Open Containers Initiative's specifications.
That way, we can "stress test" the specification and make sure that we avoid
the mono-culture that you get in other specifications (such as TLS and
OpenSSL).

The second reason is far more related to our requirements at SUSE. Put simply,
`docker build` is just not good enough. Sure, it works for most developers and
that's fine, but if you need to distribute images that have been built
reproducibly and are automatically rebuilt if a dependency has been updated,
then `docker build` doesn't really help you. To be fair, this is a fairly hard
problem to solve and `umoci` is a very small part of the solution (the rest of
the solution comes in the form of [KIWI][kiwi] and the [Open Build
Service][obs]).

So while `umoci` by itself is a fairly simple tool, and probably will only be
used by hardcore people like me that want to do crazy stuff like [run rootless
containers on university computing clusters][rootless-containers], there are a
lot of cool applications of `umoci` on the horizon.

It should therefore be no surprise that the main integration that SUSE is
planning on doing is integrating `umoci` into [KIWI][kiwi] and the [Open Build
Service][obs]. For those not familiar with these projects, I'll give you a
quick rundown.

[KIWI][kiwi] is a free software Appliance building tool. It can build VM
images, ISOs, raw disk images, and root filesystem archives based on a
specification of what packages and other metadata should be set in the image.
Currently we use the root filesystem archive functionality of KIWI to [generate
both openSUSE's and SUSE's Docker images][opensuse-kiwiconf] (we have a Jenkins
job that updates and [builds the images from the root
filesystem][opensuse-dockerfile]).  The important thing to note is that the
current system of building Docker images this way simply won't work if you want
to use KIWI to create different image profiles or multi-layered images.

The [Open Build Service (OBS)][obs] is a free software build and distribution
system used by openSUSE and SUSE for the building of packages for various
operating systems (such as the openSUSE distributions, SUSE Linux Enterprise
and others like Fedora, Arch Linux, etc). While mainly focused on packages, OBS
also allows for the building of virtual machine images with KIWI and will
publish said virtual machine images in repositories. And currently the way we
provide Docker images as packages to customers is by using KIWI to create the
image root filesystem, and using a fork of [this project][containment-rpm] to
generate an RPM that can be installed. OBS also includes dependency tracking,
and will rebuild an image (or package) if one of its dependencies (which
includes OS packages, and all of the dependencies of those packages) changes.
This is something quite unique to images generated by KIWI inside OBS (Docker
doesn't provide for this kind of workflow).

The one thing you might've noticed that all of the above sounds quite hacky.
And it is. Combine that with the fact that [Docker's official library of
images][docker-library] comes from [a repo full of `git` repos and commit
IDs][docker-library-repo], and you have a fairly complicated build system with
quite a lot of moving parts. In addition, KIWI doesn't provide us all of the
features we'd like to have in a Docker image (you can't set Docker metadata in
KIWI because KIWI just generates the root filesystem).

`umoci` can solve all of that, and potentially make the build system we have
far more reproducible. First off, integrating `umoci` support into KIWI means
that we get OCI image generation for free (something that KIWI could not
support otherwise). It also allows us to make our packaging of Docker images
(potentially) much simpler, with guarantees about the fact that all users will
get precisely the same image. And it could improve the state of the fracturing
between [`opensuse/amd64`][dockerhub-opensuse] (which is a Docker repository
that **we** control exclusively) and [`library/opensuse`][docker-library]
(which **Docker, Inc.** control).

The main concern with the fracturing between those two repositories is that
[because of the `official-library`'s build system][docker-library-repo],
`library/opensuse` will be rebuilt at random which results in the hashes not
matching `opensuse/amd64` (there's also a [separate issue about the source of
our images][twitter-signing] but that can also potentially be solved). Which is
quite bad. In my opinion, the best fix would be to switch to a setup where we
publish Docker images on OBS (**that are signed with the official openSUSE PGP
keys**), which are then just pushed to `opensuse/amd64` and then sourced in
`library/opensuse` (using `FROM opensuse/amd64@sha256:...` or similar). But
there's a lot of work that needs to happen before we can start solving this
problem.

Overall, I'm quite excited for what the general containers community will do
with `umoci`. Personally, I'm going to implement a `Dockerfile`-inspired build
system that creates OCI images using runC. And I'm also [currently implementing
rootless unpacking][umoci-rootless-unpacking] which should be quite useful for
[rootless containers][rootless-containers].

[opensuse-kiwiconf]: https://github.com/openSUSE/docker-containers
[opensuse-dockerfile]: https://github.com/openSUSE/docker-containers-build
[containment-rpm]: https://github.com/openSUSE/containment-rpm
[docker-library]: https://hub.docker.com/r/library/
[docker-library-repo]: https://github.com/docker-library/official-images
[dockerhub-opensuse]: https://hub.docker.com/r/opensuse/amd64/
[twitter-signing]: https://twitter.com/flavio_castelli/status/740274509689819136
[umoci-rootless-unpacking]: https://github.com/cyphar/umoci/issues/26

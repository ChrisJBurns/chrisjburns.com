+++ 
draft = false
date = 2023-05-20T19:02:25+01:00
title = "Inheritance vs Composition When Buiding Docker Images"
description = "Inheritance vs composition is a well-known object oriented design discussion that goes back decades. With the rise in containers over the last 5 years, is it time we start applying it to building containers?"
tags = ['docker', 'containerisation', 'software-enginering', 'security']
categories = ['docker', 'containerisation', 'software-enginering', 'security']
externalLink = ""
series = []
+++

Over the last 5 years, containers have become standard for companies that are running application workloads. With Gartner estimating that by 2026 more than 90% of global organizations will be running containerized applications in production (an increase from fewer than 40% in 2020), it's fair to say - even if you consider Serverless - containersation isn't going away anytime soon

Now without spending too much time detailing the background and history of what got us to the point of containersation and its uses, I want to cut straight to core of this blog post which aims to bring light to what I see as one of the biggest issues we face today with containersation - the image(s).

## To Base Image or Not To Base Image

So depending on the maturity and security of a platform and their container security strategy, the container images that are used on application workloads can vary - and vary they will. I've been on projects where there there was absolutely no container strategy whatsoever, so developers that were working on a standard Java / Spring microservice just used any JDK image that they find in their first Google search for "java docker images for Java 17" - you will be suprised on how much this still happens.

For other projects, I have seen more defined container strategies where they had base images in place that were purpose built for their application workloads. This meant the developers had one less decision to make as it was standardised, however, the decisions (and hassle) were pushed to those who had to maintain the base images because it was very often the case where the hierarchy of images all the way to their base were just an inheritance based mess that each at their own layer has new levels of complexities. Now [having base image](https://uploads-ssl.webflow.com/6228fdbc6c97145dad2a9c2b/624e2337f70386ed568d7e7e_chainguard-all-about-that-base-image.pdf) is a good thing, I'm definitely not arguing for an alternative. They allow standardisation, they often have reduced vulnerabilties because they _tend_ to be built with less _things_ in them, and they also contain all of the bits that you ideally want in your images. However, the same downside you get with inheritance in object-oriented languages still presents itself in Docker images which is you may end up with inherited Docker images that has things it doesn't need. This can happen because some may believe it was easier to make a base image with everything needed across all applications instead of 3 base images for the specific application types which hold only the things they need.

An example to make this clearer is, let's say there is a `BaseImage` that has `Thing1`, `Thing2`, `Thing3`, `Thing4` and `Thing5` inside of it, and there is `Application1`, `Application2` and `Application3` that all extend `FROM BaseImage`.

Now by default, all application images have all 5 things because they are baked into the base image that they all extend from. But what if `Application1` only needed `Thing1` and `Thing4`? Or if `Application3` only needs `Thing2` and `Thing5`? Well, they have them, but they also have the rest. So now you have an application image that has more things inside of it than it needs. Base images help solve the problem of packing the common things that derived images use, but they may end up having more than it needs which leads to a larger image size and attack surface.

A common way people navigate this problem is to have multiple base images that are purpose built for specific application types. So re-using the example above, we add addi 2 more base images: `BaseImage2` and `BaseImage3` and they contain the following:

- `BaseImage1` - `Thing1` & `Thing4`
- `BaseImage2` - `Thing2` & `Thing5`
- `BaseImage3` - `Thing3` & `Thing5`

The applications then look like the following:

- `ApplicationImage1` extends `FROM BaseImage1`
- `ApplicationImage2` extends `FROM BaseImage3`
- `ApplicationImage3` extends `FROM BaseImage2`
- `ApplicationImage4` extends `FROM BaseImage1`
- `ApplicationImage5` extends `FROM BaseImage3`

With this approach, each application image extends from a base image that only contains the things it needs. This keeps it as small as it can be without sacraficing on capabilities that are needed, and the attack surface is as small as it can be (there are caveats to this).

But with this, there are now more base images than before, which mean more maintenance, however most organisations still accept this because it gives structure and makes it more intuitive for engineers to use. There is a downside to this approach however, depending on your container workload estate, this can get out of control fast. If you've identified that you have more than 100 containerised application workloads, and upon investigation you realise that there are many types of combinations of images and the _things_ it needs, having specific base images for all permutations of application types and their required things would be absolute hell to manage and eventually it would become unmaintanable.

## Base Image Hell...

Up to this point, I've only focused on application workload images, these are actually not the most "flavoured" out of the types of images that organisations will end up using. The ones in my experience that have been the most customised are the pipeline task images. If you are running a containerised CI system that mandates that there must be an image specified before it runs any task, this is where the methods of building said images enter a completely wild west free for all. For example, if you wanted to build and test a Java / SpringBoot microservice, you'd have to have an image that has a JDK, Maven, the necessary config files and authentication credentials in order to pull dependencies from your internal repositories, add some certificates in there for the self-signed cert organisations of the world and so on and so on. Now do all of that again for a seperate image just because some teams use Gradle and not Maven, and they are using a different Java version. Now you can see where this get's very complicated and messy for organisations. There are so many types of images that are built by organisations for the most custom of needs it really does get quite insane.

The worst thing about all of this, is in those custom Dockerfiles there will be so many different methods of installation of binaries / dependencies etc. From `apt-get` to `yum` to `apk` to just curling a binary from Github and moving it so it's on the path ready to run, it really is the wild west. So not only does every Docker image look different, even if you've managed to create base-images out of this mess, most of the time there is a blurred line of what becomes a base image and what doesn't and you end up back in the same place.

Now something a lot of organisations don't think about is how do you patch/update all of this? If you're using `apk` or `apt-get` or whatever package manager to install things then it's _less_ of a headache to patch because it's definitely possible to build the images every week just so you get the latest versions of specific packages, but what about those Github `curl`'s? What about those people who didn't see the comms about the common `nodejs` build image and just created a new Dockerfile that have the installations of `nvm`, then the installation of `node` in them?

I hope the penny is starting to drop for those reading this because even if you do believe that you're on top of this because you have quarterly technical debt Sprints where you go through all images across your estate and upgrade everything that's in them, you still have to go through and understand each image and their respective binary installation methods - whether it's `curl`, package managers etc. I should also add that there is an increased security risk here because now the buid containers have curl and internet access - which is a dream for an attacker.

## But Chris, What Do I Do About All This?

Now although the picture painted is a bleak one, there are actually ways to somewhat make this managable and more secure. There are different things you can do on different levels. At the platform level, where platform engineers tend to have more customised tasks they need an easy way of being able to build the right image. Software engineers may have less of a customised need as platform engineers, however they will want to be able to use an image and install the required thing's inside of it easily.


- dependabots
- melange
- have simple to understand base images that really have the stuff that is common across all images and then add what you need via composition. but don't do it via curl or wget or `COPY`'s, put it in `apk`'s and pull them like any package.
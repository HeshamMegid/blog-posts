# Setting Up iOS CI Infrastructure with Buildkite and Anka

While we’re big fans of CircleCI and [use it for continuous integration](https://instabug.com/blog/how-we-automate-our-ios-workflow-at-instabug-using-circleci/) for both our iOS and Android SDKs, but as our team grows, our needs have changed. As a result, we decided to setup our own CI infrastructure. In this post, I’ll walk you through what motivated us to do it and that tools we used.

## Current Challenges

While CircleCI works great for us, we wanted to run our UI tests on physical devices instead of just simulators. This would help us find obscure UI and performance issues. This is currently not possible on CircleCI, and while there are many services that lets you do this, we preferred to utilize the extensive set of testing device we already had at our office.

In addition to that, we have an internal tool we have developed to do a combination of functional and stress testing on the SDK. This tools does nightly runs that take many hours to complete, and while running it on CircleCI is possible, it wouldn't be cost effective.

## Challenges

To help reduce flakiness and the eventual configuration/environmental drift, we decided from the get-go that builds need to run in an isolated environment, like Docker containers, rather than running them directly on the host machine. 

Unfortunately Docker doesn’t support macOS containers, so we decided to use [Veertu’s Anka](https://veertu.com/). Anka is a virtualization technology for macOS build on top of [Hypervisor.framework](https://developer.apple.com/documentation/hypervisor) that offers a container-like interface, and offers a simple command line interface to spin up and manage virtual machines.

The next challenge we faced was picking a frontend to schedule and manage our builds. We decided to stay away from Jenkins since it’s too fiddly to setup, and go for something more modern. 

After trying different tools, [Buildkite](https://buildkite.com/) seemed to best fit our needs. Buildkite offers a simple, modern UI with the ability to easily create sophisticated pipelines.

## How It All Works

The first step we needed to do was to create an Anka virtual machine that we can clone for each build. To make that process easier, we used HashiCorp’s [Packer](https://www.packer.io/). 

Packer lets us automate the virtual machine creation process out of [a bunch of confirmation files](https://github.com/HeshamMegid/anka-packer-images). Using Packer, we can create an image that has Xcode, fastlane, CocoaPods, and all the other essential tools for our build jobs while only taking the macOS installer downloaded from the App Store as input. 

The images are built as separate layers on top of each other. The first layer only installs Ruby, Homebrew and a few other basics, while the layer on top of it installs Xcode, making it much easier and faster to create a new image that has a new version of Xcode, based on the first layer. Also, since the whole process is based on configuration files, it can be source-controlled and versioned.

Once the virtual machine is created, it’s added to Anka’s Registry, which acts as a central repository of versioned virtual machine images. All build agents can pull images from this registry and clone them to run build pipelines. 

With our virtual machine images ready and the Buildkite agent running on a cluster of local Mac minis, the next step was to figure out how to provision a virtual machine for each build, and run the pipeline inside it. To do this, we used [Chef’s Buildkite plugin for Anka](https://github.com/chef/anka-buildkite-plugin). With Buildkite’s excellent support for plugins, we can simply set a step in our build pipelines to run using this plugin. 

The plugin will automatically do the following:
1. Create a clone of the specific virtual machine image.
2. Boot the virtual machine almost instantly, thanks to Anka's virtualization technology. 
3. Mount the checked out source code from the host into the virtual machine.
4. Run the pipeline step inside the virtual machine
5. Delete the cloned virtual machine when pipelines succeeds, fails, or gets cancelled.

```
                              ┌─────────────────────────────────────┐
                              │                              Hosts  │
                              │                            cluster  │
                              │                                     │                ┌────────────────────────────┐
                              │                                     │                │                    Current │
                              │    .─────.    .─────.    .─────.    │                │                     builds │
┌───────────┐                 │   ;  Mac  :  ;  Mac  :  ;  Mac  :   │  Run pipeline  │                            │
│           │  Push and pull  │   : mini  ;  : mini  ;  : mini  ;   │  steps inside  │ ┌────┐ ┌────┐              │
│   Anka    │     images      │    ╲     ╱    ╲     ╱    ╲     ╱    │      VMs       │ │VM 1│ │VM 3│              │
│ Registry  ◀─────────────────▶     `───'      `───'      `───'     ◀────────────────▶ └────┘ └────┘              │
│           │                 │                                     │                │ ┌────┐ ┌────┐              │
│           │                 │         .─────.    .─────.          │                │ │VM 2│ │VM 4│              │
└───────────┘                 │        ;  Mac  :  ;  Mac  :         │                │ └────┘ └────┘              │
                              │        : mini  ;  : mini  ;         │                └────────────────────────────┘
                              │         ╲     ╱    ╲     ╱          │
                              │          `───'      `───'           │
                              │                                     │
                              └──────────────────▲──────────────────┘
                                                 │
                                                 │
                                                 │
                                                 │
                                                 │ Manage pipelines
                                                 │ on host machines
                                                 │
                                                 │
                                                 │
                                         ┌───────▼───────┐
                                         │               │
                                         │               │
                                         │ buildkite.com │
                                         │               │
                                         │               │
                                         └───────────────┘
```

With this setup, we can run multiple pipeline steps simultaneously in separate virtual machines across several host machines. Since each machine only needs to run the BuildKite agent and the Anka command line interface, machines can be in different locations to make our setup more resilient to outages.

## Scalability

Scaling this infrastructure is as simple as adding new machines to the cluster of hosts, with the Anka CLI and the Buildkite agent running on them. We can also add remote machines from providers like [MacStadium](https://www.macstadium.com/) for pipelines that don’t require access to physical devices.

## Conclusion

Using Anka and Buildkite, we created a CI infrastructure that’s easily scalable, reduces flakiness/environmental drift, and gives us lots of flexibility to run build pipelines that take hours to complete, or pipelines that require access to physical devices.

While it took a significant time investment to build our own CI infrastructure, we think that the greater control and flexibility it gives us is well worth the effort.

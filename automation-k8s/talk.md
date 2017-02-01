# Steering an automation platform: K8s at wercker

Slide 1 (Introduction slide)

* Thank everyone for coming out 
    * today I'll be talking about how we (wercker) migrated over from our old stack over to kubernetes 
    * it will be 80% informative 10% bad jokes and 10% venting session of all the things that i had to put up with/do over my last 9 months or so at wercker.
* explain who you are
    * My name is Faiq Raza and I am a computer person over here at wercker in san francisco and i came all the way here just to talk at this meetup.... 
* explain what wercker is
    * show of hands, how many of you know what wercker is or what wercker does 
    * wercker is a container centric automation platform that helps developers build test and deploy their code
        * we support running any number of pipelines 
        * ranging from building code
        *  testing API-contracts between microservices, 
        * pushing containers to registries
        * deploying to schedulers.
    * we do all of this by running your pipelines inside of docker containers that you specify
* So what are some of the problems that we're trying to solve here?
    * How we need to run people's pipelines 
    * Finding compute space for these builds?
        * and more importantly, how do we make sure that the build loads are equally spread out for a consistent experience
    * Doing all of this inside of the docker container you specify

(Slide 2 )Started with CoreOS - did what we wanted (distributed init system with fleet) 

* The lowest barrier to entry - it ran all kiddie-pools  ( our internal job scheduler) as systemd units 
    and distributed the compute load across the cluster 
* Fleet was having problems 
    *  fleet would get out of sync with host systemd
    * overloading etcd- becoming inconsistent 
    * if any one thing went wrong fleet would stop working and the system would grind to a halt

(Slide 3) We really needed a more robust scheduler with a higher level scheduling abstractions

* We took a look at schedulers and realized there was not much of a choice for schedulers that had both extensibility, good out of the box features that make it easy to get started, and that had good community support. 

* We saw that CoreOS was starting to really get behind k8s and we had our eye on it for a while so we decided- why not? 
* And so we started prototyping a naive version of wercker to run on top of kubernetes 

(Slide 4) Building the initial prototype

* If we were going even consider kubernetes as a potential scheduler for us we would have to be able to run 2 things: 
    * our job runner called kiddie-pool - basically a service that reads from a job queue and starts jobs
    * wercker/wercker (open sourced) what actually does all the heavy lifting 
* Relationship between these two is kiddie-pool starts the wercker process similarly to as you might do it locally on your computer.
* With fleet we did this leveraging systemd units to keep the process alive- and some good old fashioned bash
* K8s doesn't have these low level abstractions- there are pods/jobs/services etc. so we had to shift over to thinking in this paradigm 
    * First thing we did was make the kiddie-pool deployment- that was pretty easy we just wrote up some yaml and ran the container as we always did
    * The next problem was a bit more complicated how do we get the actual build to start running on k8s
    * The solution we came up with was to create a new pod for every run that had all the relevant environment for checking out code, managing a cache, executing a job and uploading artifacts.
    *  We launch the pod, monitor its progress, and destroy it when the job is done.

(Slide 5)  Success baby

(Slide 6) The great migration

* Now that we proved that we were able to support our core product on k8s, it was time to start migrating all of our services over to k8s. 
    * And so the real work began- we started migrating over our web app/authentication service and started writing new services for our Saas product 
    * We started writing all of our new services using an application called blueprint. Which is a code scaffolding tool that Benno wrote. What it does is create the skeleton of a microservice using grpc, protobufs, and the swagger specification. It also exposes a REST gateway so that grpc services can be reached from normal web clients. 
    * Next, we started grouping all the machines in our infrastructure. We came up with peasants for where the pipelines are actually ran. Patricians are where our internal infrastructure lives which are things like kiddie-pool, the website, authentication services, our new service- vpp
    * We were doing great and then we started running into issues 

(Slide 7) Picture of servers on fire

(Slide 8) Pictures of github issues: 

* https://github.com/coreos/bugs/issues/965
* https://github.com/docker/docker/issues/20871
* https://github.com/docker/docker/issues/22124
* https://github.com/docker/docker/issues/22488
* https://github.com/docker/docker/issues/13885

We weren't sure what or where this problem was coming from the kernel? docker? kubernetes?  and quite frankly we did not have the time or the resources to figure out the problem 

(Slide 9) Have you considered turning it on then off again?

* We noticed that this only effected a portion of our servers- those of which that stuck around for more than 12 hours
* So we got to thinking- maybe we could actually just turn them off and on again
    * and so it was done- we wrote a systemd timer to run a bash script that killed and removed nodes from the cluster 
    * the ASG would spin back up new machines and the cloud-config/launch configuration would have to node rejoin the cluster
* We're able to do this because we treat all of our servers like cattle

(Slide 10) More problems with kuberenetes

* Kube still doesn't support cron scheduling
* this was vital to moving our web app over to kubernetes as it uses cron jobs for various tasks
* to fix this we wrote cronetes
    * its a daemon that takes kubernetes jobs and launches according to cron scheduling 
    * written in go!

(Slide 11) Life isn't too bad now

* I get paged a lot less 
* Kubernetes provides a pretty extensible API for us to write a lot of our business logic on top of  
* issues are a lot easier to debug 
    * No need to ssh in anywhere
    * Get to throw out all my systemd knowledge
    * A lot less stability issues- as kube is able to figure most things out for us 
    *  Upgrading the cluster has become way easier to do
        * for example, today we updated our cluster from 1.2.4 to 1.3.5 20 minutes before our CEO had a big meeting with investors and nothing went wrong!
    * Overall I learned that making the hop over to kubernetes wasn't so bad, sure we had some operational issues here and there- that's going to happen with no matter what kind of technology you use
* Stickers & t shirts at the end!! 



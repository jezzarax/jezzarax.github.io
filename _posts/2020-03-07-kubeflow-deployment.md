---
layout: post
title: A weekend of ops'ing (a rant)
---

Some wrap up of experience of past two days. I tried to automate deployment of [kubeflow](https://www.kubeflow.org) on to a self-built kubernetes cluster with ansible. 5/7 experience with partial success so far.

I generally like the idea of kubeflow, which on contrary to Sagemaker lets me work and test small things out while not being constantly connected, which feels great while on a train or a plane. I wanted to play more extensively with it so I could not only share my data-science related experience, but also understand if deploying it is easy enough to be handled by a product team and won't burden some dedicated in-company platform people.

### Good pieces

Ansible got much better since last time I went through its docs to get a general feeling on how things are working. The modules became more idiomatic, there's much less need to plug raw commands. Some interfaces became nicer (writing a list of packages to be installed with apt makes more sense than doing it through `with_items`). Asserts seemed a great idea, but after some time I found them more distracting than helpful. Inventory plugins are really cool. No more need for a separate magical scripts to get the list of host. Yet, if you're not a AWS or GCP user, you might get unpleasantly surprised with the quality of those, scaleway plugin works, but feels unpolished and needs experimentation to make things go as expected. In the end I managed to create a flow of playbooks creating me a set of machines and then provisioning those, which looks like one step before a pretty nice scaling automation.

Setting kubernetes to the point of it being able to run some pods was easy, kubeadm does all the work for you. I felt, it was the only easy part about kubernetes :)

### Challenging pieces

I understand, that most of it is about me not reading enough about some specific technology/library/component.

* Kubernetes documentation is visually nice. Content-wise it needs a lot of improvements and more thorough explanations. I don't like parts where you are supposed to take some command and run it just because you were told to. Because of that I had to go through pieces of cgroups documentation, which was not bad, installing kubernetes implies good knowledge of linux ecosystem, but the part about network add-ons is mind-breaking, as you're given 6 alternatives without a proper decision strategy to choose one over the others and next to it you're given a link with 10+ more, 1-liner description for each. Yet, going with this magic made things work.

<!--more-->

* Kubeflow provides you with istio if needed, but it seemed like a big thing, so I decided to go and understand at least a general idea of it. The documentation makes a better job explaining you how to install istio, rather than why would you need that in the first place. Some blog and reddit posts I found through google made a better job on that. In the end, I wonder if a problem of leaky abstraction is present in productive deployments, as the powerful beast is aimed to do too many things, from load-balancing (covering even business-oriented scenarios like A/B testing) to logs and security policies.
* Kubeflow documentation is not friendly as well. Again, external articles, and deeper investigations took place. 

### Questionnable pieces

* I really would like to see a smooth transition between a local and a cloud-based environment. Yet, the environment is that heavy, that 12GB of ram and a decent amount of CPU are required just to run the thing to the point my laptop starts melting. Kubernetes is all about microservices, where each microservice is supposed to be independent within its own domain. Even for a cluster with 5 nodes (4CPU*8GB ram) it took about half an hour to make all components up and running, so I could finally get my 2CPU*2GB ram jupyter environment.
* Even if a data scientist gets a 16 core*64GB ram laptop to run the thing, the complexity of the process is too high to make things smooth. In an ideal world, a data scientist is expected to be great at all three, statistics, coding and business domain, but in real world we have many brilliant data scientists who are not great with programming and hacking pieces, and dealing with VMs and kubernetes would make them sad.

### Current state & plans

* I can deploy kubeflow with some ansible automation and have some nice tea time while waiting it to run and be stable. Currently it requires `kubectl proxy` and is not reliably persistent. Both of those issues need to be fixed, more documentation will be read and investigations made.
* For the last 5 years I felt pretty comfortable with docker and docker swarm. Sadly enough this knowledge was not enough to hack through kubernetes. I feel a strong need to catch up to be able to work with kubernetes beyond `kubectl apply` level. Some people advised me `Kubernetes up and running` book will give it a try. I hope it'll make the "Challenging pieces" part of this post obsolete :)
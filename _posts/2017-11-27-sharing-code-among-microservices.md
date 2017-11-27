---
layout: post
title: Sharing Code Among Microservices
tags: Microservices
---

Building microservices will always lead to a situation where you have certain functionality to be implemented in multiple services. As developers, we try to apply the DRY mantra by building shared libraries which is fine but shared libraries could lead to more complexities depending how we build and share them.

## Inheritance
There are features we assume each service has to have such as request validation, user authorization or request throttling. Since each service has to provide them we start thinking of creating a class called `BaseService` that all services must **inherit** from.

Inheriting from base service might not be a good idea:
- It contradicts with the technology heterogeneity principle because each service should use the right technology to solve the business problem.
- Services development becomes coupled to the shared code so if one service requires upgrading the `BaseService` to fix a bug or add a new feature that could lead to updating all the services even if not required. Moreover, the upgrade would require large coordination between multiple teams.
- Teams starts to avoid changing the shared code because:
    - Fear to break other services.
    - Required effort to coordinate with all team to upgrade the library and rollout the change.
    - Testing the change might require running tests for all the services.
- Usually shared code is owned by all teams which means no one will be doing housekeeping tasks such as removing technical debt, upgrading the libraries, ...

## Composition
On the flip side, we could deal with that common code as a 3rd party packages (nuget, npm, ...) where each package targets a specific feature. The packages should be published and consumed by any service without any inheritance.

Publishing shared code as 3rd party package will help:
- Aligning with technology heterogeneity and will improve the coupling as each service might or might not consumer the service
-  Developers might be more encouraged to update the shared code because the package surface area is small which means small impact.
- Less time and effort to change because only consumer will decide when to upgrade to the latest version.

## Few things to consider for your packages:
- You should have versioning from day one, consider using [Semantic Versioning](https://semver.org).
- Please write a documentation on how to use the library streamline consuming and maintaining the library.
- Encourage everyone to add features and fix bugs in the shared library given that you follow the versioning process.
- Updating to the latest version of the package should be the consumer service responsibility.

## In Closing
In context of microservices, shared code for common functionalities has its place but I would recommend avoiding having base classes that all services must inherit from. Always deal with your shared code as 3rd party package.

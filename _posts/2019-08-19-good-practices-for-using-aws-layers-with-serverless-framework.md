---
layout: single
classes: wide
title:  "Tips about working with AWS layers using the Serverless framework"
date:   2020-05-05 00:00 -0500
tags:
  - serverless
  - Javascript
  - AWS Lambda
  - AWS Lambda layer
  - serverless layers
  - serverless offline
  - local testing 
  - software architecture 
 
categories: Serverless
excerpt: "Tips based on what I have learned after working during some months with AWS Lambda layers"
comments: true
---

Layers are a very powerful mechanism provided by AWS Lambda, that allows improving the performance during deployment by 
avoiding to package common code, over and over. This is particularly useful when using a Data Access Layers or utilities 
from lambda functions in different projects. Just by specifying the same ARN multiple functions can use the same version
of a set of dependencies and custom code.
I intend to give some notes about the things I learned the first months while working with AWS Lambda layers.

## Basic Tiers for a Serverless application

For making programs, we link modules, each of which handles a well defined requirement of a system and
needs to be reused in another runnable modules, namely applications. These modules are part of a particular layer of the 
architecture of a system. If we take a [Three-tier architecture](https://en.wikipedia.org/wiki/Multitier_architecture#Three-tier_architecture) 
as a reference we can understand easily how we may divide our module on these layers:

1. *Presentation tier*: Where the user or other services interact with. This is the face of this service to the outside 
world. E.g. the UI, a GraphQL or REST API, etc.
1. *Application tier* (business logic, logic tier, or middle tier). This is where business rules is shaped and controlled
by executing detailed operations.
1. *Data Access tier*: Makes possible the storage and access of information, which means that this is the boundary
of the application with the outside world, reflected in I/O operations.

When you create a Serverless application with layers, each of their parts may be logically grouped in one of these 
layers:

* `mailing_module`: Notifies via email using the SendGrid's API (Data Access tier).
* `reports_module`: Where is stored the logic and templates to generate the reports (Presentation tier).
* `endpoints`: Where the calls are controlled using a REST API (Application tier)
* `dm_module`: Contains the domain model of an application (Application tier).
* `sms_module`: Notifies via SMS messages using the Twilio API (Data Access tier).
* `promos_module`: Contains functionalities that allow creating promos and apply them to orders (Application tier).

You have already modularized your applications, before noticing that the Application tier is business biased, so it 
changes with the business logic. Most of the time, the presentation layer is subordinated to the particular logic of the 
application, so it changes altogether with the business logic. On the other hand, the layers related to the Data Access 
seem less prone to change along with the business logic, because they represent mostly technical requirements of the 
system, i.e. it is just a mean to get what the business logic wants.

### AWS Lambda restrictions

The Serverless' layers are a AWS Lambda feature, which allow to provide a more effective way to modularize its apps. 
It basically allows to:

* Deploy a reusable portion of your codebase independently, which makes the packaging of a function in most cases 
significantly smaller. Therefore, the overall time of deploying that app using `sls deploy -f <function>` is way shorter.
* The modularized content is versioned. This way each function of your app can be compliant with the version they were 
implemented with. This ensures that a new deployed change do not affect functions tha are not interested on such changes.
* Among the content you reuse, there is also included the external dependencies. Besides not expending time on 
calculating which `devDependencies` will be excluded during the deployment (which can delay a lot this process) you will 
also will be able to browse in your functions and see just the significant files.

Despite its advantages, working with layers is not an easy task, specially at the beginning. I remember that when I 
started learning how to create and use AWS Lambda layers with the Serverless framework it seemed to give more problems 
than solutions:

* Most of the time, all your functions want to make use of the last changes you made in your layer. So you have to see 
which is the later version using `sls info` in the root folder of the project of your layer (where its serverless.yml is). 
Then you will have to update the version of the layer for all the functions that use it.
* When deployed in AWS Lambda, the functions expect to see the code of the deployed layer in `/opt`, which does not 
matches the path where the lambda layer is in your local machine. Not too mention that some functions may use different 
versions of the same layer.

So when do I recommend to use layers?

### Criteria to use layers

Every time I created a Serverless layer, I asked my self these questions:

* Is this group of functionalities required by multiple AWS Lambda functions?
* Does this part of the application contains multiple resources?
* Do these functionalities belong to the Data Access Layer?

The more of these questions are true, the more reasons I will have to use a lambda layer. E.g.

* Internal modules, like `utils` or `error_handler`, that I need to reuse in many of my AWS lambda functions.
* I have a module to connect to the an external API, e.g. Twilio or Slack.
* A MongoDB module to provide a Data Access Layer to a MongoDB database, using the `mongodb` library. It would provide
custom needed features, like requiring that all connections are closed after making any call.
* A set of multiple external dependencies that are required by my functions: validation libraries, an status code 
library, etc.

Try always to use these layers in Serverless sub-projects:

1. In an independent repository, in a relative folder, e.g. `/layers/mysql_module`.
2. An independent project and integrated in the project with the functions using a set like:
    - [Lerna](https://github.com/lerna/lerna): A Javascript package manager.
    - [Github sub-modules](https://git-scm.com/book/en/v2/Git-Tools-Submodules).

> Note: In both these scenarios, the project containing the functions should be independent, i.e. it should possess it own 
`serverless.yml`, `package.json`, etc.

Although the second option is way more flexible, the first one provides some practical advantages:

* You have only one repository to deal with all the components of a project. It is supposed that the layers are not used 
by other projects.
* You can externalize configurations in a single project that can be used by the layers as well as the main project 
containing the functions.
* Less dependencies and configurations are required to make it work.

But you might be wondering how to work with layers in a development stage having into account that the path in our local 
machine differs to the one it will use when deployed. Let's see how to do it in the next section.

## How to use a AWS layer locally

One of the headaches you probably encountered at the beginning of using layers, was the fact that the path they use when
deployed are not the same to the one they have in our local environment. There is a workaround I used that proved to be 
effective.

Imagine that we have a `demoModule`, that is a layer, located in a sub-project. For any case, the best solution for 
specifying the root folder of the layer `demoModule` would be throw an environment variable:

```yml
 environment:
    NODE_PATH: "./:/opt/node_modules"
    DEMO_LAYER_ROOT: ${self:custom.stages.${self:provider.stage}.DEMO_LAYER_ROOT, '/opt'}
custom:
  stages:
    test:
      DEMO_LAYER_ROOT: ${opt:pwd, ""}/layers/demo_module
```

> Note: I consider layers to be the AWS Lambda implementations for modules in NodeJS. Therefore, as convention I 
> name my layers with the suffix `_module`.

For making it work we must specify in every command we run the option `--pwd=$(pwd)`. This will make the 
variable `pwd` to have the absolute path of the folder where we are running such command, e.g.

```
sls offline --pwd=$(pwd)
```

For this example, we supposed the layer was in a relative path of the project which is using it (the project of the 
functions). In case your layer is hosted in another repository, hence locally stored in another folder, then you can 
change the configuration to something like

```yml
 environment:
    NODE_PATH: "./:/opt/node_modules"
    DEMO_LAYER_ROOT: ${env.LOCAL_DEMO_LAYER_ROOT, '/opt'}
```

This second alternative is as simple as flexible. When you download the repository you might export the env variable
containing the absolute path of the layer project.

```bash
export LOCAL_DEMO_LAYER_ROOT = /absolute/path/of/the/project/of/your/demoModule/src
```


### Reference resources in a layer

In your `demoModule` project you have the codebase of your layer, with a directory structure like

```bash
> ls
/src
  /demo
    featuresA.js
/test
  ...
```

Let's imagine that in your functions app you want to make use of `featuresA` contained in your AWS Lambda function layer.
Then, during the development, while working locally, you must reference the env variable called `LOCAL_DEMO_LAYER_ROOT`, 
which encapsulates whether it is `/opt` or a local absolute path:

```javascript
const featuresA = require(process.env.DEMO_LAYER_ROOT + '/src/demo/featuresA');
````

This code in your local machine will be interpolated as:

```javascript
const featuresA = require('/path/to/the/project/of/your/demoModule/src/demo/featuresA');
````

But when deployed to the AWS Lambda, as the variable `LOCAL_DEMO_LAYER_ROOT` is not set, it will use `/opt` as default value.
Therefore, it will be interpolated as:

```javascript
const featuresA = require('/opt/src/demo/featuresA');
````

> Note: I personally do not like to put the codebase of the layers in the folder `/src`, because I prefer to make
> references like `demoModule/demo/featuresA`. Firstly, including `src` in the path of the client code is  redundant, 
> because all layer projects that follows this convention will have a folder with the same name. Also dangerous, because 
> whatever file you have in a `src` folder may be overwritten by another one with the same name, provided by other layer 
> that is referenced  by the client application. Therefore, better use a main package, with a canonical name in order 
> to avoid this kind of collisions, e.g. `demo`.

## Create an env variable to specify the latest version of a layer

When you specify the ARN in the layers section of a function, use an env variable

```yml
custom:
  DEMO_LAYER_ARN: arn:aws:lambda:us-east-2:212342342:layer:DemoModule:${env.DEMO_LAYER_VERSION_LATEST}

functions:
  myFunctionX:
    handler: src/xmodels/create-xmodel.handler
    events:
      - http: POST xmodelss
    package:
      include:
        - src/xmodels/create-xmodel.js
        - src/xmodels/xmodel.js
    layers:
      - ${self:custom.DEMO_LAYER_ARN}
```

Now, as we have a `LOCAL_DEMO_LAYER_ROOT` exported you could do something like

```bash
$ cd $LOCAL_DEMO_LAYER_ROOT
$ export DEMO_LAYER_VERSION_LATEST=$(sls info | grep demoModule: | grep -oE "[^:]+$")
```

You can also make use of the plugin [serverless-latest-layer-version](https://www.npmjs.com/package/serverless-latest-layer-version) 
that will do the same just by putting

```yml
custom:
  DEMO_LAYER_ARN: arn:aws:lambda:us-east-2:212342342:layer:DemoModule:latest
```

or a combination of the 2 strategies, where you can specify a particular version or by default peek the latest one.
Also remember to deploy the latest changes in your layers before you deploy the project with the functions.

## Do not create Serverless layers to handle business logic

Changes in your business logic also carries changes in the related codebase and sometimes in the routing of endpoints. 
Most of these changes are not even backward compatible. If you had them in the codebase of the functions app, then as 
the routing config is in the same project as the code which handles it the deployment is always consistent. But if you 
have the handler code in a layer, you must be sure that the configuration in the `serverless.yml` in your client app 
matches the right layer version. This way you are creating more opportunities to fail, specially if its source is hosted 
in a different project. This is why I rather not to handle biz-logic in layers, but use the routing configuration, the
handlers of the endpoints and the domain model classes in the same project:

```yml
  package:
    include:
      - src/xmodels/create-xmodel.js    # Lambda function handler
      - src/xmodels/xmodel.js           # Domain model object
```

Use AWS Lambda layers for technical (non-functional requirements) functionalities, e.g. a module for Neo4J called 
`neo4j_module`. The `serverless.yml` even allows you to have functions of the same project referencing different versions 
of AWS Lambda layers: You can even specify a version per function. This improves resiliency, by allowing functions that 
start failing with a new version of a required layer to go backward to use an stable one. In addition, layers cannot 
reference a particular dependant layer, but they will trust that the client app/functions will reference the right one.

## Do not extract your layer from your client app repository if possible

When we create a layer we think this is something profound that should be used by all the AWS lambda projects of the 
company. The sad thing is that we end up dealing with a second repository with no more than 3 files that is only used by 
the team of the app which uses it. For initial stages, I would recommend creating that layer in the sub-folder, 
e.g. `<approot>/layers/moduleA` and you will take advantage of mono-repos. If in the future, another project start 
requiring those features as well, then you can move this layer to another repository. Even if that new requirement 
appears, this strategy allows you to adapt without breaking the only project that was using at the beginning:

```bash
$ tree
/firstapp
/secondapp
/layers/
   /moduleA
   /moduleB
```


# Conclusions
It is a matter of preference the way you organize your layers in your app. You can do it in a sub-folder or in another 
independent repository. Eventually, the `serverless.yml` configuration can take care of arranging this mess. Nonetheless,
I recommend having the codebase of your layers in sub-projects of your project containing your AWS Lambda functions.  
Put in the AWS Lambda layers just the codebase that belongs to the Data Access Tier or you may be prone to critical 
errors more often. If you intend to use the latest version of a particular layer try to allow specifying it by using env 
variables or automatically let it fetch the latest available one. With environment variables you can create powerful
mechanisms to make your project configuration more versatile.

## See more
* [AWS Layers in Serverless](https://www.serverless.com/framework/docs/providers/aws/guide/layers/)
* [Microservices with Serverless & AWS Lambda Layers — A Walkthrough](https://medium.com/zoisays/microservices-with-serverless-aws-lambda-layers-a-walkthrough-b85ed35e0229)
* [Multitier architecture](https://en.wikipedia.org/wiki/Multitier_architecture)
* [Lerna](https://github.com/lerna/lerna)
* [Github sub-modules](https://git-scm.com/book/en/v2/Git-Tools-Submodules).

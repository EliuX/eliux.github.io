---
layout: single
classes: wide
title:  "How to run your serverless layers locally"
date:   2019-08-18 00:00 -0500
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
excerpt: "Good practices when working with AWS layers using the Serverless framework"
comments: true
---

## Modularizing a Serverless application

For making programs, we link modules, each of which handles a well defined requirement of a system that
needs to be reused in another runnable modules: apps. In Serverless they are named layers and as in any kind of module
they should have a well defined functional or non-functional responsibility, e.g.:

* mailing_module: Notifies via email using the SendGrid's API (non-functional).
* dm_module: Contains the domain models of a system (functional).
* sms_module: Notifies via SMS messages using the Twilio API (non-functional).
* promos_module: Contains functionalities that allows to create promos and apply them to orders (functional).

As you might have noticed, the modules that I categorized as functional are related to the business logic of an app, which
is something that defines the particular ways problems are handled. As the business logic changes depending how a company adapts
to solve their problems, these modules oftenly change much more than the non-functional modules. The later ones, can be used
in any other app, sometimes even if it has a very different biz logic because they give solution to a technological requirement.

The serverless layers are a feature created by AWS, which allows to its pioneer Function as a Service, AWS Lambda, to provide a
more effective way to modularize its apps. It basically allows to:

* Deploy a reusable portion of your codebase independently, which makes the packaging of a function in most cases significantly smaller.
Therefore, the overall time of deploying that app using `sls deploy -f <function>` is way shorter.
* The modularized content is versioned. This way each function of your app can be compliant with the version they were implemented with.
This ensures that a new deployed change do not affect functions tha are not interested on such changes
* Among the content you reuse, there is also included the node dependencies you add to them. Besides not expending time on calculating which
devDependencies will be excluded during the deployment (which can delay a lot) you will also will be able to browse in your functions and see
just the significant files.

Despite its advantages, working with layers is not an easy task, specially at the begining. The first time I learned how to create and
use serverless layers for AWS Lambda it seemed to give more problems that what it solved:

* Most of the time all your functions want to make use of the last changes you made in your layer. So you have to see which is the later version
using `sls info` in the folder of the project of your layer (when its serverless.yml is). Then you will have to update the version of the layer
for all the functions that use it.
* When deployed in AWS Lambda the functions expect to see the code of the deployed layer in `/opt` (relative path), which does not matches
the path where the lambda layer is in your local machine. Not too mention that some functions may use different versions of the same layer.

## How to use a AWS layer locally?

Imagine that we have a `demoModule`, that is a layer that is in our local machine. For any case, the best solution for specifying the
root folder of the layer `demoModule` would be throw an env variable:

```yml
 environment:
    NODE_PATH: "./:/opt/node_modules"
    DEMO_LAYER_ROOT: ${self:custom.stages.${self:provider.stage}.DEMO_LAYER_ROOT, '/opt'}
custom:
  stages:
    test:
      DEMO_LAYER_ROOT: ${opt:pwd, ""}/layers/demo_module
```

> Note: I consider layers to be in an implementation for AWS Lambda of what a module is. Therefore, as convention I name my layers with the suffix `_module`.

For this example we are supposing the layer is in a relative path of the project which is using it. For making it work we must specify in
every command we run the option `--pwd=$(pwd)`: This will make the variable `pwd` to have the absolute path of the folder where we are
running such command, e.g.

```
sls offline --pwd=$(pwd)
```

In case your layer is located in another repository you can change the configuration to something like

```yml
 environment:
    NODE_PATH: "./:/opt/node_modules"
    DEMO_LAYER_ROOT: ${env.LOCAL_DEMO_LAYER_ROOT, '/opt'}
```

It is as simple as flexible. When you download the repository of this layer in your `.env` you can have a line like this

```bash
export LOCAL_DEMO_LAYER_ROOT = /path/to/the/project/of/your/demoModule
```

and in your demoModule project you have the codebase of your layer, e.g.

```bash
> ls
/src
  /demo
    featuresA.js
/test
  ...
```

Then in the codebase of your app where you want to make use of `featuresA` contained in your AWS Lambda function layer,
you must reference the env variable called `LOCAL_DEMO_LAYER_ROOT`, instead of `/opt`:

```javascript
const featuresA = require(process.env.DEMO_LAYER_ROOT + '/src/demo/featuresA');
````

Which in your local machine will be traduced to:

```javascript
const featuresA = require('/path/to/the/project/of/your/demoModule/src/demo/featuresA');
````

But when deployed to the AWS Lambda, as the variable `LOCAL_DEMO_LAYER_ROOT` is not set, it will use `/opt` as default value.
Therefore, it will be translated to:

```javascript
const featuresA = require('/opt/src/demo/featuresA');
````

> Note: I personally do not like to put the codebase of the layers in the folder `/src`, because I prefer to make
> references like `demoModule/demo/featuresA`.

## Create an env variable to specify the latests version of a layer

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

There is also the plugin [serverless-latest-layer-version](https://www.npmjs.com/package/serverless-latest-layer-version) that will do
the same just by putting

```yml
custom:
  DEMO_LAYER_ARN: arn:aws:lambda:us-east-2:212342342:layer:DemoModule:latest
```

## Do not create Serverless layers to handle business logic

As the business logic changes you must change a lot of your codebase. Do you want to make an update of the domain model of your
app while your collegue is doing the same? The volume of version will make the app weird to handle. If you make a change in the
business model, maybe your partner would like to have that change also for working on to have it on the function he is using.
Most of these changes are not backward compatible. If you had them in the codebase of the app (Lambda functions) then with a simple
merge those changes would be applied. But as you have it in a lambda you must deploy the app and tell your partner
which version to use. You might be asking. What if I want to be compatible with an older version of that domain model?
Remember than when you reference a version of a layer you are copying like a snapshot of that layer into the codebase of your lambda 
function. So if the new version added a new required field and you are using an old version, that new field will not be added. 
That is why you should be up to date with the biz-model object: So you can be compliant with the current biz-logic always.
This is why I rather not to handle biz-logic in layers, instead you can include those biz logic files in your project:

```yml
  package:
    include:
      - src/xmodels/create-xmodel.js    # Lambda function handler
      - src/xmodels/xmodel.js           # Domain model object
```

Use AWS Lambda layers for non-functional related functionalities, like a module for Neo4J called `neo4j_module`. You might even
have functions of the same project referencing different versions of it without having having too much impact: If the new additions are a
security improvements all functions should update, but if they added a new feature it will impact those functions which do not require it.

## Do not extract your layer from your app repository if possible

When we create a layer we think this is something profound that should be used by all the AWS lambda projects of the company. The
sad thing is that we endup dealing with a second repository with no more than 3 files that is only used by the team of the app which
created it for it. Better start creating that layer in the folder `<approot>/layers/moduleA` and you will take advantage of being 
working with only one repo. If in the future another project requires the exact same capabilities of this project then you can extract it
to another repository, but if those needs change then you can adopt the same strategy but with those particular changes, without affecting
the project that is currently using it.

You can also use the approach of a mono-repo:

```bash
$ ls
/myapp
/moduleA
```

It is just a matter of preference.

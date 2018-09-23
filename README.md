
# README

## Overview

Sources for the airline simulator projects dataloader which performs a bunch of ETL stuff to get the necessary info requried to simulate the feeds to APR and other products in need of **_Real Time Feeds_**. Some of the initial ones are listed below and many more to follow. Currently uses Python's Bonobo as the ETL library to accomplish the pipelining.

* SSIM file to schedule message, flight legs
* Inventory initial data from Mongo DB to Postgress
* PNR raw initial data from Mongo to Postgress
* .....

## Setup

### Get and install pipenv

This project makes use of [pipenv](https://docs.pipenv.org/) for managing dependencies and environments.

To install pipenv (MacOS + Homebrew):

```shell
brew install pipenv
```

For more information on pipenv:

```shell
pipenv --help
```

### Install project deps

To install all dependencies for the project:

```shell
pipenv install --dev
```

This creates a virtualenv underneath but provides simplified management and interaction. To spawn a shell
that has access to all the dependencies, run:

```shell
pipenv shell
```

Similarly, running files in the project that require these dependencies can be run individually with:

```shell
pipenv run <the filename or command>
```

_Note: The VS Code Python extension looks for a `Pipefile` and, I believe, creates the virtualenv and installs dependencies. I have not tested this as I had the project initialized from the commandn line befor eopening in VS Code._

### General Guidelines for using Bonobo

[Bonobo] (http://docs.bonobo-project.org/en/master/index.html) is still a work in progress and is in alpha release and not a great deal of documentation is provided and does not provide good examples as well. For the purposes of this project, the following is the style that we are intending to follow for any of the data processors that would be used as part of the ETL.

Even though bonbo allows any callable to be used , we would restrict ourselves to use classes with **Bonobo's _Configurable_** Option . This would enable to use a skeleton for the processors that would be created for this project. 

Here is an example

```python
from bonobo.config import Configurable, ContextProcessor
import json


class ssim_file_reader(Configurable):
    start_yielding = False

    @ContextProcessor
    def initialise(self, context):

            yield context

    def __call__(self, context):
            lines = []

    def finalise(self):
        if self.sqlconnection is not None:
            self.sqlconnection.close()
```
Each class should have the methods _initialise_, _call_, _finalize_ .
#### Method Initialise:
This method is more like a initialisation that would be done at the run time level . You could initialize counters, get the necessary database connections, file streams that the processor would be using for any of the ETL oeprations for the particular node.

Also , the initialise method woudld initialise the object with the Nodeexecution context and override the bonobo's Finalise method with the finalise method written for the class.

```python
    # For the finalise call
        self.context = context
        context.input.on_finalize = self.finalise
```

 **Note** The method should be prefixed by the decorator **_ContextProcessor_** as this ensures the initialisation method is only called once during the execution of the node.

#### Method Call:
 This is the core processor method that would be called for every message passed in by the previous stage in the pipeline. 

#### Method Finalise:

 More like a destructor for a class, enusring to dispose the garbages that are no longer needed also perform _finally()_ steps for the node as such here.

 Could include closing db connections, closing files etc.

 **Note** : Should include the calling call as we are overriding the Bonobo's On_finalize method.

```python
         self.context.stop()
``` 



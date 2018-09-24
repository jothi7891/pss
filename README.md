
# README

## Overview

**Airline Simulator System** mimics a real time Airline Reservation system (a.k.a **PSS**) and could act as the source of messages to  APR and other products in the organization that would require airline reservation feeds. It would eventually be provided with the capability to produce all of the following feeds.

* SSM/ASM Schedule messages
* Real time Inventory updates
* PNR Creates/Updates
* E-Ticket Creates/Updates

The simulator system provides couple of interfaces through which the other products could subscribe the feeds. The **Phase 1** of the project deals with the providing inventory feeds real time and hence the remainder would be mostly focussed on that feed and the document would be updated appropriately with information regarding other feeds as and when they are developed.

* ***Rabbit MQ*** - The simulator system provides a configuration file where the user could provide the `exchange` and `queue` names and the inventory feeds would be published to those exchanges in the **`RabbitMQ`** . 
* ***API Call***  - The subscribing system could also make use of the **`API`** provided to get the latest info on the inventory for a particular flight.


##  Installation
First and foremost, it is necessary to get the simulator system up and running. The system is very light weight and could be very easily run in the same instance where the product in need of the feeds is being run.
### System Requirements

####Hardware

The following is the minimum level of hardware that airline simulator system requires. While it may be technically possible to run the simulator system on hardware older than this, it is neither recommended nor supported.

#####Linux

* Processor: Intel Core 2 Duo, AMD Athlon X2, or better
* Free Memory:
	* 1 GB for less than 1000 feeds/second
	* 2 GB or more for 1000 or more feeds/second
* System Storage: 
	* 100 MB for the Airline Simulator System
	* 2 GB or more recommended for storing the `Postgres` data.

#####Mac

* Processor: Intel Core 2 Duo or better
* Free Memory:
	* 1 GB for less than 1000 feeds/second
	* 2 GB or more for 1000 or more feeds/second
* System Storage: 
	* 100 MB for the Airline Simulator System
	* 2 GB or more recommended for storing the `Postgres` data.

#####Windows

* Currently, **`Windows O/S`** is not supported for running the airline simulator system.

####Software
The core of the product is written in python and the package dependencies are managed through the usage of `pipenv`. For convenience, the entire software package and its related dependencies like data initialization are managed through `DOCKER` and the `docker_compose.yml` is available in the root directory of the repository.

The following commands should get the airline simulator system up and running.


```shell
git clone git@github.com:jothi7891/pss.git
docker-compose up
```

##Usage:

As described earlier, the feeds could be obtained through either subscribing to a Rabbit queue or calling an API for acquiring data as needed.

###Rabbit-MQ:

The only input needed from the user would be to setup the `Rabbit` Connection parameters , if they ever wanted to override the defaults provided and set up the right `exchange` and `queue` names in the `config.json` present in the root directory. The connection parameters are defined in the `.env` file. The output format would be `json` message in the format defined below in the `Inventory Examples` section

**.env**

```
RABBIT_MQ_HOST=localhost
RABBIT_MQ_USER=guest
RABBIT_MQ_PASSWORD=guest
RABBIT_MQ_PORT=5672
```

**Config.json**

```json
{
    "RABBIT_INV_EXCHANGE": "Inventory"
    "RABBIT_INV_QUEUE":"Inventory"
}
```

###API:

The Inventories API allows you to retrieve inventory information for flights with availability of seats on each cabin(First class, Business Class and Economy Class if they are available). The result would be a single flight or a list of flights based on the parameters passed in the query. The 

```api
https://api.pssmock.com/v1/inventories/output?parameters
```
where output can have the possible values of

* `json` (recommended) indicates output in JavaScript Object Notation (JSON)
* `xml` indicates output as XML. XML is not currently supported and would be available later for use.

All parameters are separated using the ampersand(`&`) character

####Required Parameters:
* `flightnumber` - the `flightnumber` for which the inventory information should be retrieved. It could be either in `string` or `integer` format.
* `date` - the `departure date` of the flight. It could be in any standard date format as supported by `RFC 3339`

####Optional Parameters:
* `airport` - 3 character `IATA` airport code. Its the `departure airport` of the flight that you are interested in. This is very helpful in filtering out a particular flight leg from multi-leg flight.

###Inventory API Request Examples

####Example 1: Retrieve flight inventory for flight number *0008* for departure date *2019-01-01*

#####Request:
```api
https://api.pssmock.com/v1/inventories/json?flightnumber=0008&date="2019-01-01"
```
####Example 2: Retrieve flight inventory for flight number *0009* for departure date *2019-01-01* departing from airport *SFO*

#####Request:
```api
https://api.pssmock.com/v1/inventories/json?flightnumber=0009&date="2019-01-01"&airport="SFO"
``` 

###Inventory Response:

####Example 1:
Below is a sample inventory response that is produced as a response to the `API` below call.

```api
https://api.pssmock.com/v1/inventories/json?flightnumber=906&date="2018-06-01"
```
It's also the same message that gets published to the `Rabbit-MQ` as well 

```json
    "results" : [{
    "airline_designator": "ST",
    "flight_number": 906, 
    "departure_city": "DAV",
    "departure_date": "20180601",
    "inventory" : [
	    {"cabin": "First", "seats_available": 5,"seats_sold": 1},
	    {"cabin": "Business", "seats_available": 16 ,"seats_sold": 4},
	    {"cabin": "Economy", "seats_available": 150,"seats_sold": 0}
    ]
    }],
    "status" : "OK"
```

The `results` json is an list even if the result contains a single flight just to maintain consistency for the cases returning multiple flights as a response.
Each `flightInventory` object contains the following elements

* `airline_designator` - 2 character `IATA` airline code( Ex. AA for American Airlines, BA for British Airways)
* `flight_number` - an `integer` representing a flight number and it would be always greater than `0` and less than `9999`.
* `departure_city`- 3 character `IATA` airport code and its the airport code from which the flight is departing.
* `departure_date`- departure date of the flight sent in the format `YYYYMMDD`.
* `inventory`- a list containing all cabins present in the flight with the inventory information associated with it.
	* `cabin`- can contain one of the following values . `First`, `Business` or `Economy`
	* `seats_available` - an `integer` representing the number of seats available in the compartment.
	* `seats_sold` - an `integer` representing the number of seats sold in the compartment.

#####Status Code:
The `"status"` field within the response object contains the status of the request and may contain debugging information.

* `OK` indicates that no errors occurred.
* `NO_RESULTS` indicates the search was successful but no results were returned.
* `INVALID_REQUEST` indicates an error in the parameter that is passed along with the query.

####Example 2: 	
This example illustrates a case where multiple flight legs would be returned. Flight 905 is multi-leg flight from `ATL-MSW-EWR`.

```json
    "results" : [{
    "airline_designator": "ST",
    "flight_number": 905, 
    "departure_city": "ATL",
    "departure_date": "20180601",
    "inventory" : [
	    {"cabin": "First", "seats_available": 5,"seats_sold": 1},
	    {"cabin": "Business", "seats_available": 16 ,"seats_sold": 4},
	    {"cabin": "Economy", "seats_available": 150,"seats_sold": 0}
    ]
    },
    {
    "airline_designator": "ST",
    "flight_number": 905, 
    "departure_city": "MSW",
    "departure_date": "20180601",
    "inventory" : [
	    {"cabin": "First", "seats_available": 5,"seats_sold": 1},
	    {"cabin": "Business", "seats_available": 16 ,"seats_sold": 4},
	    {"cabin": "Economy", "seats_available": 150,"seats_sold": 0}
    ]
    }],
    "status" : "OK"
```
###Developer Notes:
For developers who wish not to use `Docker` and would like to play around with the product, the following steps would be helpful. Please skip this , if you have already got the docker running with the `docker_compose.yml`.

##### Get and install pipenv

This project makes use of [pipenv](https://docs.pipenv.org/) for managing dependencies and environments.

To install pipenv (MacOS + Home-brew):

```shell
brew install pipenv
```

For more information on pipenv:

```shell
pipenv --help
```

##### Install project dependencies

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

### General Guidelines for using Bonobo

The core of the project is written in [Bonobo] (http://docs.bonobo-project.org/en/master/index.html), a **WIP** light-weight `python` based ETL tool. For the purposes of this project, the following is the style that we are intending to follow for any of the data processors that would be used as part of the ETL. So if you ever want to modify/add a data-processor in the pipeline, please follow the guidelines defined below.

Even though bonobo allows any callable to be used , we would restrict ourselves to use `classes` with **Bonobo's Configurable** Option . This would enable to use a skeleton for the processors that would be created for this project. 

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
Each class should have the methods *initialise*, *call*, *finalize*.

#### Method Initialise:
This method is more like a initialisation that would be done at the run time level . You could initialize counters, get the necessary database connections, file streams that the processor would be using for any of the ETL operations for the particular node.

Also , the initialise method would initialise the object with the Nodeexecution context and override the bonobo's Finalise method with the finalise method written for the class.

```python
    # For the finalize call
        self.context = context
        context.input.on_finalize = self.finalise
```

 **Note** The method should be prefixed by the decorator ***ContextProcessor*** as this ensures the initialisation method is only called once during the execution of the node.

#### Method Call:
 This is the core processor method that would be called for every message passed in by the previous stage in the pipeline. 

#### Method Finalise:

 More like a destructor for a class, ensuring to dispose the garbage that are no longer needed ,also performs *finally()* steps for the node such as closing db connections, closing file-handlers etc.

 **Note** : Should include a call to Bonobo's *on_finalize()* method.

```python
         self.context.stop()
``` 



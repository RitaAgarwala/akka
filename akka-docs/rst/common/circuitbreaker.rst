.. _circuit-breaker:

###############
Circuit Breaker
###############

==================
Why are they used?
==================
A circuit breaker is used to provide stability and prevent cascading failures in distributed
systems.  These should be used in conjunction with judicious timeouts at the interfaces between
remote systems to prevent the failure of a single component from bringing down all components.

As an example, we have a web application interacting with a remote third party web service.  
Let's say the third party has oversold their capacity and their database melts down under load.  
Assume that the database fails in such a way that it takes a very long time to hand back an
error to the third party web service.  This in turn makes calls fail after a long period of 
time.  Back to our web application, the users have noticed that their form submissions take
much longer seeming to hang.  Well the users do what they know to do which is use the refresh
button, adding more requests to their already running requests.  This eventually causes the 
failure of the web application due to resource exhaustion.  This will affect all users, even
those who are not using functionality dependent on this third party web service.

Introducing circuit breakers on the web service call would cause the requests to begin to 
fail-fast, letting the user know that something is wrong and that they need not refresh 
their request.  This also confines the failure behavior to only those users that are using
functionality dependent on the third party, other users are no longer affected as there is no
resource exhaustion.  Circuit breakers can also allow savvy developers to mark portions of
the site that use the functionality unavailable, or perhaps show some cached content as 
appropriate while the breaker is open.

The Akka library provides an implementation of a circuit breaker called 
:class:`akka.pattern.CircuitBreaker` which has the behavior described below.

================
What do they do?
================
* During normal operation, a circuit breaker is in the `Closed` state:
	* Exceptions or calls exceeding the configured `callTimeout` increment a failure counter
	* Successes reset the failure count to zero 
	* When the failure counter reaches a `maxFailures` count, the breaker is tripped into `Open` state
* While in `Open` state:
	* All calls fail-fast with a :class:`CircuitBreakerOpenException`
	* After the configured `resetTimeout`, the circuit breaker enters a `Half-Open` state
* In `Half-Open` state:
	* The first call attempted is allowed through without failing fast
	* All other calls fail-fast with an exception just as in `Open` state
	* If the first call succeeds, the breaker is reset back to `Closed` state and the `resetTimeout` is reset
	* If the first call fails, the breaker is tripped again into the `Open` state (as for exponential backoff circuit breaker, the `resetTimeout` is multiplied by the exponential backoff factor)
* State transition listeners: 
	* Callbacks can be provided for every state entry via `onOpen`, `onClose`, and `onHalfOpen`
	* These are executed in the :class:`ExecutionContext` provided. 
* Calls result listeners:
    * Callbacks can be used eg. to collect statistics about all invocations or to react on specific call results like success, failures or timeouts.
    * Supported callbacks are: `onCallSuccess`, `onCallFailure`, `onCallTimeout`, `onCallBreakerOpen`.
    * These are executed in the :class:`ExecutionContext` provided.

.. image:: ../images/circuit-breaker-states.png


========
Examples
========

--------------
Initialization
--------------

Here's how a :class:`CircuitBreaker` would be configured for:
  * 5 maximum failures
  * a call timeout of 10 seconds 
  * a reset timeout of 1 minute

~~~~~
Scala
~~~~~

.. includecode:: code/docs/circuitbreaker/CircuitBreakerDocSpec.scala
   :include: imports1,circuit-breaker-initialization

~~~~
Java
~~~~

.. includecode:: code/docs/circuitbreaker/DangerousJavaActor.java
   :include: imports1,circuit-breaker-initialization

------------------------------
Future & Synchronous based API
------------------------------

Once a circuit breaker actor has been intialized, interacting with that actor is done by either using the Future based API or the synchronous API. Both of these APIs are considered ``Call Protection`` because whether synchronously or asynchronously, the purpose of the circuit breaker is to protect your system from cascading failures while making a call to another service. In the future based API, we use the :meth:`withCircuitBreaker` which takes an asynchronous method (some method wrapped in a :class:`Future`), for instance a call to retrieve data from a database, and we pipe the result back to the sender. If for some reason the database in this example isn't responding, or there is another issue, the circuit breaker will open and stop trying to hit the database again and again until the timeout is over.

The Synchronous API would also wrap your call with the circuit breaker logic, however, it uses the :meth:`withSyncCircuitBreaker` and receives a method that is not wrapped in a :class:`Future`.

~~~~~
Scala
~~~~~

.. includecode:: code/docs/circuitbreaker/CircuitBreakerDocSpec.scala
   :include: circuit-breaker-usage

~~~~
Java
~~~~

.. includecode:: code/docs/circuitbreaker/DangerousJavaActor.java
   :include: circuit-breaker-usage

.. note::

	Using the :class:`CircuitBreaker` companion object's `apply` or `create` methods
	will return a :class:`CircuitBreaker` where callbacks are executed in the caller's thread.
	This can be useful if the asynchronous :class:`Future` behavior is unnecessary, for
	example invoking a synchronous-only API.

.. note::
	
	There is also a :class:`CircuitBreakerProxy` actor that you can use, which is an alternative implementation of the pattern.
	The main difference is that it is intended to be used only for request-reply interactions with another actor. See :ref:`Circuit Breaker Actor <circuit-breaker-proxy>`

--------------------------------
Control failure count explicitly
--------------------------------

By default, the circuit breaker treat :class:`Exception` as failure in synchronized API, or failed :class:`Future` as failure in future based API.
Failure will increment failure count, when failure count reach the `maxFailures`, circuit breaker will be opened.
However, some applications might requires certain exception to not increase failure count, or vice versa,
sometime we want to increase the failure count even if the call succeeded.
Akka circuit breaker provides a way to achieve such use case:

   * `withCircuitBreaker`
   * `withSyncCircuitBreaker`

   * `callWithCircuitBreaker`
   * `callWithCircuitBreakerCS`
   * `callWithSyncCircuitBreaker`

All methods above accepts an argument ``defineFailureFn``

~~~~~
Scala
~~~~~

Type of ``defineFailureFn``: ``Try[T] ⇒ Boolean``

This is a function which takes in a :class:`Try[T]` and return a :class:`Boolean`. The :class:`Try[T]` correspond to the :class:`Future[T]` of the protected call. This function should return ``true`` if the call should increase failure count, else false.

.. includecode:: code/docs/circuitbreaker/CircuitBreakerDocSpec.scala
   :include: even-no-as-failure

~~~~
Java
~~~~

Type of ``defineFailureFn``:  :class:`BiFunction[Optional[T], Optional[Throwable], java.lang.Boolean]`

For Java Api, the signature is a bit different as there's no :class:`Try` in Java, so the response of protected call is modelled using :class:`Optional[T]` for succeeded return value and :class:`Optional[Throwable]` for exception, and the rules of return type is the same.
Ie. this function should return ``true`` if the call should increase failure count, else false.

.. includecode:: code/docs/circuitbreaker/EvenNoFailureJavaExample.java
   :include: even-no-as-failure

-------------
Low level API
-------------

The low-level API allows you to describe the behaviour of the CircuitBreaker in detail, including deciding what to return to the calling ``Actor`` in case of success or failure. This is especially useful when expecting the remote call to send a reply. CircuitBreaker doesn't support ``Tell Protection`` (protecting against calls that expect a reply) natively at the moment, so you need to use the low-level power-user APIs, ``succeed``  and  ``fail`` methods, as well as ``isClose``, ``isOpen``, ``isHalfOpen`` to implement it.	

As can be seen in the examples below, a ``Tell Protection`` pattern could be implemented by using the ``succeed`` and ``fail`` methods, which would count towards the :class:`CircuitBreaker` counts. In the example, a call is made to the remote service if the ``breaker.isClosed``, and once a response is received, the ``succeed`` method is invoked, which tells the :class:`CircuitBreaker` to keep the breaker closed. If on the other hand an error or timeout is received, we trigger a ``fail`` and the breaker accrues this failure towards its count for opening the breaker.

.. note::

	The below examples doesn't make a remote call when the state is `HalfOpen`. Using the power-user APIs, it is your responsibility to judge when to make remote calls in `HalfOpen`.


~~~~~~
Scala
~~~~~~

.. includecode:: code/docs/circuitbreaker/CircuitBreakerDocSpec.scala
   :include: circuit-breaker-tell-pattern


~~~~
Java
~~~~

.. includecode:: code/docs/circuitbreaker/TellPatternJavaActor.java
   :include: circuit-breaker-tell-pattern


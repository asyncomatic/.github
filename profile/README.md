## Async-o-matic
Async-o-matic is a workflow management framework specifically designed to support and facilitate asynchronous test 
execution patterns. 
- [Asynchronous Test Execution](#asynchronous-test-execution)
- [Architecture](#architecture)
- [Annotations](#annotations)
  - [@Schedule](#schedule)
  - [@Retry](#retry)
- [Take a Test Drive](#take-a-test-drive)

&nbsp;

### Asynchronous Test Execution
Lower-level tests, such as unit and integration tests, typically follow a synchronous execution pattern: each test is 
treated as a single unit of execution, defined in a single test method, executed from start to finish. 
Any pauses required as part of the test are normally of short duration (milliseconds to seconds) and easily 
accommodated within the test method: networking code will block on an API call until a response is 
received; a polling loop can be used to check and wait until a specific condition is met; or (as a last resort) sleep 
statements can be used to pause execution as needed.

This synchronous execution pattern breaks down when we begin to look at higher-level tests, tests that may require 
longer, non-trivial pauses (minutes, hours, even days). Consider, for example, a case where we need to confirm that our 
data processing system is correctly handling data from various external sources. We might begin by defining, at a high 
level, our test scenario as follows:

- ingest a known external data set into the data processing system; and
- analyze the subsequent result set to verify it is correct

Unfortunately, intensive data processing is never instantaneous, so in all likelihood we need to update our scenario to 
allow for non-trivial processing time:

- ingest a known external data set into the data processing system;
- **_pause to allow time for processing to occur;_** and
- analyze the subsequent result set to verify it is correct

Implemented as a traditional, synchronous test case (as a JUnit test, for example), we could write something 
similar to the following:

<pre>
    
    @Test
    public void dataProcessiongTest() {

        // ingest a known external data set into the data processing system
        ingestData();

        // pause to allow time for processing to occur (30 seconds, more may be required)
        Thread.sleep(30 * 1000);

        // analyze the subsequent result set to verify it is correct
        assertValidResultSet();
    }

</pre>
&nbsp;

Executed by itself, this test would take slightly more than 30 seconds to execute, assuming (for the sake of argument) 
that ingestion and validation are relatively quick operations. Let us now assume that we have numerous data sets we need 
to test, let's say 1000. Executing 1000 iterations of our test as a test suite, using a single worker, would take 
over 8 hours! Parallelization (via additional workers) would help reduce the overall execution time, but does nothing 
to alleviate the inherent inefficiency of tests that spends most of their execution time sleeping. This synchronous 
approach is simply not scalable as the number of tests, and the delays required, increase.

What if we could treat ingestion and validation (or any parts of a test, for that matter) as independent steps, each to 
be scheduled individually? Processing delays could automatically be accounted for in how the steps are scheduled: the 
validation step could be scheduled to run 30 seconds after the ingestion step completes, allowing sufficient time for 
processing while removing the need for the worker to block execution and allowing it to perform other tasks.

Async-o-matic takes a novel approach at addressing this challenge. Async-o-matic provides a framework for treating a 
test as a workflow, implemented as a class (rather than a method), with the class' methods as the workflow's steps. 
This framework provides a custom scheduler that allows for the individual steps to be scheduled independently, 
each to be executed at the appropriate time. The entire workflow is defined through simple Java annotations, and any 
state that needs to be maintained is automatically passed from method to method as the test progresses.

Using this framework, our previous example could be implemented as follows:

<pre>

class DataProcessingTest {

    @Schedule(method = "assertValidResultsSet", delay = 30, units = Delay.SECONDS)
    public void ingestData(TestState state) {
        // Ingest data into pipeline
    }

    public void assertValidResultSet(TestState state) {
        // Retrieve and validate result set
    }
}

</pre>
&nbsp;

Execution time for a single test? Slightly more than 30 seconds (as before). 

Execution time for a suite of 1000 similar scenarios? Slightly more than 30 seconds (an exaggeration, but meant to 
draws attention to the fact that at no point in time during execution of the entire suite does the executor need to 
sleep). 

Total savings in execution time? 1000 * 30 seconds!

&nbsp;

### Architecture
![](https://github.com/asyncomatic/.github/blob/main/profile/high_res.png?raw=true)

&nbsp;

### Annotations
Async-o-matic scheduling is managed via two simple Java annotations: ```@Schedule``` and ```@Retry```.

&nbsp;
#### *@Schedule*
The ```@Schedule``` annotation allows for a subsequent test method to be scheduled upon completion of the annotated 
method. The parameters available to the ```@Schedule``` annotation, and any default values, are shown below:

<pre>

public @interface Schedule {

    String method();                                            // required

    long delay() default 0;                                     // optional (default: no delay interval)
    int units() default Delay.NONE;                             // optional (default: no delay units)
    
    int condition() default Condition.ANY;                      // optional (default: schedule unconditionally)

}

</pre>
&nbsp;

In its most basic form, the ```@Schedule``` annotation specifies the subsequent **_method_** to execute 
following completion of the annotated method, without delay:

<pre>   
    
    @Schedule(method = "assertValidResultSet")
    public void ingestData(State state) {

        // on "ingestData" completion:
        //    - execute "assertValidResultSet" immediately
        
    }

</pre>
&nbsp;

Using the optional **_delay_** and **_units_** parameters, execution of the subsequent method can be delayed:

<pre>   
    
    @Schedule(method = "assertValidResultSet", delay = 30, units = Delay.MINUTES)
    public void ingestData(State state) {

        // on "ingestData" completion:
        //    - execute "assertValidResultSet" in 30 minutes
        
    }

</pre>
&nbsp;

The optional **_condition_** parameter allows for conditional scheduling of subsequent method(s), based on the exit 
status of the annotated method:

<pre>
    @Schedule(condition = Condition.SUCCESS, method = "reportTestResults", delay = 10, units = Delay.MINUTES)
    @Schedule(condition = Condition.FAILURE, method = "alertTriageTeam", delay = 30, units = Delay.SECONDS)
    public void assertValidResultSet(State state) {

        // on "assertValidResultSet" completion:
        //    - on success: schedule "reportTestResults" with "10 minutes" delay
        //    - on failure: schedule "alertTriageTeam" with "30 seconds" delay
        
    }
    
</pre>
&nbsp;

Async-o-matic supports attachment of multiple ```@Schedule``` annotations to a given method, allowing for execution to 
branch within a test and achieve parallelization where it makes sense.

<pre>
    @Schedule(method = "triggerAdditionalProcessing")
    @Schedule(method = "assertValidResultSet", delay = 30, units = Delay.SECONDS)
    public void ingestData(State state) {

        // on "ingestData" completion:
        //    - execute "triggerAdditionalProcessing" immediately; and
        //    - schedule "assertValidResultSet" with "30 seconds" delay
        
    }
    
</pre>
&nbsp;

#### *@Retry*
The ```@Retry``` annotation allows for the annotated method to re-schedule itself on failure, with an appropriate 
delay, up to a maximum number of retry attempts. The parameters available to the ```@Retry``` annotation are shown 
below:

<pre>

public @interface Retry {

    int count();                                                // required
    long delay();                                               // required
    int units();                                                // required

}

</pre>
&nbsp;

The ```@Retry``` annotation specifies the maximum **_count_** of retry-on-failure attempts of the annotated method,
along with the scheduling **_delay_** and **_units_**:

<pre>

    @Retry(count = 4, delay = 30, units = Delay.SECONDS)
    @Schedule(condition = Condition.SUCCESS, method = "reportTestResults", delay = 10, units = Delay.MINUTES)
    @Schedule(condition = Condition.FAILURE, method = "alertTriageTeam", delay = 30, units = Delay.SECONDS)
    public void assertProcessingCompleted(State state) {

        // on "assertProcessingCompleted" success:
        //    - schedule "reportTestResults" with "10 minute" delay
        //
        // on "assertProcessingCompleted" failure:
        //    - on retry attempts < 4: re-schedule "assertProcessingCompleted" with "30 seconds" delay
        //    - on retry attempts = 4 (or greater): schedule "alertTriageTeam" with "30 seconds" delay

    }

</pre>
&nbsp;

### Take a Test Drive
To take Async-o-matic for a test drive on your local machine, follow these simple instructions:
- setup and start [devcloud](https://github.com/asyncomatic/devcloud) on your local machine 
([instructions found here](https://github.com/asyncomatic/devcloud/blob/main/README.md));
- setup and start the [java-starter](https://github.com/asyncomatic/java-starter) on your local machine 
([instructions found here](https://github.com/asyncomatic/java-starter/blob/main/README.md));
- execute one of the sample tests using ```curl``` or Postman!

Happy testing!

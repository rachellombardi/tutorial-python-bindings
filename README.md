# HTCondor Python Bindings Hands-On Tutorial

HTCondor provides an interface for interacting with HTCondor services in Python, referred to as the "Python Bindings".

This document will guide you through a hands-on tutorial to explore the function and features of these Python bindings.

## Setup

Copy the contents of the tutorial directory from
`/data/datagrid/htcondor_tutorial/tutorial-python-bindings`
to your personal working directory on your HTCondor Access Point.

You can also clone this repository with

```
git clone -b '2024-09-Utrecht' https://github.com/CHTC/tutorial-python-bindings.git
```

Once download is complete, you should see a directory called `tutorial-python-bindings`. Navigate into that directory:

```
cd tutorial-python-bindings
```

If you list the contents of this directory with the command `ls`, you should see the following files: 

```
LICENSE  README.md  sleep.sh  sleep.sub
```

You are now ready to continue with the tutorial. 


## Loading the Python bindings

HTCondor comes with the Python bindings installed, available via the `htcondor` python module.

To start, login to the server you would normally submit HTCondor jobs, and launch a Python interactive terminal:

```
$ python3
```

You should see something like this for the Nikhef cluster:

```
$ python3
Python 3.9.18 (main, Jul  3 2024, 00:00:00)
[GCC 11.4.1 20231218 (Red Hat 11.4.1-3)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

You can load the python bindings with

```
import htcondor
```

> Some HTCondor Access Points do not have the HTCondor python bindings pre-installed. 
> If you need to install the python bindings, in your system shell, run
>
> ```
> python3 -m pip install htcondor
> ```

To demonstrate that the bindings are loaded, run this command:

```
htcondor.version()
```

You should see output similar to this:

```
>>> htcondor.version()
'$CondorVersion: 23.7.0 2024-04-23 BuildID: 728289 PackageID: 23.7.0-0.728289 GitSHA: a3bbc13f RC $'
```

> A newer implementation of the Python bindings is available for beta testing via the `htcondor2` module.
> From the user's perspective, very little should change.
> To test existing code with the new version, try using
>
> ```
> import htcondor2 as htcondor
> ```
>
> Commands in this tutorial should function the same regardless.
> If you want to use the beta version, I recommend that you exit and restart your
> Python session.

## Submitting Jobs

In this tutorial directory are files `sleep.sub` and `sleep.sh`.
We will be submitting test jobs using these files to demonstrate how to use the
Python bindings for job placement and monitoring.

### Loading the template submit file

Start by loading the contents of `sleep.sub` into the Python variable `template`:

```
with open('sleep.sub', 'r') as my_file:
        template = my_file.read()
```

You can see the contents of the original file with

```
print(template)
```

As the variable name suggests, we will be using this file as a "template" for the
job submission process.

> There is a `queue` statement in the `sleep.sub` file, but in v1 Python bindings
> this statement is generally ignored.
> In the v2 Python bindings, this `queue` statement is respected.
> In both cases, however, the statement can be overriden during the submission step.

### Creating a htcondor.Submit() object

The raw contents of the submit file contained within the `template` variable needs to be parsed.
We do so by using the `htcondor.Submit()` object (not to be confused with the submit method we will be discussing next).

Let's create a `template_object` that contains the parsed submit information.

```
template_object = htcondor.Submit(template)
```

This action parses the submit file and processes any special submit options that are
more complicated than key:value pairs.

You can view the contents of the `Submit` object with

```
print(template_object)
```

You can also add/modify values of the `Submit` object like you would a regular Python `dict` object (except with case-insensitive keys!).

```
print(template_object['executable'])
```

> Note that special submit options will not be processed correctly if added after
> the `Submit` object has been created.

### Instantiating the htcondor.Schedd() object

To submit a job using our template information, we need to create the interface
between our Python session and the HTCondor service that manages jobs on the access point.
This is accomplished with the following:

```
schedd = htcondor.Schedd()
```

This object contains the methods for interacting with the access point, including submission and job monitoring.
(This only needs to be instantiated once per session.)

### Submitting the Submit object via the Schedd object

Now that both the template `Submit` object and the `Schedd` object have been instantiated, we are ready to submit our test jobs.

> If using the v1 bindings, we must manually define the number/variety of jobs to submit,
> at submission time.
> If using the v2 bindings, the original `queue` statement in the submit file will be respected but we can override.

While the original file `sleep.sub` has `queue 2`, let's modify our submission. 

We could simply submit 3 jobs instead, with the following command:

```
submit_info = schedd.submit(template_object, count=3)
```

(Because a lot of information is returned by the `.submit()` method, we capture that in its own variable for future reference.)

**But** the more interesting case is to use the "itemdata" feature.

### Submitting the Submit object using Itemdata

In the submit template, we have `arguments = $(Process)`.
You can confirm this with

```
template_object['arguments']
```

Let's change this to `arguments = $(my_variable)`:

```
template_object['arguments'] = '$(my_variable)'
```

Now, we want to create a list of jobs where each job has a different value of `my_variable`.
To do so, we are going to create a list object, where each item corresponds to the changes to the template `Submit` object for one job.

Let's make the list, where you can create however many rows and use whatever values you want.
(Though since this is a tutorial, let's keep the count below 10.)

```
joblist = [
    {'my_variable': 'foo'},
    {'my_variable': 'bar'},
    {'my_variable': 'Utrecht'},
]
```

> Note that you can have however many key:value pairs you want in each dictionary.

In the above example, the first job submitted using this information will set the value of `my_variable` to `'foo'`.

To submit, we pass this `joblist` as an `iter` object to the `itemdata` argument of the `.submit` method:

```
submit_info = schedd.submit(template_object, itemdata=iter(joblist))
```

To confirm the submission worked as expected, let's check the info in `submit_info`:

```
print(submit_info.cluster())
print(submit_info.num_procs())
```

In a separate terminal, you should confirm these details are correct with a `condor_q` command.

## Monitoring Jobs

We can use the `Schedd` object to make queries for active and inactive jobs.

### Checking active jobs in the queue

Let's check on the jobs we just placed into the queue.
We do so using the `query` method of the `Schedd` object.
Run this command, and then we'll explain what's happening.

```
query = schedd.query(constraint='ClusterId == {}'.format(submit_info.cluster()), projection=['ClusterId', 'ProcId', 'Args', 'JobStatus'])
```

This method is taking two arguments: `constraint` and `projection`.
The `constraint` argument limits the results to only the jobs in the queue that match the constraint(s).
The `projection` argument limits the information that is returned; you can think of it as an output selector.

> If you do not set the `constraint`, the query will return information on **everyone's** active jobs that are in the queue.
> If you do not set the `projection`, the query will return **all the information** HTCondor has for each job it returns.

The query is returned as a list with the information you asked for, as well as the `ServerTime`:

```
print(query)
```

You should see something like this:

```
[[ Args = "foo"; ProcId = 0; ClusterId = 2021620; JobStatus = 2; ServerTime = 1726520029 ], [ Args = "bar"; ProcId = 1; ClusterId = 2021620; JobStatus = 2; ServerTime = 1726520029 ], [ Args = "Utrecht"; ProcId = 2; ClusterId = 2021620; JobStatus = 2; ServerTime = 1726520029 ]]
```

We can monitor the status of the jobs using the `query` method, but **be careful!**.
Many repeated `query`s can overwhelm the Schedd, causing it to freeze up, which means
no one else can submit jobs or make queries.
If you want have live updates, only query the Schedd every now and then (say once per minute).

> You may want to define a query function that requests and outputs the information
> you want, in a form that you want.

### Checking inactive jobs in the history

Once a job has completed (from HTCondor's perspective) its information will leave the queue and be placed in the history.
You can ask the `Schedd` object to return information from history similarly to the `query` method.

Once your jobs have finished (as indicated by your `query`, or by `condor_q` in another terminal), run the following:

```
history = schedd.history(constraint='ClusterId == {}'.format(submit_info.cluster()), projection=['ClusterId', 'ProcId', 'JobStatus'])
```

This command returns an `iterator` object and will be consumed upon use.
To keep a copy of its contents around, create a list from the iterator:

```
records = [i for i in history]
```

Then to view the outputs, can do

```
for info in records:
        print(info)
```

You'll see something like this:

```
    [
        ProcId = 2;
        ClusterId = 547678;
        JobStatus = 4
    ]

    [
        ProcId = 1;
        ClusterId = 547678;
        JobStatus = 4
    ]

    [
        ProcId = 0;
        ClusterId = 547678;
        JobStatus = 4
    ]
```

Like with the `query` method, **do not run high-frequency checks of the history!**
Otherwise you can overwhelm the Schedd.

> While the above example commands using the `query` and `history` are not necessarily useful on their own,
> they do provide the methodology for you to report on any information that you may find useful.

## Parsing the Job Log File

HTCondor tracks the events in the job management process in the user-provided `log` file.
In the file `sleep.sub` is the definition `log = sleep.$(Cluster).log`.

You can confirm this with

```
template_object['log']
```

You can use the Python bindings to load and parse the information contained within this file.

### Loading the job log file

Here we'll use the `htcondor.JobEventLog` class to load the events from the log file.

```
jel = htcondor.JobEventLog('sleep.{}.log'.format(submit_info.cluster()))
```

This returns an `iterator` object containing all of the events that were recorded by HTCondor in the log file.

### Viewing events

You can iterate over the events how you see fit, but for this example we will create an `events` list that contains all of the events.

```
events = [i for i in jel.events(stop_after=0)]
```

> The `stop_after` argument is used to tell the command how long to watch the
> contents of the specificed log file; this is useful for tracking new events
> that get added to the log file. (This is a feature that `condor_watch_q` uses!)

Let's look at the first event.

If we just ask for the item of the list, we see a `JobEvent` object representation:

```
events[0]
```

Looks something like this:

```
>>> events[0]
JobEvent(type=0, cluster=547678, proc=0, timestamp=1726515390)
```

On the other hand, if we ask to print the item, we see the actual message that is written in the log file:

```
print(events[0])
```

Looks something like this:

```
>>> print(events[0])
000 (547678.000.000) 2024-09-16 21:36:30 Job submitted from host: <145.107.7.246:9618?addrs=145.107.7.246-9618+[2a07-8500-120-e070--3f6]-9618&alias=taai-007.nikhef.nl&noUDP&sock=schedd_4212_a8fe>

```

### Extracting specific information from an event

The `JobEvent` object is another `htcondor` object and behaves like a dictionary.
For example, you can see the time when event 0 was recorded with

```
events[0]['EventTime']
```

The exact keys that you can extract information from depends on the exact type of the event.
In general, each event will have 'Proc', 'MyType', 'Cluster', 'Subproc', 'EventTime', and 'EventTypeNumber'. Other information specific to that message will be mapped to an appropriate key.
(For example, event 0 is of type 'SubmitEvent', and so has a key 'SubmitHost' that other event types will not have.)

### Filtering events

If you know that you only want information from events for a particular ProcID,
or that you only want information on events that are of type "Hold",
you can filter an existing `events` list for those particular features.

Let's filter out the events list for just the events with ProcID 0:

```
proc0 = [i for i in events if i['Proc'] == 0]
```

> For a long log file, you should probably check the conditionals when you
> first create the `events` list from the `JobEventLog` iterator.
> We can accomplish this with:
> 
> ```
> proc0 = [i for i in jel.events(stop_after=0) if i['Proc'] == 0]
> ```
>
> Keep in mind that the `jel` iterator was consumed when you created the `events` list earlier,
> so if you want to try this now you will have to re-create the `jel` iterator.

You can now see all of the ProcID 0 events with

```
for i in proc0:
        print(f'{i}...')
```

(Appending the '...' string to the end replicates the syntax you see in the actual log file.)


## Appendix: DAGMan jobs

To learn how to submit DAGMan jobs using HTCondor's python bindings, <a href="https://htcondor.readthedocs.io/en/latest/apis/python-bindings/tutorials/DAG-Creation-And-Submission.html">follow this tutorial</a>.

---
layout: post
title: Comparing AWS's Simple-Workflow Service & Step Functions
---

## Introduction

A cursory Google search will yield articles comparing SWF to Step-Functions, yet none that we've found exhibit any depth. The following article contains some lessons that have been learnt using these two products.  

Both services can be considered state-machines-as-a-service. Both provide resiliency and are great "glue" for building distributed systems that (a) don't rely on low-latency execution and (b) don't rely on transaction volume. Building any such resilient state-machines is time-consuming and algorithm heavy.

Both services are remarkably similar. SWF was released in 2012, with Step-Functions in 2016. Step-Functions were a clear attempt to modernize SWF. However, it is possible to surmise that the Step-Function implementation is immature compared to SWF. These aspects will be discussed.

Also note, this article is not a discussion of serverless architectures. 

## Background

### Simple Workflow Service (SWF)
SWF separates logic into (a) a decider (i.e. workflow logic) and (b) activities. The decider is essentially the code that routes calls to activities. Each time a decider task is executed it is passed the entire workflow execution/event history. It's up to the recipient to parse the events, determine the current state, and issue new tasks.

SWF's basic implementation is a collection of HTTP APIs. On top of you that you can use either (a) the AWS Java SDK (a wrapper over the HTTP APIs) or (b) the "Flow Framework", which is open-source. The Flow Framework provides a reactive-like API. You specify the workflow as calls to activities, which depend on each other based on  `Promise`  instances. What's exceedingly confusing is that it  _looks_  like asynchronous code, but it's not. As mentioned above, each time it is called it receives the entire workflow event history. The Flow Framework then passes through the workflow of  `Promise`  implementations and "replays" execution.

### Step-Functions
Step-Functions strives to provide a purer state-machine implementation. There is no discrimination between a decider and an activity. The flow supports pass-through states, wait-states, choice-states, parallel-states (branching) and task-states. Of these, task-states are implemented in code (in Java, Go, JavaScript, etc.). The AWS Java SDK API for Step-Functions is simple and directly maps to the advertised HTTP API.

# Comparison
The rest of this article will attempt to compare various aspects of using SWF and Step-Functions.

## Workflow Definition
In SWF, when using the Flow Framework API, there is no static/concrete workflow definition: you build the workflow in code based on  `Promise`  instances, which are returned by requesting an activity is executed. Between that, and the overly-complex use of the  `Async`  annotation (or the simpler  `Task`  implementation) it can be a challenge to understand the workflow. IMHO, it would be interesting to attempt a replacement of the Flow Framework API, coded to the AWS Java SDK, that allows the state-machine to be defined (as a DSL?) and be decorated with interface implementations that contain decider logic. This is a more natural fit for replay logic.

In Step-Functions, you have to explicitly define the workflow, either via (a) the UI/editor or (b) as json which can be uploaded via HTTP API. For (b), the  ["AWS Step Functions Fluent Java API"](https://aws.amazon.com/blogs/developer/stepfunctions-fluent-api/)  is provided to provide a DSL/fluent approach to workflow definition.

## Workflow Flexibility

### Parallelism
Both SWF and Step-Functions suffer from the same problem. They only offer a join operator.  There is no aggregated-join. To add parallelism you have to explicitly define  `B0`,  `B1`,  `B2`,  `B_n_`, etc. It is exceedingly limiting and severely affects runtime parallelism (though mitigated if you have lots of concurrent workflows executing). In SWF you have to manage lists of  `Promise`  instances where you define branching – but at least that can be done at run-time. In Step-Functions, it has to be done when you define the workflow, which is static, i.e. it  _cannot_  be done at runtime. There is some discussion of this  [here](https://forums.aws.amazon.com/thread.jspa?messageID=754967).

### Loops
In SWF, you can code whatever you want into the decider task, as long it's deterministic, sane, etc. Thus, writing loops for activities is straightforward. However, in Step-Functions you are forced to handle loop increment logic within activities.

```java
private static StateMachine uploadBatchStateMachine() {
    return StateMachine.builder()
        .startAt("Check Batch Completion")
        .state("Check Batch Completion", choiceState()
            .choice(choice()
                .transition(next("Upload Batch Complete"))
                .condition(eq("$.remaining", 0)))
            .defaultStateName("Upload Part"))
        .state("Upload Part", taskState()        // the loop counter is updated in this activity
            .resource("arn:aws:states:us-west-2:XXXXXXXXXXXX:activity:UploadS3Part")
            .transition(next("Check Batch Completion")))
        .state("Upload Batch Complete", taskState()
            .resource("arn:aws:states:us-west-2:XXXXXXXXXXXX:activity"
                + ":CompletedUploadBatchChildWorkflow")
            .transition(end()))
        .build();
}
```

### Workflow Instance Identification
In SWF, each workflow instance has (a) a  `jobId`  which is assigned by SWF, (b) an optional  `workflowId`  which is provider by the caller and (c) an optional list of tags (a list of strings). All of these can be filtered. The  `workflowId`  has to be unique across to  `OPEN`  instances of that specific workflow. This is  very  useful. It means that if you have a workflow that is executing against a given object/resource, it means you can guarantee that you only have one active workflow against that resource.

Step-Functions are more restrictive: they can only have a  `name`  per flow execution instance (discussed  [here](https://docs.aws.amazon.com/step-functions/latest/apireference/API_StartExecution.html)). It has to be unique across  `OPEN`  and  `CLOSED`  instances. It is also idempotent, such that if you initiate a workflow with (a) the same  `name`  and (b) the same input json, you could receive the handle for an existing active workflow. Having it across both  `OPEN`  and  `CLOSED`  flow instances is limiting.

A workaround could be to only use the Step-Function history for  `OPEN`  instances, i.e. as soon as it is finished, store the event-history in S3, and then delete it from the Step-Function service. This could be tricky as the last step of a state-machine would then be to instantiate a new workflow to perform the cleanup.

### Child Workflows
Child workflows are exceptionally useful, for example, to help work around API limits/throttling. If you exceed the event-history limit, one workaround is to split functionality across child-workflows.

SWF has native support for child-workflows. It works as you'd expect. Errors are propagated, a cancellation of the parent workflow kills any running child workflows, etc, etc. However, note, if you terminate a workflow  you are not notified. This is discussed in another section.

Step-Functions has no explicit support for child-workflows. However, it is possible to construct in Step-Functions in a clean fashion. From a parent workflow, i.e. within an activity, you'd create an instance of a new flow, and pass the  `taskToken`  of the originating activity into the new workflow instance. When the child workflow has completed, you call back to  `SUCCESS` the task associated with that  `taskToken`. We have proved that this works. However, we haven't yet proved the associated error handling. Presumably, the child-workflow needs to detect/handle the error in the state-machine, and set  `FAILED`  on the originating task. However, because Step-Functions have a simple API it would be straightforward to abstract/hide that logic.

The following code shows how such a child workflow can be built with Step-Functions. This is just the first solution we found, there might be others. Please see the notes below for more detail.

``` java
private static StateMachine uploadBatchStateMachine() {
    return StateMachine.builder()
        .startAt("Child Flow")
        .state("Child Flow", parallelState()
            .transition(next("Child Flow Successful"))
            .branch(branch()
                .startAt("Check Batch Completion")
                // For a loop, detect if all parts of the batch have been uploaded or not.
                .state("Check Batch Completion", choiceState()
                    .choice(choice()
                        .transition(next("Upload Batch Complete"))
                        .condition(eq("$.remaining", 0)))
                    .defaultStateName("Upload Part"))
                // Upload the current part.
                .state("Upload Part", taskState()
                    .resource("arn:aws:states:us-west-2:XXXXXXXXXXXX:activity:UploadS3Part")
                    .transition(next("Check Batch Completion")))
                .state("Upload Batch Complete", passState().transition(end())))
            .catcher(catcher()
                .catchAll()
                .resultPath("$.error")
                .transition(next("Child Flow Failed")))
        )
        // Child state-machine has failed, notify parent state-machine.
        .state("Child Flow Failed", taskState()
            .resource("arn:aws:states:us-west-2:XXXXXXXXXXXX:activity:ChildWorkflowFailed")
            .transition(end()))
        // Child state-machine is complete, notify parent state-machine.
        .state("Child Flow Successful", taskState()
            .resource("arn:aws:states:us-west-2:XXXXXXXXXXXX:activity:ChildWorkflowSuccessful")
            .transition(end()))
        .build();
}
```
NOTES:
-   The input object contains the value for  `completionTaskToken`. This is the taskToken that  `ChildWorkflowSuccessful`  and  `ChildWorkflowFailed`  report back to the parent state-machine with.
-   Due to how error-handling works, to trap an error from a sequence of states, you can wrap the sequence in a parallel-state of only 1x branch.
-   When launching a child state-machine it's important to record the  `name`  of the child-workflow. Without that, you don't have a full audit trail of parent/child state-machines.

### External/Manual Activity Completion

This is where you have an activity, but you want it to respond to some external stimulus, e.g. someone calls in to a REST endpoint, or a human's approval is required, etc. SWF has native support for this. To do that, you have to get the activity's  `taskToken`. This token is retrieved from the context provider within the activity, e.g.:

```java
new ActivityExecutionContextProviderImpl().getActivityExecutionContext().getTaskToken()
```

### Halting a Workflow Instance 
SWF supports two workflow concepts: (i) cancellation and (ii) termination.  Upon cancellation, an event is placed into the workflow event history.  In the case of SWF with Flow Framework, this is presented as a Java exception.  Upon termination, the workflow is simply halted, and no event is created.

Step-Functions only support termination (called "Abort").  The state-machine cannot detect the termination has occurred.

So how do you deal with cases were you aren't notified (SWF termination and SF abort)?  It is certainly possible to build a daemon to track workflow execution status. However, it is likely you'd run into API limits retrieving the status and/or event-history. Once that you have detected that a state-machine has  `ABORTED`, etc., you can trigger a mitigation workflow. This would have to include logic to identify any active parent/child state-machines, etc, etc.

In fact, the daemon could handle  `SUCCESSFUL`,  `FAILED`  and  `ABORTED`  cases – assuming you can take the hit on latency, which would be need to not trigger API throttling/limits.

### Threading Model

In SWF, you create separate queues for decider and activity tasks. When creating activity queues you can create multiple queues and assign various activity-types to each.

In Step-Functions, you are forced to create a thread-pool for each individual activity-type. Remember, Step-Functions were probably geared to the Lambda world where this is irrelevant. Still, for the non-Lambda case it should probably be fixed.

### Activity Queue Routing (i.e. Affinity)

In SWF, when requesting an activity be executed you can specify the name of the queue you route it to. Obviously, the queue with that name has to exist. This is very useful: imagine a case where you have 2x clusters of your SWF service, one in AWS, and another running in a colo site. Using this affinity capability you can route activities accordingly.

Step-Functions does not appear to have affinity support. You could do it by having multiple copies of an activity (each with different ARNs), and choice logic in the state-machine, but that would be . . . messy.

### Deployment

SWF supports versioning of a workflow and each activity type. Though, run-time behaviour can be little strange. If you increment the version number of an activity, you have to delete and re-create the domain the workflow lives in, else the workflow won't execute properly. This has to be managed somehow.

Step-Functions do not support versioning. If you modify a state-machine, the Java SDK API won't let you upload it. You're forced to delete the workflow (and all associated invocations!) and re-create it. When you do re-deploy a state-machine, the ARN changes. Any service/code handling Step-Function activities should probably not hard-code ARNs. Instead, query by name to get the ARN.

### Logging/Event-Tracing

For both SWF and Step-Functions, you need to call the HTTP APIs to retrieve the event-history for a given workflow instance. Each request can read up to 10,000 events. Past that, you need to use the pagination feature. If you have child workflows, you have to call the API again to get it's history. This causes an acute problem. If you're trying to get status-events with short intervals, you'll run into API throttling issues very quickly. In SWF, if using the Flow-Framework API, the only way around this would be to add code to each decider and activity implementation to separately send out state-change information,  _which is tedious_. Step-Functions are easier, in that you can easily hide the code to send out state-change information within your Step-Function activity handler.

In SWF, there is another challenge. When parsing the event-history, activity inputs are given as json, but, they are typed – in that they include class names. I don't know if the parser (a Jackson configuration?) is exposed via the Flow-Framework API. We've ended up having to parse it manually.

### API Throttling

It is important to be aware of this. In SWF, the default limit for the size of an event-history is 25,000 entries. Bear in mind, if you're in loop/iteration use-case, you bounce between the decider and an activity, and that consumes 7 events. It's similar in Step-Functions.  You could split the workflow into multiple (e.g. child) workflow executions. Or, you could ask for your limit to be increased.

### Handling State Inputs/Outputs

This is trivial in SWF. Your decider logic essentially has full access to all inputs/outputs.

It's more of a challenge in Step-Functions. It is discussed  [here](https://docs.aws.amazon.com/step-functions/latest/dg/amazon-states-language-input-output-processing.html). It's tricky. You can get into cases, e.g. with parallel branching, where, for example, you can specify the input as an an entry on an array, e.g. "`$.[0]`". However, as soon as you do that, the rest of discarded input is then lost for that tree/branch of execution. When a parallel branch is complete, the results are returned as an array.




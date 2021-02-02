## execution


The Flowable engine expects processes to be defined in the BPMN 2.0 format, which is an XML standard that is widely accepted in the industry. In Flowable terminology, we speak about this as a **process definition**. From a *process definition*, many **process instances** can be started. Think of the *process definition* as the blueprint for many **executions** of the process. <sup>[🔗](https://flowable.com/open-source/docs/bpmn/ch02-GettingStarted/#deploying-a-process-definition)</sup>

Every step (in BPMN 2.0 terminology, '**activity**') has an *id* attribute that gives it a unique identifier in the XML file. All *activities* can have an optional name too, which increases the readability of the visual diagram, of course.

The *activities* are connected by a **sequence flow**, which is a directed arrow in the visual diagram. When executing a process instance, the **execution** will flow from the *start event* to the next *activity*, following the *sequence flow*.

When the process instance is started, an **execution** is created and put in the start event. From there, this *execution* follows the sequence flow to the user task for the manager approval and executes the user task behavior. This behavior will create a task in the database that can be found using queries later on. A **user task is a *wait state*** and the engine will stop executing anything further, returning the API call.

The Runtimeservice also allows you to query on process instances and *executions*. ****Executions** are a representation of the 'token' concept of BPMN 2.0. Basically**, an *execution* is a pointer to where the process instance currently is. Lastly, the RuntimeService is used whenever a process instance is waiting for an external trigger and the process needs to be continued. **A process instance can have various wait states** and this service contains various operations to 'signal' to the instance that the external trigger is received and the process instance can be continued.<sup>[🔗](https://flowable.com/open-source/docs/bpmn/ch04-API/#the-process-engine-api-and-services)</sup>

So a process instance might have one ore more *executions* depending on the flow.<sup>[🔗](https://forum.flowable.org/t/what-exactly-executionid-signifies/6122/2?u=feng)</sup>

**The executions are used to store the state of a process instance**. <sup>[🔗](https://forum.flowable.org/t/how-to-use-execution/377/6?u=feng)</sup>

Flowable will only persist when reaching a *wait state*. If you have ```start -> stepA -> … -> stepX```. And stepX is a *wait state* (e.g. a user task), the state will only be persisted when stepX is reached.<sup>[🔗](https://forum.flowable.org/t/flowable-process-execution/4610/3?u=feng)</sup>
  
Gateways are not considered *execution* points. Setting a local variable on the *execution* in the service task will be available in the next sequence flows that will reuse the same *execution*. This is however not good design, because if you would add a parallel gateway after the service task, this won’t be valid anymore for example. Because that will create new *executions* for the outgoing paths of the parallel gateway.<sup>[🔗](https://forum.flowable.org/t/are-gateways-considered-execution-points/1827/5?u=feng)</sup>
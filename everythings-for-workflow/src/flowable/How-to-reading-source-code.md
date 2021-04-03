## How to reading source code

1. Write a simple process or case that you start and then debug what happens behind the scenes.
2. Inspect important parts to understand how it fundamentally works are:
    * the agenda en the operations: [https://github.com/flowable/flowable-engine/tree/master/modules/flowable-engine/src/main/java/org/flowable/engine/impl/agenda](https://github.com/flowable/flowable-engine/tree/master/modules/flowable-engine/src/main/java/org/flowable/engine/impl/agenda)
    * the entitymanager (kinda like a service) and data managers who are responsible for data persistence: [https://github.com/flowable/flowable-engine/tree/master/modules/flowable-engine/src/main/java/org/flowable/engine/impl/persistence/entity](https://github.com/flowable/flowable-engine/tree/master/modules/flowable-engine/src/main/java/org/flowable/engine/impl/persistence/entity)
    * the ActivityBehaviors which actually implement the actual logic that happens when executing a certain step: [https://github.com/flowable/flowable-engine/tree/master/modules/flowable-engine/src/main/java/org/flowable/engine/impl/bpmn/behavior](https://github.com/flowable/flowable-engine/tree/master/modules/flowable-engine/src/main/java/org/flowable/engine/impl/bpmn/behavior)


The engine will inspect the process definition and when it reaches a certain step (let’s say a user task), it will execute the associated behavior (e.g. the UserTaskActivityBehavior).
# Summary

[Preface]()

# Part 1 - Understanding the Domain

- [Introducing Domain-Driven Design]()

- [Understanding the Domain]()

- [A Functional Architecture]()


# Part 2 - Modeling the Domain

- [Understanding Types]()

- [Domain Modeling with Types]()

- [Integrity and Consistency in the Domain]()

- [Modeling Workflows as Pipelines]()


# Part 3 - Implementing the Model

- [Understanding Functions](./understanding-functions/README.md) 
    - [Functions, Functions, Everywhere](./understanding-functions/Functions-Functions-Everywhere.md)  
    - [Functions Are Things](./understanding-functions/Functions-Are-Things.md)
    - [Total Functions](./understanding-functions/Total-Functions.md)
    - [Composition](./understanding-functions/Composition.md)
    - [Wrapping Up](./understanding-functions/Wrapping-Up.md)


- [Implementation: Composing a Pipeline](./implementation-composing-a-pipeline/README.md) 
    - [Working With Simple Types](./implementation-composing-a-pipeline/Working-With-Simple-Types.md)  
    - [Using Function Types to Guide the Implementation](./implementation-composing-a-pipeline/Using-Function-Types-to-Guide-the-Implementation.md)  
    - [Implementing the Validation Step](./implementation-composing-a-pipeline/Implementing-the-Validation-Step.md)  
    - [Implementing the Rest of the Steps](./implementation-composing-a-pipeline/Implementing-the-Rest-of-the-Steps.md)  
    - [Composing the Pipeline Steps Together](./implementation-composing-a-pipeline/Composing-the-Pipeline-Steps-Together.md)  
    - [Injecting Dependencies](./implementation-composing-a-pipeline/Injecting-Dependencies.md)  
    - [Testing Dependencies](./implementation-composing-a-pipeline/Testing-Dependencies.md)  
    - [The Assembled Pipeline](./implementation-composing-a-pipeline/The-Assembled-Pipeline.md)  
    - [Wrapping Up](./implementation-composing-a-pipeline/Wrapping-Up.md)  


- [Implementation: Working with Errors](./implementation-working-with-errors/README.md) 
    - [Using the Result Type to Make Errors Explicit](./implementation-working-with-errors/Using-the-Result-Type-to-Make-Errors-Explicit.md)  
    - [Working with Domain Errors](./implementation-working-with-errors/Working-with-Domain-Errors.md)  
    - [Chaining Result-Generating Functions](./implementation-working-with-errors/Chaining-Result-Generating-Functions.md)  
    - [Applying “bind” and “map” to Our Pipeline](./implementation-working-with-errors/Applying-bind-and-map-to-Our-Pipeline.md)  
    - [Adapting Other Kinds of Functions to the Two-Track Model](./implementation-working-with-errors/Adapting-Other-Kinds-of-Functions-to-the-Two-Track-Model.md)  
    - [Making Life Easier with Computation Expressions](./implementation-working-with-errors/Making-Life-Easier-with-Computation-Expressions.md)  
    - [Monads and more](./implementation-working-with-errors/Monads-and-more.md)  
    - [Adding the Async Effect](./implementation-working-with-errors/Adding-the-Async-Effect.md)  
    - [Wrapping Up](./implementation-working-with-errors/Wrapping-Up.md)  


- [Serialization](./serialization/README.md) 
    - [Persistence vs. Serialization](./serialization/Persistence-vs-Serialization.md)  
    - [Designing for Serialization](./serialization/Designing-for-Serialization.md)  
    - [Connecting the Serialization Code to the Workflow](./serialization/Connecting-the-Serialization-Code-to-the-Workflow.md)  
    - [A Complete Serialization Example](./serialization/A-Complete-Serialization-Example.md)  
    - [How to Translate Domain types to DTOs](./serialization/How-to-Translate-Domain-types-to-DTOs.md)  
    - [Wrapping Up](./serialization/Wrapping-Up.md)  
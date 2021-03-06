# Domain Modeling Made Functional




[Cover](./Cover.md)  
[Preface](./preface/README.md)  
[Who Is This Book For?](./preface/Who-Is-This-Book-For.md)  
[What’s In This Book?](./preface/Whats-In-This-Book.md)  
[Other Approaches To Domain Modeling](./preface/Other-Approaches-To-Domain-Modeling.md)  
[Working With the Code In This Book](./preface/Working-With-the-Code-In-This-Book.md)  
[Acknowledgements](./preface/Acknowledgements.md)  

<!-- # Part 1 - Understanding the Domain

- [Introducing Domain-Driven Design]()

- [Understanding the Domain]()

- [A Functional Architecture]()


# Part 2 - Modeling the Domain

- [Understanding Types]()

- [Domain Modeling with Types]()

- [Integrity and Consistency in the Domain]()

- [Modeling Workflows as Pipelines]() -->


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



- [Persistence](./persistence/README.md)
    - [Pushing Persistence to the Edges](./persistence/Pushing-Persistence-to-the-Edges.md)
    - [Command-Query Separation](./persistence/Command-Query-Separation.md)
    - [Bounded Contexts Must Own Their Data Storage](./persistence/Bounded-Contexts-Must-Own-Their-Data-Storage.md)
    - [Working with Document Databases](./persistence/Working-with-Document-Databases.md)
    - [Working with Relational Databases](./persistence/Working-with-Relational-Databases.md)
    - [Transactions](./persistence/Transactions.md)
    - [Wrapping Up](./persistence/Wrapping-Up.md)


- [Evolving a Design and Keeping It Clean](./evolving-a-design-and-keeping-it-clean/README.md)
    - [Change 1: Adding Shipping Charges](./evolving-a-design-and-keeping-it-clean/Change-1-Adding-Shipping-Charges.md)
    - [Change 2: Adding Support for VIP Customers](./evolving-a-design-and-keeping-it-clean/Change-2-Adding-Support-for-VIP-Customers.md)
    - [Change 3: Adding Support for Promotion Codes](./evolving-a-design-and-keeping-it-clean/Change-3-Adding-Support-for-Promotion-Codes.md)
    - [Change 4: Adding a Business Hours Constraint](./evolving-a-design-and-keeping-it-clean/Change-4-Adding-a-Business-Hours-Constraint.md)
    - [Dealing with Additional Requirements Changes](./evolving-a-design-and-keeping-it-clean/Dealing-with-Additional-Requirements-Changes.md)
    - [Wrapping Up](./evolving-a-design-and-keeping-it-clean/Wrapping-Up.md)
    - [Wrapping Up the Book](./evolving-a-design-and-keeping-it-clean/Wrapping-Up-the-Book.md)



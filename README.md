# Apex Test Kit

![](https://img.shields.io/badge/version-3.0-brightgreen.svg) ![](https://img.shields.io/badge/build-passing-brightgreen.svg) ![](https://img.shields.io/badge/coverage-%3E95%25-brightgreen.svg)


Apex Test Kit is a Salesforce library to help generate massive records for either Apex test classes, or sandboxes. It solves two pain points during records creation:

1. Establish arbitrary levels of many-to-one, one-to-many relationships.
2. Generate field values based on simple rules.

Imagine the complexity to generate the following sObjects to establish all the relationships in the diagram.

<img src="docs/images/sales-object-graph.png" width="400" alt="Sales Object Graph">

However, with ATK we can create them within one Apex statement. Here, we are generating:

1. 200 accounts.
2. Each of the account have 2 contacts.
3. Each of the contact has 1 opportunity linked with opptunity contact role.
4. Also each of the account have 2 orders.
5. Also each of the order belongs to 1 opportunity from the same account.

```java
ATK.SaveResult result = ATK.prepare(Account.SObjectType, 200)
    .field(Account.Name).index('Name-{0000}')
    .withChildren(Contact.SObjectType, Contact.AccountId, 400)
        .field(Contact.LastName).index('Name-{0000}')
        .field(Contact.Email).index('test.user+{0000}@email.com')
        .field(Contact.MobilePhone).index('+86 186 7777 {0000}')
        .withChildren(OpportunityContactRole.SObjectType, OpportunityContactRole.ContactId, 400)
            .field(OpportunityContactRole.Role).repeat('Business User', 'Decision Maker')
            .withParents(Opportunity.SObjectType, OpportunityContactRole.OpportunityId, 400)
                .field(Opportunity.Name).index('Name-{0000}')
                .field(Opportunity.ForecastCategoryName).repeat('Pipeline')
                .field(Opportunity.Probability).repeat(0.9, 0.8, 0.7)
                .field(Opportunity.StageName).repeat('Prospecting')
                .field(Opportunity.CloseDate).addDays(Date.newInstance(2020, 1, 1), 1)
                .field(Opportunity.TotalOpportunityQuantity).add(1000, 10)
                .withParents(Account.SObjectType, Opportunity.AccountId)
    .also(4)
    .withChildren(Order.SObjectType, Order.AccountId, 400)
        .field(Order.Name).index('Name-{0000}')
        .field(Order.EffectiveDate).addDays(Date.newInstance(2020, 1, 1), 1)
        .field(Order.Status).repeat('Draft')
        .withParents(Contact.SObjectType, Order.BillToContactId)
        .also()
        .withParents(Opportunity.SObjectType, Order.OpportunityId)
    .save(true);
```

### Performance

To genereate the above 2200 records and saving them into Salesforce, it will take less than 3000 CPU time. That's already 1/3 of the Maximum CPU time. However, if we use `.save(false)` without saving them, and just create them in the memory, it will take less than 700 CPU time, 4x faster.

### Demos

There are four demos under the `scripts/apex` folder, they can be successfully run in a clean Salesforce CRM organization. If not, please try to fix them with FLS, validation rules or duplicate rules etc.

| Subject  | File Path                         | Description                                                  |
| -------- | --------------------------------- | ------------------------------------------------------------ |
| Campaign | `scripts/apex/demo-campaign.apex` | How to genereate campaigns with hierarchy relationships, and implement `ATK.FieldBuilder` to reuse the logic on how to populate the fields. |
| Cases    | `scripts/apex/demo-cases.apex`    | How to generate Accounts, Contacts and Cases.                |
| Sales    | `scripts/apex/demo-sales.apex`    | You've already seen it in the above paragraph.               |
| Users    | `scripts/apex/demo-users.apex`    | How to generate community users in one goal.                 |

## Setup Relationship

### One to Many

```java
ATK.prepare(Account.SObjectType, 10)
  	.field(Account.Name).index('Name-{0000}')
    .withChildren(Contact.SObjectType, Contact.AccountId, 20)
        .field(Contact.LastName).index('Name-{0000}')
        .field(Contact.Email).index('test.user+{0000}@email.com')
        .field(Contact.MobilePhone).index('+86 186 7777 {0000}')
    .save(true);
```

Here is how the relationship going to be mapped:

| Account Name | Contact Name |
| ------------ | ------------ |
| Name-0001    | Name-0001    |
| Name-0001    | Name-0002    |
| Name-0002    | Name-0003    |
| Name-0002    | Name-0004    |
| ...          | ...          |

### Many to One

```java
ATK.prepare(Contact.SObjectType, 20)
   	.field(Contact.LastName).index('Name-{0000}')
    .field(Contact.Email).index('test.user+{0000}@email.com')
    .field(Contact.MobilePhone).index('+86 186 7777 {0000}')
    .withParents(Account.SObjectType, Contact.AccountId, 10)
  			.field(Account.Name).index('Name-{0000}')
    .save(true);
```

### Many to Many

```java
ATK.prepare(Contact.SObjectType, 40)
        .field(Contact.LastName).index('Name-{0000}')
        .field(Contact.Email).index('test.user+{0000}@email.com')
        .field(Contact.MobilePhone).index('+86 186 7777 {0000}')
        .withChildren(OpportunityContactRole.SObjectType, OpportunityContactRole.ContactId, 40)
            .field(OpportunityContactRole.Role).repeat('Business User', 'Decision Maker')
            .withParents(Opportunity.SObjectType, OpportunityContactRole.OpportunityId, 40)
                .field(Opportunity.Name).index('Name-{0000}')
                .field(Opportunity.CloseDate).addDays(Date.newInstance(2020, 1, 1), 1)
                .field(Opportunity.StageName).repeat('Prospecting')
    .save(true);
```

## Keyword And APIs

There are only two keyword categories Entity keyword and Field keyword. They are used to solve the two pain points we addressed at the beginning:

1. **Entity Keyword**: Establish arbitrary levels of many-to-one, one-to-many relationships.
2. **Field Keyword**: Generate field values based on simple rules.

### Entity Keywords

Each of them will start a new sObject context. And it is advised to use the following indentation for clarity.

```java
ATK.prepare(A__c.SObjectType, 10)
    .withChildren(B__c.SObjectType, B__c.A_ID__c, 10)
        .withParents(C__c.SObjectType, B__c.C_ID__c, 10)
            .withChildren(D__c.SObjectType, D__c.C_ID__c, 10)
            .also() // go back 1 depth to C__c
  					.withChildren(E__c.SObjectType, E__c.C_ID__c, 10)
        .also(2)    // go back 2 depth to B__c
        .withChildren(F__c.SObjectType, F__c.B_ID__c, 10)
    .save(true);
```

The following APIs with a size param at the last, indicate the associated sObjects will be created on the fly.
| Keyword API | Description                                                  |
| ----------- | ------------------------------------------------------------ |
| prepare(SObjectType objectType, Integer size) | Always start chain with `prepare()` keyword. It is the root sObject to start relationship with. |
| withChildren(SObjectType objectType, SObjectField referenceField, Integer size) | Establish one to many relationship between the previous working on sObject and the current sObject. |
| withParents(SObjectType objectType, SObjectField referenceField, Integer size) | Establish many to one relationship between the previous working on sObject and the current sObject. |

The following APIs without a third param at the last, indicate the associated sObjects won't be created, rather it will look up the previously created sObjects with the same type. **Note**: Once these APIs are used, please make sure there are sObjects with the same type created previously, and only created once.
| Keyword API | Description                                                  |
| ----------- | ------------------------------------------------------------ |
| withChildren(SObjectType objectType, SObjectField referenceField) | Establish one to many relationship between the previous working on sObject and the current sObject. |
| withParents(SObjectType objectType, SObjectField referenceField) | Establish many to one relationship between the previous working on sObject and the current sObject. |

### Field Keywords

Here is an dummy example to demo the use of entity decoration keywords. 

```java
ATK.prepare(A__c.SObjectType, 10)
    .withChildren(B__c.SObjectType, B__c.A_ID__c, 10)
        .field(B__C.Name__c).index('Name-{0000}')
        .field(B__C.PhoneNumber__c).index('+86 186 7777 {0000}')
        .field(B__C.Price__c).repeat(12.34)
        .field(B__C.CampanyName__c).repeat('Google', 'Apple', 'Microsoft')
        .field(B__C.Counter__c).add(1, 1)
        .field(B__C.StartDate__c).addDays(Date.newInstance(2020, 1, 1), 1)
    .save(true);
```

| Keyword API                                         | Description                                                  |
| --------------------------------------------------- | ------------------------------------------------------------ |
| index(String format)                                | Formated string with `{0000}`, can recogonize left padding. For example: 0001, 0002, 0003 etc. |
| repeat(Object value)                                | Repeat with a fixed value.                                   |
| repeat(Object value1, Object value2)                | Repeat with the provided values alternatively.               |
| repeat(Object value1, Object value2, Object value3) | Repeat with the provided values alternatively.               |
| repeat(List\<Object\> values)                       | Repeat with the provided values alternatively.               |
| recordType(String name)                             | Look up record type ID by record type name or developer name. |
| profile(String name)                                | Look up profile ID by profile name.                          |

#### Arithmetic Field Keywords

These keywords will increase/decrease the init values by the provided step values. They must be applied to the correct feild data types that support them.

| Keyword API                           | Description                             |
| ------------------------------------- | --------------------------------------- |
| add(Object init, Object step)         | Must be applied to a number type field. |
| substract(Object init, Object step)   | Must be applied to a number type field. |
| divide(Object init, Object step)      | Must be applied to a number type field. |
| multiply(Object init, Object step)    | Must be applied to a number type field. |
| addYears(Object init, Integer step)   | Must be applied to a Date type field.   |
| addMonths(Object init, Integer step)  | Must be applied to a Date type field.   |
| addDays(Object init, Integer step)    | Must be applied to a Date type field.   |
| addHours(Object init, Integer step)   | Must be applied to a Time type field.   |
| addMinutes(Object init, Integer step) | Must be applied to a Time type field.   |
| addSeconds(Object init, Integer step) | Must be applied to a Time type field.   |

## License

Apache 2.0

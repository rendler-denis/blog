---
title: "Handling technical id's in Gherkin with SpecFlow"
date: 2020-06-27T00:00:00+02:00
image: "images/handling-technical-ids-in-gherkin-with-specflow.jpg"
tags: [ "Gherkin", "SpecFlow", "Specification by Example", "ATDD", "BDD", "Test Automation", "Cleaner Code" ]
summary: "In this post I'll show a simple trick on how to handle technical id's in Gherkin using SpecFlow."
draft: true
---

When you use Specification by Example with the Gherkin syntax and automate your scenario's with SpecFlow, you're bound to encounter situations where you'll need a technical id. For example to stub data retrieved by the id from a repository or external service.

Gherkin scenario's in Specification by Example are used to describe the functional requirements of your software. They should be readable for the team and also for the business people that use the software. Technical id's don't have a place here. So what to do when your code requires a technical id?

Let start with an example scenario:

```Gherkin
Given the following people
    | Id | Address                           |
    | 1  | 221B Baker Street, London, UK     |
    | 2  | 1630 Revello Drive, Sunnydale, US |
    | 3  | 31 Spooner Street, Quahog, US     |
    | 4  | 12 Grimmauld Place, London, UK    |
When person 3 moves to '742 Evergreen Terrace, Springfield, US'
Then the new address of person 3 is '742 Evergreen Terrace, Springfield, US'
```

This scenario describes functionality for moving a person from one address to another.

The `MovingService` class that implements the moving functionality has a simple `MovePerson` method that retrieves a person by its id from a repository and sets the new address.

```csharp
public class MovingService
{
    private readonly IPeopleRepository _peopleRepository;

    public MovingService(IPeopleRepository peopleRepository)
    {
        _peopleRepository = peopleRepository ?? throw new ArgumentNullException(nameof(peopleRepository));
    }

    public void MovePerson(int personId, string newAddress)
    {
        var person = _peopleRepository.GetById(personId);
        person.Address = newAddress;
    }
}
```

The SpecFlow glue code that automates the scenario:
- injects a simple in-memory stub into `MovingService`
- adds the people specified in the `Given` step
- calls the `MovingService.MovePerson` method 
- and verifies that the person has the new address.

```csharp
[Binding]
class InitialScenarioSteps
{
    private readonly PeopleRepositoryStub _peopleRepositoryStub = new PeopleRepositoryStub();
    private readonly MovingService _movingService;

    public InitialScenarioSteps()
    {
        _movingService = new MovingService(_peopleRepositoryStub);
    }

    [Given(@"the following people")]
    public void GivenTheFollowingPeople(Table table)
    {
        var people = table.CreateSet<Person>();
        _peopleRepositoryStub.AddRange(people);
    }

    [When(@"person (.*) moves to '(.*)'")]
    public void WhenPersonMovesTo(int personId, string newAddress)
    {
        _movingService.MovePerson(personId, newAddress);
    }

    [Then(@"the new address of person (.*) is '(.*)'")]
    public void ThenTheNewAddressOfPersonIs(int personId, string expectedAddress)
    {
        var person = _peopleRepositoryStub.GetById(personId);
        Assert.AreEqual(expectedAddress, person.Address);
    }
}
```

## Refactoring our scenario

If we look at the scenario again you can see an technical integer is used as an id to identify a person.

```Gherkin
Given the following people
    | Id | Address                           |
    | 1  | 221B Baker Street, London, UK     |
    | 2  | 1630 Revello Drive, Sunnydale, US |
    | 3  | 31 Spooner Street, Quahog, US     |
    | 4  | 12 Grimmauld Place, London, UK    |
When person 3 moves to '742 Evergreen Terrace, Springfield, US'
Then the new address of person 3 is '742 Evergreen Terrace, Springfield, US'
```

For our test automation code the id is super helpful because we can just pass it into to the `MovingService.MovePerson` method. For business people, requirements engineers, etc., who might be less technical, this scenario is probably more difficult to read. Especially if the technical id is something like a `GUID` intead of an number.

It's better to look for a functional id to identify our person in this example. Preferably one that is commonly used by the business. Usually one or a combination of properties of an object can be use to uniquely identify that object. 

The name of a person is ideal for our specific scenario because it's often used in real life to identify a person. So we replace the technical id with the name of the person in our scenario.

```Gherkin
Given the following people
    | Name            | Address                           |
    | Sherlock Holmes | 221B Baker Street, London, UK     |
    | Buffy Summers   | 1630 Revello Drive, Sunnydale, US |
    | Peter Griffin   | 31 Spooner Street, Quahog, US     |
    | Sirius Black    | 12 Grimmauld Place, London, UK    |
When 'Peter Griffin' moves to '742 Evergreen Terrace, Springfield, US'
Then the new address of 'Peter Griffin' is '742 Evergreen Terrace, Springfield, US'
```
> Note that the functional id that you've chosen does not have to be a field that is unique within your system or database. Multiple people might have the same name in our system. However, as long as the name is unique within our scenario's, there is no problem.

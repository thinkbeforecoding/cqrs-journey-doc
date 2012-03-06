## Chapter 3
# Designing and Implementing BC #1 

Our first stopping point.

# A Description of the Attendee Registrations Bounded Context

Summary description of this Bounded Context. What is its relative 
importance/significance in the domain - is it core, how does it relate 
to the other bounded contexts? 

## Working Definitions for this Chapter

Outline any working definitions of CQRS/ES related concepts that were 
adopted for this chapter. 

## User Stories

Introductory comments.

### Domain Definitions (Ubiquitous Language)

The following list defines the key domain related terms that the team 
used during the development of this Attendee Registrations bounded 
context. 

- Attendee. An attendee is someone who has paid to attend a conference.
  An attendee can interact with the system to perform tasks such as:
  manage his agenda, print his badge, and provide feedback after the
  conference.
- Registrant. A registrant is a person who interacts with the system to
  make registrations and to make payments for those registrations. A
  registrant can register multiple attendees on a conference. A
  registrant may also be an attendee.
- Registration. Registration is the process...
- Wait-list. A wait-list is...
- Conference site. Every conference defined in the system can be
  accessed using a unique URL.
	
### Registration Stories

## Architecture

What are the key architectural features? Server-side, UI, multi-tier, 
cloud, etc. 

# Patterns and Concepts
- What are the primary patterns or approaches that have been adopted for
  this bounded context? (CQRS, CQRS/ES, CRUD, ...)
- What were the motivations for adopting these patterns or approaches
  for this bounded context? 
- What trade-offs did we evaluate?
- What alternatives did we consider?

The team decided that they would try to implement the first bounded 
context without using event sourcing in order to keep things simple. 
However, they did agree that if they later decided that event sourcing 
would bring specific benefits to this bounded context, then they would 
revisit this decision. 

One of the important discussions in the team was around the choice of 
aggregates and entities that they would implement. The following images 
from the team's whiteboard illustrate some of their initial thoughts, 
and questions about the alternative approaches they could take with a 
simple conference seat booking scenario to try and understand the
alternative approaches.

This scenario considers what happens when a registrant tries to book
several seats at a conference. The system must:

- Check that sufficient seats are available.
- Record details of the booking.
- Update the total number of seats booked for the conference.

> **Note:** The scenario is kept deliberately simple to avoid
  distractions while the team examines the alternatives.  

The first possible approach, shown in figure 1, uses two separate 
aggregates. 

![Figure 1][fig1]

**Approach #1, two separate aggregates**

The numbers in the diagram correspond to the following steps:

1. The UI sends a command to book X and Y onto conference #157. The 
   command is routed to a new **Booking** aggregate.
2. The **Booking** aggregate invokes a method on a **Conference**
   aggregate.
3. The **Conference** aggregate with an ID of 157 is re-hydrated from the
   data store.
4. The **Conference** aggregate updates its total number of seats
   booked.
5. The updated version of the **Conference** aggregate is persisted to
   the data store.
6. The new **Booking** aggregate, with an ID of 4239, is persisted to the
   data store.

The team identified these questions about the approach:

- If the **Booking** aggregate needs to know the total number of seats
  booked so far in order to determine whether the new **Booking** can be
  made: how does it get this information from the **Conference**
  aggregate?
- Should the **Booking** aggregate invoke a method on **Conference**
  aggregate or send a command?
- Where exactly is the transaction boundary?
- What happens if several **Booking** aggregates invoke the method on
  the **Conference** aggregate simultaneously?
- Should we really have two aggregates?

The second possible approach, shown in figure 2, uses a single 
aggregate in place of two. 

![Figure 2][fig2]

**Approach #2, a single aggregate**

The numbers in the diagram correspond to the following steps:

1. The UI sends a command to book X and Y onto conference #157. The 
   command is routed to the **Conference** aggregate with an ID of
   157.
2. The **Conference** aggregate with an ID of 157 is re-hydrated from the
   data store.
3. The **Booking** entity validates the booking (it queries the 
   **Conference Capacity** entity to see if there are enough seats
   left), and then invokes the method to update the number of seats
   booked on the conference entity.
4. The **Conference Capacity** entity updates its total number of seats
   booked.
5. The updated version of the **Conference** aggregate is persisted to
   the data store.

The team identified these questions about the approach:

- Which entity should be the aggregate root within the **Conference**
  aggregate?
- What else will end up in the **Conference aggregate**? Will it become
  too large.
- How does this approach handle multiple simultaneous bookings?

The third possible approach, shown in figure 3, uses a saga to 
coordinate the interaction between two aggregates.

![Figure 3][fig3]

**Approach #3, using a saga**

The numbers in the diagram correspond to the following steps:

1. The UI sends a command to book X and Y onto conference #157. The 
   command is routed to a new **Booking** aggregate.
2. The new **Booking** aggregate, with and ID of 4239,  is persisted to
   the data store.
3. The **Booking** aggregate raises an event that is handled by the
   **Booking** saga.
4. The **Booking** saga determines that a command should be sent to the
   **Conference** aggregate with an ID of 157.
5. The **Conference** aggregate is re-hydrated from the data store.
6. The total number of seats booked is updated in the **Conference**
   aggregate  and it is persisted to the data store.


The team identified these questions about the approach:

- Is using a saga overkill in this scenario?
- If the booking aggregate needs to know how many seats have been booked
  so far to validate the booking, how does it get this information from
  the conference aggregate?
- How does this approach handle multiple users making simultaneous
  bookings?
	
# Implementation Details
Describe significant features of the implementation with references to 
the code. Highlight any specific technologies (including any relevant 
trade-offs and alternatives). Provide significantly more detail for 
those BCs that use CQRS/ES. Significantly less detail for more 
"traditional" implementations such as CRUD.

# Testing
Describe any special considerations that relate to testing for this 
bounded context. 

Because this was the first bounded context the team tackled, one of the 
key concerns was how to approach testing given that the team wanted to 
adopt a Test-Driven Development approach. The following conversation 
summarizes their thoughts: 

**A conversation between two developers about how to do TDD when they 
are implementing the CQRS pattern without ES.** 


> *Developer #1*: If we were using event sourcing it would be easy to use 
> a TDD approach when we are creating our domain objects. The input to the 
> test would be a command (that perhaps originated in the UI), and we 
> could then test that the domain object fires the expected events. 
> However if we're not using event sourcing we don't have any events: the 
> behavior of the domain object is to persist its changes in data store 
> through an ORM layer. 

> *Developer #2*: So why don't we raise events anyway? Just because we're 
> not using event sourcing doesn't mean that our domain objects can't 
> raise events. We can then design our tests in the usual way to check for 
> the correct events firing in response to a command. 

> *Developer #1*: Isn't that just making things more complicated than they 
> need to be? One of the motivations for using CQRS is to simplify things! 
> We now have domain objects that need to persist their state using an ORM 
> layer, and also raise events reporting on what they have persisted just 
> so we can run our unit tests. 

> *Developer #2*: I see what you mean. 

> *Developer #1*: Perhaps we're getting hung up on how we're doing the 
> tests. Maybe instead of designing our tests based on the expected 
> *behavior* of the domain objects, we should think about testing the 
> state of the domain objects after they've processed a command? 

> *Developer #2*: That should be easy to do, after all the domain objects 
> will have all of the data we want to check stored in properties so that 
> the ORM can persist the right information to the store. 

> *Developer #1*: So we really just need to think about a different style 
> of testing in this scenario. 

> *Developer #2*: There is another aspect of this we'll need to consider: 
> we might have a set of tests that we can use to test our domain objects, 
> and all of those tests might be passing. We might also have a set of 
> tests to test that our ORM layer can save and retrieve objects 
> successfully. However, we will also have to test that our domain objects 
> function correctly when we run them against the ORM layer. It's possible 
> that a domain object performs the correct business logic, but can't 
> properly persist its state, perhaps because of a problem related to how 
> the ORM handles specific data types. 

For more information about the two approaches to testing discussed here, 
see Martin Fowler's article [Mocks Aren't Stubs][tddstyle]. 

The following code sample shows an example of a test written using the 
behavioural approach discussed above. 

The following code sample shows an example of a test written using the 
state of the objects being tested. 

[tddstyle]:		  http://martinfowler.com/articles/mocksArentStubs.html

[fig1]:           images/Journey_03_Aggregates_01.png?raw=true
[fig2]:           images/Journey_03_Aggregates_02.png?raw=true
[fig3]:           images/Journey_03_Aggregates_03.png?raw=true
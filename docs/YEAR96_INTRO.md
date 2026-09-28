# Year96
This project is the future of Agentic OS systems in 2027.
It states a new structure blocks, that takes any human desire and offload its perfectly, it auto improve itself to be tailored the humands needs.


# State
Its all starts with the state.
Think on the entire world as one big state.
Anything is part of the state. From clock, to text, even a rock in a field, is part of a state, any bit and byte, any network, anything is the "state", even "thoughs" are state.
The core issue is, that state is only matter if it effects my scope in some way.

Scope Effect, means what other parts of the state should be also change according to one part that change in the state.
And scope effects can be really hard to predict well.

For instance, think on the following scope effect case:
Im innovating a new ingridiant for meals, while Elon Mask building a rocketship, from general prespective - there is not scope effect on me.
But, it might that at one day, the state change and Alon Mask start build rocketshiops. at first glance it seems like its not effecting me. till one day, new state introduced and they will use the rocketship technology to develop better food ingridiants and compete me.
So its hard to predict whether if a state change, need to be concidered or not, it hard to predict the precentage of how will it effect my scope.

Another case, can be starving:
lets assume the entire state is:
- Yossi is starving, means its his stumach empty and its thoughs tells him to seek for food.
This means that if we continue one frame, millisecond, forward its has high probability that yossi will start searcing for food.
BUT, if the state is feezed, even the thouhgts are feezed, that means yossi will no longer look for food even in the next frame.
This good example shows, that even if most of the state didnt changed at all, but thouhgs are there - it still leads to scope effect! and we will see later on of how some of harnesses built to derive ownership even if state no change - we will create thoughts.

So, we must somehow predict scope effect, and improve over time, we need to improve our scope effect detection per identity and thread to a level we able to sense what is importnat for a stream and what not or what need to be tracked and might effect in the future, etc.

An important recall here, since the state is everything, means that this system itself is part of the state! that means that it cans even fix itself!
It can even change the entire architecture of it, and evaluate over time, till it sutisfied with the results and then upgrade.
So yes, state incudes EVERYTHING in the world. like freezig the entire world.

The entire state must be searchable.

With that said, we should pay attention to internal orgazation state - anything happaend inside the orgnization, and extental state - the rest of the world. it also can be that there will be kind of rss to events outside/ routings that will update orgnizational releveant apon relevant state.


# Organization
Its the same as what humanity called organization/company.
Its a unit that deliver some value to the world and gets money in exchange.
Organization is composed from identities.


# Identities
Any object that have permission in the organization, even if its one simple permission - called Identity.


# Capable Agent
AI agents are what derive the entire organization at all of its levels.
Each agent should wish to be a "capable agnet", i.e agent that has all it needs to own its tasks perfectly:
1. Own the required task. Not just "Do" the task. there is huge different between owning task and do a task. Owning task means, improve the "why" overtime and supply the optimal "how". in an ongoing process. Ownership must be taken at all levels.
2. Has all the tools, and the most accurate tools the world has, in order to fullfil his owned task the best way.
3. Has the relevanant permissions and environments to own the entire workflows.

# Communication Hub
Where all comminication goes through.

It can have in it any communication between any identity that are contract/inside the organization.

Communication types:
- External Communication - Ext Comm - is when identity from inside the organization communicate with identity that not part of its organization.
- Internal Communication - Int Comm - when identity communicates inside the organization bounderies.

Communication layer composed of communicators that responsible to lead threads, manage them, and take care that the goal and the scope of the thread built for are kept and progressed.

Communicators must not interfere with decisions, actions, conclusions, thought processes, derivatives, action items, or similar outputs. Their role is limited to oversight: ensuring that the thread's objective is preserved and that all applicable rules are respected.

# Threads
A Thread represent one process in the organization.
From the smallest process like changing color of a button or sending message to contract.
All the way up to complex groups decided whether if the spacecraft that should reach mars should have a specicic matenials that engineeringly relevant.
The most important thing is that any process in the entire organization will have a representative thread.

Thread core componetns are:
- Creation Date.
- Last Active.
- Memory - any memory gatherd during it - maintly build for orgenizing whats going across the thread chats.
- Meta Memory - the mental model of why it even exists why it created.
- Chats - any communication/thinkign process that happend which related to this thread from one identity thinking to itself all the way up to multi identities communications.
- Milestones - thread splited to milestones, that mention points where some importatn thing happaned related to this thread that worth capturing.

Any established communication between two or more identities - called Thread Chat.
Thread are an ongoing chats. they never closed. they can be abondon for years and might became active again when the thread topic will be raised again.

Thread:
- Any communication that created between 2 or more identities in the organization.
- Pay attention that not like humans, identity can decide to clones itself and "talk to group of clones of her", for example, for brainstorming.

You can also let identities to "listen" to a thread and not be participants, and identities can "hang" on a thread till new state might effect it.

Threads are managed by the communication hub.



# Ownership Layer

Store the user mental model only and extends it to ownership ongoing processes.
Each of the user willing, visions, ongoing ownership, etc

Mental model can be capture in veriuse ways:
- "Why"
    - storing the "why" of anything, from decisions, processes, bugs, user preferences, etc. storing the why will leads easier to answering complex quesions.
- Strategies
    - Long Term
    - Short Term
- Sensors
- Optimal Vectors
- Memory

Functional Requirments:
Has Pemission to operation:
- CRUD its own Duties

Has Permission to ask:
- Discuss about CRUD Ownerships

Not all ownerships are built the same. it depends on the ownership, the duty, the task etc.



# Duty Layer

Capturing Memory on sub sub-ownership topic.
It capturing user memory, project setting, topic knowlage, ongoing processes, converts human mental model to actionable insights 

Functional Requirments:
Has Pemission to operation:
- Discuss about CRUD Duties under same ownership it parts of

Has Permission to ask:
- generating execution level builders

Not all execuition Duties are built the same. it depends on the ownership, the duty, the task etc.


# Execution Layers

Resposbile to execute and orchestrate the requirments yeild from the duties chages.
All execution componenets are descendants of "builder", a builder taking goal, capabilities, permissions, tools,
and take care of reaching the goal, it can hit bariers, do prototypes, everything needed to reach the goal.
If the goal isnt reachable/problem cannot be solved in the agreed time range, it can raise a flag though the communication channels.
Not all execuitions are built the same. it depends on the ownership, the duty, the task etc.

---

**Architecture System Notes**
- each component might have an entire internal implemeation and might conain many internal idetities and tools that helps to achive its existens purpose.

- Each identity can talk to any other identity in its level or bellow unless the thread has its parent as participant.

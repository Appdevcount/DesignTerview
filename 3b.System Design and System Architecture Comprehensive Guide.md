System Design is the process of defining the architecture, components, modules, interfaces, and data for a system to satisfy specified requirements.

Involves translating user requirements into a detailed blueprint that guides the implementation phase.
The goal is to create a well-organized and efficient structure that meets the intended purpose while considering factors like scalability, maintainability, and performance.
Objectives of Systems Design
Practicality: We need a system that should be targetting the set of audiences(users) corresponding to which they are designing.
Accuracy: Should be designed in such a way it fulfills nearly all requirements around which it is designed be it functional or non-functional requirements.
Completeness: Should meet all user requirements
Efficient: Should make good use of resources—avoiding waste that increases costs and preventing underuse, which can slow down performance and increase response time (latency).
Reliability: Should be in proximity to a failure-free environment for a certain period of time.
Optimization: Time and space are just likely what we do for code chunks for individual components to work in a system.
Scalable(flexibility): Should be adaptable with time as per different user needs of customers which we know will keep on changing on time.
Objectives of System Design
Objectives of System Design
Note: System Design also helps us to achieve fault tolerence which is ability of a software to continue working where even its 1 or 2 component fails.

Advantages of System Design
After detailed discussion of the introduction to System Design, Some of the major advantages of System Design include:

Reduces the Design Cost of a Product: By using established design patterns and reusable components, teams can lower the effort and expense associated with creating new software designs.
Speedy Software Development Process: Using frameworks and libraries accelerates development by providing pre-built functionalities, allowing developers to focus on unique features.
Saves Overall Time in SDLC: Streamlined processes and automation in the Software Development Life Cycle (SDLC) lead to quicker iterations and faster time-to-market.
Increases Efficiency and Consistency of a Programmer: Familiar tools and methodologies enable programmers to work more effectively and produce uniform code, reducing the likelihood of errors.
Saves Resources: Optimized workflows and shared resources minimize the need for redundant efforts, thereby conserving both human and material resources.
The greatest advantage of system design is inculcating awareness and creativity in full-stack developers' via synergic bonding of API protocols gateways, networking and databases.

System Design Life Cycle (SDLC)
The System Design Life Cycle (SDLC) is a comprehensive process that outlines the steps involved in designing and developing a system, be it a software application, hardware solution, or an integrated system combining both. It encompasses a series of phases that guide engineers through the creation of a system that aligns with the user’s needs and organizational goals. The SDLC aims to ensure that the end product is reliable, scalable, and maintainable.

System-Design-Life-Cycle-22
System Design Life Cycle (SDLC)
System Architecture
System Architecture is a way in which we define how the components of a design are depicted design and deployment of software. It is basically the skeleton design of a software system depicting components, abstraction levels, and other aspects of a software system. In order to understand it in a layman's language, it is the aim or logic of a business should be crystal clear and laid out on a single sheet of paper. Here goals of big projects and further guides to scaling up are there for the existing system and upcoming systems to be scaled up.

System Architecture Patterns
There are various ways to organize the components in software or system architecture. And the different predefined organization of components in software architectures are known as software architecture patterns.  A lot of patterns were tried and tested. Most of them have successfully solved various problems. In each pattern, the components are organized differently for solving a specific problem in software architectures.

Types of System Architecture Patterns
1. Client-Server Architecture Pattern: Separates the system into two main components: clients that request services and servers that provide them.

2. Event-Driven Architecture Pattern: Uses events to trigger and communicate between decoupled components, enhancing responsiveness and scalability.

3. Microkernel Architecture Pattern: Centers around a core system (microkernel) with additional features and functionalities added as plugins or extensions.

4. Microservices Architecture Pattern: Breaks down applications into small, independent services that can be developed, deployed, and scaled independently.

System Architecture Patterns
System Architecture Patterns
Modularity and Interfaces In System Design
Modularity and interfaces in systems design are essential concepts that enhance flexibility and usability by breaking down complex systems into manageable components and providing intuitive user interactions.

1. Modularity
Modular design involves breaking down complex products into smaller, independent components or modules. This allows each module (e.g., a car's engine or transmission) to be developed and tested separately, making the overall system more flexible and easier to manage. The final product is assembled by integrating these modules, enabling changes without affecting the entire system.

2. Interfaces
In systems design, interfaces are the points where users interact with the system. This includes navigation elements, data input forms, and report displays. Effective interfaces are intuitive and user-friendly, enhancing the overall user experience and ensuring efficient data collection and system navigation. Together, modularity and well-designed interfaces contribute to creating scalable, maintainable, and user-friendly systems.

Evolution/Upgrade/Scaling of an Existing System
With increasing tech usage, it’s crucial for developers to design scalable systems. If a system is not scalable, it is likely to crash with the increase in users. Hence the concept of scaling comes into effect. Suppose there is a system with configurations of specific disk and RAM which was handling tasks. Now if we need to evolve our system or scale up, we have two options with us.

1. Upgrade Specifications of existing system(Vertical Scaling):
We are simply improving the processor by upgrading the RAM and disk size and many other components. Note that here we are not caring about the scalability and availability of network bandwidth. Here as per evolution we are working over the availability factor only considering scalability will be maintained. This is known as vertical scaling.

2. Create a Distributed System by connecting multiple systems together(Horizontal Scaling):
Horizontal scaling involves connecting multiple systems together to scale up. This method allows for better scalability and fault tolerance by using multiple systems instead of upgrading individual components. In order to scale up, we need more systems (more chunks of blocks) and this is known as horizontal scaling.

Evolution/Upgrade/Scale of an Existing System
Evolution/Upgrade/Scale of an Existing System
How Data Flows Between Systems?
Data Flow Diagrams or DFDs is defined as a graphical representation of the flow of data through information. DFDs are designed to show how a system is divided into smaller portions and to highlight the flow of data between these parts. Below is an example to demonstrate the Data Flow Diagram's basic structure:

Basic-Structure-of-DFD
Basic Stucture of DFD
Components of DFD include:
Representation	Action performed
Square	Defines the source of destination of data
Arrow	Identifies data flow and acts as a pipeline throughwhich information flows
Circle/Bubble	Represents a process that transforms incoming data flow into outgoing data
Open Rectangle	It is a data store or data at rest/temporary repository of data
 Note:  Sender and Receiver should  be written in uppercase always. Rather it is good practrice to use uppercaswe letter what so ever is placed in square box as per DFD conventions. 

System Design Example: Airline Reservation System
Having discussed the fundamentals of system design, we can now explore a practical example: the Airline Reservation System. This system will help illustrate the various components and design considerations involved. To better understand the Airline Reservation System, let's first examine its context-level flow diagram (DFD). In this diagram, key entities—Passenger, Travel Agent, and Airline—serve as the primary data sources and destinations.

System Design Example: Airline Reservation System
System Design Example: Airline Reservation System
Data Flow: The flow diagram illustrates how data moves through the system. For instance, when a Passenger wants to book a flight, they initiate a travel request, which is represented by an arrow pointing towards the system.
Interaction with Travel Agent and Airline: The request is then transmitted to two main entities: the Travel Agent and the Airline. The Travel Agent checks seat availability and preferences, sending an air flight request to the Airline.
Ticketing Process: If a seat is available, the Travel Agent proceeds to issue a ticket based on the Passenger's request. This interaction is captured in the flow diagram as the ticketing process.
Handling Unavailability: If no tickets are available, the system generates a request for Passenger Reservation, indicating that the Airline must manage and inform the Passenger about their reservation options.
This context-level flow diagram effectively encapsulates the interactions and data flow among the various components of the Airline Reservation System, providing a clear overview of how the system operates and how users engage with it.

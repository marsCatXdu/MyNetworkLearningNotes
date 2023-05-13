# Quality of Service (QoS) - reading note

> A reading note for [Computer Network | Quality of Service and Multimedia - GeeksforGeeks](https://www.geeksforgeeks.org/computer-network-quality-of-service-and-multimedia/#)

QoS refers to traffic control mechanisms. These mechanisms seek to provide performance for different requirements. Basic phenomenon for QoS means in terms of packet delay and losses of various kinds.

For example, video conferencing require bounded delay and loss rate.

QoS Requirements can be specified as:

- Delay
- Delay Variation (jitter)
- Throughput
- Error Rate

There're two types of QoS solutions, stateless one and stateful one. Stateless one is scalable and robust but cannot provide fine control, while stateful one can provide powerful services like guarenteed service, high resource utilization and providing protection.

Concepts:

- Integrated Services (IntServ): An architecture for providing QoS guarentees in IP networks for individual app sessions
- IntServ QoS Components: Resource reservation, QoS-sensitive scheduling, QoS-sensitive routing algorithm (QSPF), QoS-sensitive packet discard strategy
- RSVP-Internet Signaling: It creates and maintains distributed reservation state, initiated by the receiver and scales for multicast
- Call Admission: Session must first declare it's QoS requirement and characterize the traffic. Routers will admit calls based on QoS requested specs and current resource allocated at the routers to other calls
- Diff-Serv: Differentiated Service, provides reduced state services
### WebSockets

- **Full-duplex communication**: is a communication mode in which both endpoints send and receive data simultaneously at the same time over the same connection --> full‑duplex offers continuous bi‑directional flow—like a natural telephone conversation
- **Hall-duplex communication**: is a communication mode in which transmissions happen one direction at a time or simplex (one-way-only)

**Definition**: A protocol providing a persistent, full-duplex communication channel between client and server ==over a single TCP connection==. WebSockets enable real-time features (like chat, live notifications, and collaborative editing) without repeatedly polling the server

![Web Socket](./images/web-socket.png)

- **Handshake Upgrade** - Starts as an HTTP request that “upgrades” the connection to WebSocket, negotiating a persistent TCP link.

- **Full-Duplex Messaging** - Once upgraded, both client and server can send text or binary messages independently at any time.

- **Connection Lifecycle** - You must handle connection open, message receipt, errors, and clean closure to avoid resource leaks.

### TCP/IP Model

==The TCP/IP model== is a **framework** that is used to model the communication in a network. It is mainly a collection of network protocols and organization of these protocols in different layers for modeling the network.

- It has four layers, Application, Transport, Network/Internet and Network Access.
- While the OSI model has seven layers, the 4 layer TCP/IP model is simpler and commonly used in today’s Internet and networking systems.

--> _Goal_: The role of TCP/IP is to make sure data sent by the sender arrives safely and correctly at the receiver's end. To do this, data before being sent it will be broken down into smaller parts called ==packets==, and these packets are sent separately and are reassembled in the correct order when they reach destination

#### Layers of TCP/IP Model

![alt text](./images/tcp-model.png)

##### Application layer

The Application Layer is the top layer of the TCP/IP model and the one closest to the user. This is where all the apps you use like web browsers, email clients, or file sharing tools connect to the network. It acts like a bridge between your software (like Chrome, Gmail, or WhatsApp) and the lower layers of the network that actually send and receive data.

It supports different protocols like HTTP (for websites), FTP (for file transfers), SMTP (for emails), and DNS (for finding website addresses).

##### Transport layer

The Transport Layer is responsible for making sure that data is sent reliably and in the correct order between devices. It checks that the data you send like a message, file, or video arrives safely and completely. This layer uses two main protocols: TCP and UDP.

TCP is used when data must be correct and complete, like when loading a web page or downloading a file. It checks for errors, resends missing pieces, and keeps everything in order. On the other hand, UDP (User Datagram Protocol) is faster but doesn’t guarantee delivery useful for things like live video or online games where speed matters more than perfect accuracy.

##### Internet Layer

The Internet Layer is used for finding the best path for data to travel across different networks so it can reach the right destination. It works like a traffic controller, helping data packets move from one network to another until they reach the correct device. This layer uses the Internet Protocol (IP) to give every device a unique IP address, which helps identify where data should go.

##### Network Access Layer

The Network Access Layer is the bottom layer of the TCP/IP model. It deals with the actual physical connection between devices on the same local network like computers connected by cables or communicating through Wi-Fi.

#### How they work

**When Sending Data (From Sender to Receiver)**

- Application Layer: Prepares user data using protocols like HTTP, FTP, or SMTP.
- Transport Layer (TCP/UDP): Breaks data into segments and ensures reliable (TCP) or fast (UDP) delivery.
- Internet Layer (IP): Adds IP addresses and decides the best route for each packet.
- Link Layer (Network Access Layer): Converts packets into frames and sends them over the physical network.
  **When Receiving Data (At the Destination)**
- Link Layer: Receives bits from the network and rebuilds frames to pass to the next layer.
- Internet Layer: Checks the IP address, removes the IP header, and forwards data to the Transport Layer.
- Transport Layer: Reassembles segments, checks for errors, and ensures data is complete.
- Application Layer: Delivers the final data to the correct application (e.g., displays a web page in the browser).

### WebSockets
- **Full-duplex communication**: is a communication mode in which both endpoints send and receive data simultaneously at the same time over the same connection -->  full‑duplex offers continuous bi‑directional flow—like a natural telephone conversation
- **Hall-duplex communication**: is a communication mode in which transmissions happen one direction at a time or simplex (one-way-only)  

**Definition**: A protocol providing a persistent, full-duplex communication channel between client and server ==over a single TCP connection==. WebSockets enable real-time features (like chat, live notifications, and collaborative editing) without repeatedly polling the server

![Web Socket](./images/web-socket.png)

- **Handshake Upgrade** - Starts as an HTTP request that “upgrades” the connection to WebSocket, negotiating a persistent TCP link.

- **Full-Duplex Messaging** - Once upgraded, both client and server can send text or binary messages independently at any time.

- **Connection Lifecycle** - You must handle connection open, message receipt, errors, and clean closure to avoid resource leaks.


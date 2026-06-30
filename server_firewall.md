# How Cloud-based firewalls work

1. **Default-Deny Rule**
Bye default, cloud firewalls operate on a **Zero Trust** model. If a rule isn't created, explicitly allowing traffic to the server, the firewall will simply drop every connection attempt using `DENY ALL`.


2. **Anatomy of a firewall rule**

To let traffic through, it is necessary to define explicit whitelist rules in the Terraform configuration or the cloud console. Every rule requires **four** specifics data points to evaluate a packet:

* **Source Range (CIDR)**: Where is the traffic coming from? If set to 0.0.0.0/0, it means anywhere on the internet.

* **IP Protocol**: Is it standard web traffic (`TCP`), streaming/DNS traffic (`UDP`) or a ping request (`ICMP`)?

* **Source Port**: The random high port on the sender's web browser used to send the request (usually set to `Any` or *).

* **Destination Port**: The exact port on your server the traffic wants to talk to (e.g. `22` SSH, `80` HTTP or `443` HTTPS).


3. **Packet Evaluation**

This information is intercepted by the firewall and extracted from its **headers**. The rules are **matched** with the connection attempt and the firewall checks everything and read the rules from top to bottom to see if a connection from the IP 187.54.36.88 is allowed on the port `X`, if it isn't, it checks if the IP is allowed on another port, if yes, then the firewall allows the connection. In case it doesn't match any rule, the connection is `Dropped`. Then a **proxy manager** like Nginx takes control and routes it to the correct application.


4. **Why it's Stateful**

Modern cloud firewalls are Stateful. This is a huge feature for infrastructure automation because it eliminates half of the networking code part.

*Stateful* means the firewall remembers the choices it makes. When a request from a user on port `443` is allowed, the firewall opens a temporary, hidden back-door tracking record. When the web server compiles the webpage and sends the data back out to the user's browser, the firewall recognizes that this outbound traffic is part of the previously approved conversation.

Because it tracks the 'state' of the connection, it automatically alows the server's response packet out of the network, without requiring the writing of **outbound firewall rules**. It's only necessary to configure the *Ingress* rules, the *Egress* are handled by this smart cloud architecture.

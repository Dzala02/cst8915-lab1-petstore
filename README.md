# CST8915 Lab 1: Algonquin Pet Store on Azure VM

**Student Name**: Dhruvansh Zala
**Student ID**: 041214130
**Course**: CST8915 Full-stack Cloud-native Development
**Semester**: Fall 2026

---

## Demo Video

🎥 [Watch Demo Video](https://youtu.be/4qU-rRuc_LI)

---

## My Setup

I ran everything on an Azure VM called `petstore-vm` (Ubuntu 24.04, Standard B2als v2 with 2 vCPUs and 4 GB RAM) in the Sweden Central region. I connected to it from VS Code using the Remote - SSH extension and opened ports 8080, 3000, 3030 and 15672 in the Network Security Group so I could reach the app from my browser.

---

## Technical Explanations

### Order Service (Node.js)

The Order Service is the part of the app that takes orders from customers. When someone clicks "Place Order" on the website, the order gets sent here, and the service's job is to pass it on to RabbitMQ. It's written in JavaScript using Node.js and Express. Node.js makes sense for this because the service mostly just waits on network stuff (receiving HTTP requests and talking to RabbitMQ), and Node handles that kind of work well without blocking. It also uses `amqplib` to connect to RabbitMQ and the `cors` package so the browser is allowed to call it from a different port.

In the microservices setup, the Order Service works as a "producer". It has one endpoint, `POST /orders`, which receives the order as JSON from the Store Front. For every order, it connects to RabbitMQ on the same VM, makes sure a durable queue called `order_queue` exists, and publishes the order as a persistent message. What I found interesting is that it waits for RabbitMQ to confirm it actually saved the message before telling the browser "Order received". If something goes wrong, it sends back an error instead. This means taking an order and processing it are separated, so a different service could process the orders later without the two needing to be running at the same time. The Order Service doesn't talk to the Product Service at all.

### Product Service (Rust)

The Product Service gives the store its list of products. It has one endpoint, `GET /products`, that returns three products (Dog Food for $19.99, Cat Food for $34.99 and Bird Seeds for $10.99) with an ID, name and price for each. The products are written directly in the code instead of coming from a database, so the service doesn't store anything. It's built in Rust with the Warp web framework, which runs on Tokio (Rust's async runtime). Rust is a good choice for a small API like this because it compiles into a fast program that uses very little memory, and the compiler catches a lot of memory-related bugs before the code even runs.

This service is a good example of how microservices are independent. It could be changed, restarted or even rewritten in another language, and the rest of the app wouldn't care as long as it still returns the same JSON. It also shows that each service can use a different language, since here we have Rust, Node.js and Vue all working together. The only thing that talks to it is the Store Front, which calls it over HTTP from the browser. It listens on `0.0.0.0:3030` so it can be reached from outside the VM, and it has CORS turned on for GET requests so the browser accepts the response.

### Store Front (Vue.js)

The Store Front is the website that customers actually see and use. It shows the products, lets you pick one and enter a quantity, calculates the total, and sends the order. It's built with Vue.js, and most of the logic is in one component, `OrderForm.vue`. Vue is useful here because the page updates automatically when data changes. For example, when I picked Dog Food and changed the quantity to 2, the total changed to $39.98 right away without any extra code to refresh the page. It runs on port 8080 using `npm run serve`.

In the architecture, the Store Front is the only piece that talks to both backend services. When the page loads, it sends a GET request to the Product Service (port 3030) to get the products. When you place an order, it sends a POST request with the product, quantity and total price to the Order Service (port 3000). It never talks to RabbitMQ directly. One thing I learned during setup is that this code runs in my browser, not on the VM. That's why I had to change the URLs in `OrderForm.vue` from `localhost` to my VM's public IP. Otherwise my browser would have looked for the services on my own laptop. It's also why the ports had to be opened in Azure and why the backend services need CORS.

---

## Challenges and Learnings

- **Region problems with Azure for Students.** My VM kept failing validation with a `RequestDisallowedByAzure` error because my student subscription only allows certain regions, and the B2s size wasn't available in Canada Central. I ended up using Sweden Central with the B2als_v2 size, which has the same CPU and RAM.
- **SSH key permissions on Windows.** SSH refused to use my `.pem` key and said the permissions were "too open". I fixed it with `icacls` by removing all the extra permissions and giving read access only to my own account.
- **RabbitMQ login.** I got "Not_Authorized" the first time I tried the RabbitMQ dashboard. I checked the user with `rabbitmqctl list_users` and tested the password with `rabbitmqctl authenticate_user`, and it worked after that. I also learned the `guest` account only works from localhost, which is why a separate user is needed for the web UI.
- **localhost vs public IP.** At first it wasn't obvious why the Store Front needed the public IP. Once I understood the Vue code runs in my browser, it made sense.
- **Services stopping in VS Code.** When I opened a new folder in VS Code, the window reloaded and my three services stopped. I checked which ports were still listening with `ss -ltnp` and restarted them. RabbitMQ kept running because it's a system service, and my old orders were still in the queue because it's durable.

---

## Acknowledgments

- Lab instructions and starter code: [26F_Lab1_CST8915](https://github.com/ramymohamed10/26F_Lab1_CST8915)
- [RabbitMQ documentation](https://www.rabbitmq.com/docs)
- [Microsoft Azure documentation](https://learn.microsoft.com/azure/)

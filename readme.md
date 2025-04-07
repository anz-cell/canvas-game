# Problem

In this assignment we should develop a socket-based centralised DME using a coordinator node. The coordinator sequentially passes a unique token to each single node in the system that requests it. When the node is granted the unique token, the node should be allowed to execute its critical section, and on completion of the critical section, the node should give the given token back to the coordinator. All the nodes are connected to the coordinator.

## Assumptions

1. The system consists of n processes; each process Pi resides at a different processor.
2. Each process has a critical section that requires mutual exclusion.

## Requirement

1. If Pi is executing in its critical section, then no other process Pj is executing in its critical section.

## Approach

In this assignment I am using the DME: Centralized approach. In this approach:

1. One of the processes in the system is chosen to coordinate the entry to the critical section.
2. A process that wants to enter its critical section sends a request message to the coordinator.
3. The coordinator decides which process can enter the critical section next, and it sends that process a reply message. (It may have a number of requests queued.)
4. When the process receives a reply message from the coordinator, it enters its critical section.
5. After exiting its critical section, the process sends a release message to the coordinator and proceeds with its execution.
6. This scheme requires three messages per critical-section entry:
A. request
B. reply
C. release

## Implementation
## 1. Basic
## A. Each node will be running on a specific host and port (host and port must be configured at launch
## time, e.g. by using a command line parameter

While running Node.java this block of code is responsible for configuration of host and port. When running args[0] is the port number, args[2] host, args[1] the time for delay between requests.
```java
public static void main(String args[]) throws NumberFormatException, InterruptedException {
        String n_host_name = "";
        int n_port;

        // port and millisec (average waiting time) are specific of a node
        if ((args.length < 1) || (args.length > 3)) {
            System.out.print("Usage: Node [port number] [millisecs] [host]");
            System.exit(1);
        }

        // get the IP address and the port number of the node
        try {
            InetAddress n_inet_address = InetAddress.getLocalHost();
            n_host_name = n_inet_address.getHostName();
            n_host = args[2];
            System.out.println("node hostname is " + n_host_name + ":" + n_inet_address);
        } catch (java.net.UnknownHostException e) {
            System.out.println(e);
            System.exit(1);
        }

        n_port = Integer.parseInt(args[0]);
        System.out.println("node port is " + n_port);
        Node n = new Node(n_host_name, n_port, Integer.parseInt(args[1]));
    }
```

## B. request the token from the coordinator by passing the node's IP and port to the
## coordinator (the coordinator listens on port 7000 at a known IP address)

When the node is running it requests the token from the coordinator using this block of code and it does repeatedly from the Node constructor since while(true) and then after sleeping for few seconds as described when starting the node thr requestToken() method gets called. This method creates a socket connection to the coordinator at c_host:c_request_port where c_request_port is defined as 7000. It then sends the node's own host n_host and port n_port to the coordinator. Then it uses PrintWriter to send the data through the socket's output stream. This connects to the coordinator listening port 7000 and sends the node's the token.
```java
private void requestToken() throws IOException {
    try (Socket s = new Socket(c_host, c_request_port); PrintWriter pout = new PrintWriter(s.getOutputStream(), true)) {
        pout.println(n_host);
        pout.println(n_port);
        System.out.println("Node " + n_host + ":" + n_port + " sent token request");
    }
}
```

## C. wait until the token is granted by the coordinator. Granting the token can be
## implemented by a simple message the coordinator sends to the node, analogously to
## what you did in the token ring-based solution.

When the node is running and after the token has been granted it creates a server socket on the node's port (n_port). And then it calls n_ss.accept() which blocks until the coordinator connects - this is effectively the wait for the token. 
When the coordinator grants the token by connecting to the port, accept() call returns. The node then enters its critical section.
```java
private void processCriticalSection() throws IOException, InterruptedException {
    try (ServerSocket n_ss = new ServerSocket(n_port)) {
        System.out.println("Node " + n_host + ":" + n_port + " waiting for token...");
        try (Socket n_token = n_ss.accept()) {
            System.out.println("\n>>> Node " + n_host + ":" + n_port + " ENTERING critical section <<<");
            Thread.sleep(3000 + ra.nextInt(2000));
            System.out.println("<<< Node " + n_host + ":" + n_port + " EXITING critical section >>>\n");
        }
    }
}
```

After the token grant from the coordinator's side is in C_mutex.java where it connects to the waiting node
to signal the token grant.
```java
try (Socket s = new Socket(n_host, n_port)) {
    System.out.println("C:mutex   Token granted to " + n_host + ":" + n_port);
}
```

## D. execute the critical region, simulated by sleeping for about 3-5 secs, and outputting
## indicative messages marking the start and end of the critical section. Important: with
## multiple nodes running on the same machine, in different windows, it must be evident
## from the messages that only a single node at any one time is executing the critical
## session. Implement randomised sleep durations to experiment.

Now when its in the critical section there is clearly entry/exit messages and then random sleep duration 3000_ra.nextInt(2000) which creates a sleep between 3-5 seconds. Only one node can have the token at a time, so these 
messages will never overlap between nodes.
```java
private void processCriticalSection() throws IOException, InterruptedException {
    try (ServerSocket n_ss = new ServerSocket(n_port)) {
        System.out.println("Node " + n_host + ":" + n_port + " waiting for token...");
        try (Socket n_token = n_ss.accept()) {
            System.out.println("\n>>> Node " + n_host + ":" + n_port + " ENTERING critical section <<<");
            Thread.sleep(3000 + ra.nextInt(2000));
            System.out.println("<<< Node " + n_host + ":" + n_port + " EXITING critical section >>>\n");
        }
    }
}
```

## E. return the token to the coordinator. This can also be done through a simple message
## (the coordinator listens on port 7001).

This creates a socket connection to the coordinator at c_return_port defined as 7001 and then it sends the node's
identification host and port to confirm which node is returning the token. Then it uses PrintWriter to send the data through the socket's output stream.
```java
private void returnToken() throws IOException {
    try (Socket returnSocket = new Socket(c_host, c_return_port); PrintWriter returnPout = new PrintWriter(returnSocket.getOutputStream(), true)) {
        returnPout.println(n_host);
        returnPout.println(n_port);
        System.out.println("Node " + n_host + ":" + n_port + " returned token");
    }
}
```

The coordinator receives this token return in C_mutex.java where it listens on port 7001
```java
try (Socket returnSocket = ss_back.accept()) {
    System.out.println("C:mutex   Token returned from " + n_host + ":" + n_port);
}
```
## F. The coordinator consists of two concurrent tasks that share a buffer data structure:

The Coordinator starts 2 main threads C_receiver listens on port 7000 for new token requests and then C_mutex mamages token distribution and listens on port 7001 for the token returns.

Both the threads share a C_buffer for request queuing.
```java
import java.net.*;

public class Coordinator {
	
    public static void main (String args[]){
		int port = 7000;
		
		Coordinator c = new Coordinator ();
		
		try {    
		    InetAddress c_addr = InetAddress.getLocalHost();
		    String c_name = c_addr.getHostName();
		    System.out.println ("Coordinator address is "+c_addr);
		    System.out.println ("Coordinator host name is "+c_name+"\n\n");    
		}
		catch (Exception e) {
		    System.err.println(e);
		    System.err.println("Error in corrdinator");
		}
				
		// allows defining port at launch time
		if (args.length == 1) port = Integer.parseInt(args[0]);
	
		// Create and run a C_receiver and a C_mutex object sharing a C_buffer object
		C_buffer buffer = new C_buffer();
		C_receiver receiver = new C_receiver(buffer, port);
		C_mutex mutex = new C_mutex(buffer, 7001);
		receiver.start();
		mutex.start();

    }   
}
```

## G. a receiver that listens for requests from nodes. On connection from a client, the receiver
## will spawn a thread (C_connection_r) that receives IP and port and stores these in the
## buffer using a request data structure, defined in the code.

C_receiver continously listens for incoming connections and then for each connection it creates a new C_Connection_r thread
and this allows handling multiple requests concurrently.
```java
public void run () {
	
	// >>> create the socket the server will listen to
	try {
		s_socket = new ServerSocket(port);
	} catch (IOException e) {
		System.out.println("Exception creating server socket " + e);
	}
    System.out.println("C:receiver    Listening for requests on port " + port);

	while (true) {
	    try{
	        
			// >>> get a new connection
			socketFromNode = s_socket.accept();
			System.out.println ("C:receiver    Coordinator has received a request ...") ;
			
			// >>> create a separate thread to service the request, a C_Connection_r thread.
			connect = new C_Connection_r(socketFromNode, buffer);
			connect.start();

	    }
	    catch (java.io.IOException e) {
	    	System.out.println("Exception when creating a connection "+e);
	    }
	    
	}
}//end run
```

C_Connection_r then reads the node's connection details and creates a request object with node's information and stores request in shared buffer.
```java
public void run() {
        final int NODE = 0;
        final int PORT = 1;

        String[] request = new String[2];

        System.out.println("C:connection IN  dealing with request from socket " + s);
        try {

            // >>> read the request, i.e. node ip and port from the socket s
            // >>> save it in a request object and save the object in the buffer (see C_buffer's methods).
            in = s.getInputStream();
            bin = new BufferedReader(new InputStreamReader(in));

            request[NODE] = bin.readLine();
            request[PORT] = bin.readLine();

            if (request[NODE] == null || request[PORT] == null) {
                System.err.println("Invalid request received: " + Arrays.toString(request));
                return;
            }

            buffer.saveRequest(request);

            s.close();
            System.out.println("C:connection OUT    received and recorded request from " + request[NODE] + ":" + 
            request[PORT] + "  (socket closed)");

        } catch (java.io.IOException e) {
            System.out.println(e);
            System.exit(1);
        }
        buffer.show();

    }
```

C_buffer uses synchronised methods for threads and then it uses FIFO queue using vector and then blocking
the get() operation when empty and then it nofies for new requests.
```java
import java.util.*;

public class C_buffer {
	
    private final Vector<Object> data;
    
    public C_buffer() {
        data = new Vector<>(); 
    }    

    public int size(){
        return data.size();
    }

    /*
    This method is used to save the request to the queue
     */
    public synchronized void saveRequest(String[] r) {
        data.add(r);
        System.out.println("Request from " + r[0] + ":" + r[1] + " queued");
        show();
        notifyAll();
    }

    /*
    This method prints the current requests in the queue
     */
    public synchronized void show() {
        System.out.println("\nCurrent request queue status:");
        if (data.isEmpty()) {
            System.out.println("  [Queue Empty]");
            return;
        }
        for (int i = 0; i < data.size(); i++) {
            String[] request = (String[])data.get(i);
            System.out.println("  " + (i+1) + ". Node at " + request[0] + ":" + request[1]);
        }
        System.out.println();
    }

    public void add(Object o){
        data.add(o);
    }

    /*
    This method executes the requests
     */
    synchronized public Object get() {
        while (data.isEmpty()) {
            try {
                System.out.println("C:mutex   Buffer empty - waiting for requests...");
                wait();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException("Buffer wait interrupted", e);
            }
        }
        Object request = data.remove(0);
        System.out.println("C:mutex   Request dequeued - remaining requests: " + data.size());
        return request;
    }

}
```
## H. a mutex thread that continuously fetches available requests from the buffer in a FIFO
## order. Currently, this is a non-blocking call, change this to be a blocking call. Make sure
## you use correct synchronisation calls and remove any unnecessary if conditions. For
## each request, the mutex thread issues the token to the requesting node, by connecting
## to the node's indicated port. It then waits for the token to be returned by means of a
## synchronisation (on port 7001)

C_mutex then gets a next request from the buffer and grants token by connecting to requesting node
and then waits for the token return before processing next request and this ensures mutual exclusion
by waiting for the return from the node.

```java
import java.io.IOException;
import java.net.*;

public class C_mutex extends Thread {
    C_buffer buffer;
    Socket   s;
    int      port;

    // ip address and port number of the node requesting the token.
    // They will be fetched from the buffer
    String n_host;
    int    n_port;
    
    public C_mutex(C_buffer b, int p) {
        buffer = b;
        port = p;
    }

    public void run() {
        try (ServerSocket ss_back = new ServerSocket(port)) {
            // ip address and port number of the node requesting the token.
            // They will be fetched from the buffer
            System.out.println("C:mutex   Token manager started on port " + port);
            
            while (true) {
                // >>> Print some info on the current buffer content for debugging purposes.
                // >>> please look at the available methods in C_buffer
                System.out.println("C:mutex Buffer size is "+ buffer.size());

                // >>> Getting the first (FIFO) node that is waiting for a TOKEN form the buffer
                // Type conversions may be needed.
                String[] request = (String[])buffer.get();
                n_host = request[0];
                n_port = Integer.parseInt(request[1]);
                
                System.out.println("C:mutex   Processing token request from " + n_host + ":" + n_port);
                
                // >>> **** Granting the token
                //
                try (Socket s = new Socket(n_host, n_port)) {
                    System.out.println("C:mutex   Token granted to " + n_host + ":" + n_port);
                } catch (IOException e) {
                    System.err.println("C:mutex   Failed to grant token: " + e.getMessage());
                    continue;
                }
                
                // >>> **** Getting the token back
                System.out.println("C:mutex   Waiting for token return from " + n_host + ":" + n_port);
                try (Socket returnSocket = ss_back.accept()) {
                    System.out.println("C:mutex   Token returned from " + n_host + ":" + n_port);
                } catch (IOException e) {
                    System.err.println("C:mutex   Failed to receive token return: " + e.getMessage());
                }
            } // endwhile
        } catch (IOException e) {
            System.err.println("C:mutex   Fatal error: " + e.getMessage());
            throw new RuntimeException("Mutex manager failed", e);
        }
        
    }
}
```
## 2. Advance
## A. Implement a file logging mechanism using a single text file for all nodes and the
## coordinator. That is, nodes log their start of critical section and return of token to the
## coordinator in the file using timestamps. The coordinator may log token requests and
## issuing and queue length.

This feature is implemented in the Node.java class where it uses a single file DME.txt shared
by all nodes and coordinator this method is synchronized to handle concurrent writes from multiple
nodes and adds time stamps to each log entry, and then appends to file FileWriter(log_file, true)
```java
static String log_file = "DME.txt";

public static synchronized void logToFile(String message) {
    try (PrintWriter out = new PrintWriter(new FileWriter(log_file, true))) {
        String timestamp = LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
        out.println(timestamp + " - " + message);
    } catch (IOException e) {
        System.err.println("Error writing to log file: " + e.getMessage());
    }
    System.out.println(message);
}
```
This logging is used throughout the code like:
```java
Node.logToFile("\n=== Node " + n_host + ":" + n_port + " started with priority " + prio + " ===\n");

logToFile(">>> Node " + n_host + ":" + n_port + " ENTERING critical section <<<");
logToFile("<<< Node " + n_host + ":" + n_port + " EXITING critical section >>>");

logToFile("Node " + n_host + ":" + n_port + " sent token request");
logToFile("Node " + n_host + ":" + n_port + " returned token");
```

## B. Implement a priority discipline in the request queue. Imagine a critical process that will
## have priority in acquiring the token over all the other nodes. Implement a solution to avoid starvation of low priority nodes.

In Node.java the main() method accepts the priority as a command line argument and then the 
constructor converts text priority ("HIGH"/"MID"/"LOW") to numbers (2/1/0) and then the requestToken() sends node's priority to coordinator with token request and other methods dont have much changes for regarding this feature

```java
public class Node {
    int priority;

    public Node(String nam, int por, int sec, String prio) {
        if (prio.equalsIgnoreCase("HIGH")) {
            priority = 2;
        } else if (prio.equalsIgnoreCase("MID")) {
            priority = 1;
        } else {
            priority = 0;
        }
    }

    private void requestToken() throws IOException {
        try (Socket s = new Socket()) {
            s.connect(new InetSocketAddress(c_host, c_request_port), timeout);
            try (PrintWriter pout = new PrintWriter(s.getOutputStream(), true)) {
                pout.println(n_host);
                pout.println(n_port);
                pout.println(priority);  // Send priority
            }
        }
    }

    public static void main(String args[]) {
        String priority = (args.length == 4) ? args[3] : "LOW";
        Node n = new Node(n_host_name, n_port, Integer.parseInt(args[1]), priority);
    }
}
```

C_Connection_r.java receives and validates incoming priority requests in its run() method from the nodes and passes priority information from nodes to buffer by calling saveRequest() and other methods dont have much changes
```java
public class C_Connection_r extends Thread {
    public void run() {
        try {
            String[] request = new String[3];

            request[NODE] = bin.readLine();
            request[PORT] = bin.readLine();
            request[PRIORITY] = bin.readLine();

            if (request[NODE] == null || request[PORT] == null || request[PRIORITY] == null) {
                Node.logToFile("Invalid request received: " + Arrays.toString(request));
                return;
            }

            buffer.saveRequest(request);
        } catch (IOException e) {
            Node.logToFile("Error processing request: " + e.getMessage());
        }
    }
}
```

C_buffer.java uses saveRequest() method to save request with its priority in queue and then uses show()
to display all the queued requests with their priority and then the get method selects and removes highest
priority request from queue after it has been resolved when called by C_mutex and tis trakcs wait time for
each request and promoting reuests to HIGH priority after more than 10 second of waiting which reduces starvation of 
LOW and MID priority nodes.
```java
public class C_buffer {
    private Vector<Object> data;
private Vector<Long> timestamps; 
int age = 10;

    public synchronized void saveRequest(String[] r) {
        data.add(r);
        Node.logToFile("Request from " + r[0] + ":" + r[1] + " with priority " + r[2]);
        notifyAll();
    }

public synchronized void show() {
        Node.logToFile("\nCurrent request data status:");
        if (size() == 0) {
            Node.logToFile("  no requests");
            return;
        }

        long now = Instant.now().getEpochSecond();
        int count = 1;

        for (int i = 0; i < data.size(); i++) {
            String[] r = (String[]) data.get(i); 
            long waitTime = now - timestamps.get(i);
            int priority = Integer.parseInt(r[2]);
            String priorityStr = switch (priority) {
                case 2 -> "HIGH";
                case 1 -> "MID";
                default -> "LOW";
            };

            if (waitTime >= age) {
                priorityStr += " (AGED)";
            }

            Node.logToFile("  " + count + ". Node at " + r[0] + ":" + r[1] +
                          " (Priority: " + priorityStr + ", Waiting: " + waitTime + "s)" + "\n");
        }
    }

    public synchronized Object get() {
        while (data.isEmpty()) {
            try {
                wait();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException("Buffer wait interrupted", e);
            }
        }

        int selectedIndex = 0;
        String[] firstReq = (String[]) data.get(0);
        int maxPriority = Integer.parseInt(firstReq[2]);

        // Find highest priority request
        for (int i = 0; i < data.size(); i++) {
            String[] req = (String[]) data.get(i);
            int currentPriority = Integer.parseInt(req[2]);
            if (currentPriority > maxPriority) {
                selectedIndex = i;
                maxPriority = currentPriority;
            }
        }

        return data.remove(selectedIndex);
    }
}
```

C_mutex.java has a run method that processes requests in priority order using buffer's get() method
```java
public class C_mutex extends Thread {
    public void run() {
        try (ServerSocket ss_back = new ServerSocket(port)) {
            while (running) {
                // Get highest priority request
                String[] request = (String[]) buffer.get();
                n_host = request[0];
                n_port = Integer.parseInt(request[1]);

                try (Socket s = new Socket(n_host, n_port)) {
                    Node.logToFile("C:mutex   Token granted to " + n_host + ":" + n_port);
                } catch (IOException e) {
                    Node.logToFile("C:mutex   Failed to grant token: " + e.getMessage());
                    continue;
                }
            }
        } catch (IOException e) {
            Node.logToFile("C:mutex   Fatal error: " + e.getMessage());
        }
    }
}
```

## C. Modify the nodes so that they deal with the coordinator being closed down (or crashing!). Implement 
## full gracefully closing down of the system initiated by a closing down request initiated by a node.

The node initiates the request by taking a command line argument and sends that request to Coordinator and then 
Coordinator to others
```java
   public static void main(String args[]) throws NumberFormatException, InterruptedException {
        if (args.length == 1 && args[0].equalsIgnoreCase("shutdown")) {
            try (Socket shutdownSocket = new Socket(c_host, c_request_port);
                 PrintWriter pout = new PrintWriter(shutdownSocket.getOutputStream(), true)) {
                pout.println("SHUTDOWN");
                logToFile("Node sent shutdown request to Coordinator");
            } catch (IOException e) {
                logToFile("Failed to send shutdown request: " + e.getMessage());
            }
            return;
        }
}
```

When the C_Connection_r receives this shudown request from the Node its shuts down and also closes the return port 
by calling shutdown() in C_mutex
```java
public void run() {
        final int NODE = 0;
        final int PORT = 1;
        final int PRIORITY = 2;

        String[] request = new String[3];

        Node.logToFile("C:connection IN  dealing with request from socket " + s);
        try {

            // >>> read the request, i.e. node ip and port from the socket s
            // >>> save it in a request object and save the object in the buffer (see C_buffer's methods).
            in = s.getInputStream();
            bin = new BufferedReader(new InputStreamReader(in));

            request[NODE] = bin.readLine();

            if ("SHUTDOWN".equalsIgnoreCase(request[NODE])) {
                Node.logToFile("Shutdown command received from Node.");
                s.close();
                mutex.shutdown();
                System.exit(0);
            }

            request[PORT] = bin.readLine();
            request[PRIORITY] = bin.readLine();

            if (request[NODE] == null || request[PORT] == null || request[PRIORITY] == null) {
                Node.logToFile("Invalid request received: " + Arrays.toString(request));
                return;
            }

            buffer.saveRequest(request);

            s.close();
            Node.logToFile("C:connection OUT    received and recorded request from " + request[NODE] + ":" + request[PORT] + "  (socket closed)");

        } catch (IOException e) {
            Node.logToFile(e.getMessage());
            System.exit(1);
        }
        buffer.show();

    }
```

C_mutex also gets close when shutdown method is called by the C_Connection+_r
```java
public void shutdown() {
        running = false;
        interrupt();
    }
```

## References

1. DS-1 and DS-3 practical’s and lecture notes from these practical’s available in the Canvas.
2. Applied Operating Systems Concepts, A. Silberschatz, P. Galvin, G.Gagne, John Wiley & Sons
3. Concurrent Systems, J. Bacon, Addison-Wesley
4. Modern Operating Systems, A.S. Tanenbaum, Prentice-Hall
5. Distributed Systems, A.S. Tanenbaum, M. v. Steen, Prentice-Hall

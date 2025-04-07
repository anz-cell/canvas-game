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

## References

1. DS-1 and DS-3 practical’s and lecture notes from these practical’s available in the Canvas.
2. Applied Operating Systems Concepts, A. Silberschatz, P. Galvin, G.Gagne, John Wiley & Sons
3. Concurrent Systems, J. Bacon, Addison-Wesley
4. Modern Operating Systems, A.S. Tanenbaum, Prentice-Hall
5. Distributed Systems, A.S. Tanenbaum, M. v. Steen, Prentice-Hall

## Full Source Code Listing

1. Advanced

## A. Node.java

```java
import java.net.*;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.io.*;
import java.util.*;

public class Node {

    private Random ra;
    private Socket s;
    private PrintWriter pout = null;
    private ServerSocket n_ss;
    private Socket n_token;
    static String c_host = "127.0.0.1";
    static int c_request_port = 7000;
    int c_return_port = 7001;
    static String n_host = "127.0.0.1";
    String n_host_name;
    int n_port;

    //The following attributes are used for priority and for the retries in case the coordinator doesn't repsond
    int priority;
    int max_retries = 3;
    int delay = 3000;
    int timeout = 5000;

    // This attribute is used for logging the processes
    static String log_file = "DME.txt";

    public Node(String nam, int por, int sec, String prio) throws InterruptedException {
        ra = new Random();
        n_host_name = nam;
        n_port = por;

        if (prio.equalsIgnoreCase("HIGH")) {
            priority = 2;
        } else if (prio.equalsIgnoreCase("MID")) {
            priority = 1;
        } else {
            priority = 0;
        }

        Node.logToFile("\n=== Node " + n_host + ":" + n_port + " started with priority " + prio + " ===\n");

        // NODE sends n_host and n_port through a socket s to the coordinator
        // c_host:c_req_port
        // and immediately opens a server socket through which will receive
        // a TOKEN (actually just a synchronization).
        while (true) {
            // >>> sleep a random number of seconds linked to the initialisation sec value
            logToFile("Node " + n_host + ":" + n_port + " waiting " + sec + "ms before request");
            Thread.sleep(sec);
            try {
                // **** Send to the coordinator a token request.
                // send your ip address and port number
                requestToken();

                // **** Then Wait for the token
                // Print suitable messages
                // **** Sleep for a while
                // This is the critical session
                processCriticalSection();

                // **** Return the token
                // Print suitable messages - also considering communication failures
                returnToken();

            } catch (IOException e) {
                logToFile("Node " + n_host + ":" + n_port + " encountered error: " + e.getMessage());
                Thread.sleep(1000);
            }
        }
    }

    /*
    This method is used to send the coordinator a token request
     */
    private void requestToken() throws IOException, InterruptedException {
        int retries = 0;
        while (retries < max_retries) {
            try (Socket s = new Socket()) {
                s.connect(new InetSocketAddress(c_host, c_request_port), timeout);
                try (PrintWriter pout = new PrintWriter(s.getOutputStream(), true)) {
                    pout.println(n_host);
                    pout.println(n_port);
                    pout.println(priority);
                
                    logToFile("Node " + n_host + ":" + n_port + " sent token request");
                    return;
                }
            } catch (IOException e) {
                retries++;
                logToFile("Node " + n_host + ":" + n_port + " failed to request token (attempt " + retries + "/" + max_retries + "): " + e.getMessage());

                if (retries < max_retries) {
                    Thread.sleep(delay);
                } else {
                    logToFile("Node " + n_host + ":" + n_port + " the coordinator has been shutdown or crashed");
                    System.exit(1);
                }
            }
        }
    }

    /*
    This method is used to processing the critical section
     */
    private void processCriticalSection() throws IOException, InterruptedException {
        try (ServerSocket n_ss = new ServerSocket(n_port)) {
            n_ss.setSoTimeout(timeout);
            logToFile("Node " + n_host + ":" + n_port + " waiting for token...");
            try (Socket n_token = n_ss.accept()) {
                logToFile(">>> Node " + n_host + ":" + n_port + " ENTERING critical section <<<");
                Thread.sleep(3000 + ra.nextInt(2001));
                logToFile("<<< Node " + n_host + ":" + n_port + " EXITING critical section >>>");
            }
        }
    }

    /*
    This method is used send the token back to the mutex or the coordinator
     */
    private void returnToken() throws IOException, InterruptedException {
        int retries = 0;
        while (retries < max_retries) {
            try (Socket returnSocket = new Socket()) {
                returnSocket.connect(new InetSocketAddress(c_host, c_return_port), timeout);
                try (PrintWriter returnPout = new PrintWriter(returnSocket.getOutputStream(), true)) {
                    returnPout.println(n_host);
                    returnPout.println(n_port);
                
                    logToFile("Node " + n_host + ":" + n_port + " returned token");
                    return;
                }
            } catch (IOException e) {
                retries++;
                logToFile("Node " + n_host + ":" + n_port + " failed to return token (attempt " + retries + "/" + max_retries + "): " + e.getMessage());

                if (retries < max_retries) {
                    Thread.sleep(delay);
                } else {
                    throw e;
                }
            }
        }
    }
    
    // This method is used to log the processes of the system
    public static synchronized void logToFile(String message) {
        try (PrintWriter out = new PrintWriter(new FileWriter(log_file, true))) {
            String timestamp = LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
            out.println(timestamp + " - " + message);
        } catch (IOException e) {
            System.err.println("Error writing to log file: " + e.getMessage());
        }
        System.out.println(message);
    }
    
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

        String n_host_name = "";
        int n_port;

        // port and millisec (average waiting time) are specific of a node
        if ((args.length < 1) || (args.length > 4)) {
            System.out.print("Usage: Node [port number] [millisecs] [host] [priority]");
            System.exit(1);
        }

        // get the IP address and the port number of the node
        try {
            InetAddress n_inet_address = InetAddress.getLocalHost();
            n_host_name = n_inet_address.getHostName();
            n_host = args[2];
            Node.logToFile("node hostname is " + n_host_name + ":" + n_inet_address);
        } catch (java.net.UnknownHostException e) {
            Node.logToFile(e.getMessage());
            System.exit(1);
        }

        n_port = Integer.parseInt(args[0]);
        Node.logToFile("node port is " + n_port);

        // This attribute is used to set the default priority
        String priority = (args.length == 4) ? args[3] : "LOW";
        Node n = new Node(n_host_name, n_port, Integer.parseInt(args[1]), priority);
    }

} 
```

## B. C_buffer.java

```java
import java.util.*;
import java.time.Instant;
import java.net.*;
import java.io.*;

public class C_buffer {
    
    int age = 10;
    
    private Vector<Object> data; 
    private Vector<Long> timestamps; 

    public C_buffer() {
        data = new Vector<>();
        timestamps = new Vector<>();
    }

    public int size() {
        return data.size();
    }

    /*
    This method is used to save the request to the queue
     */
    public synchronized void saveRequest(String[] r) {
        data.add(r); 
        timestamps.add(Instant.now().getEpochSecond()); 

        Node.logToFile("Request from " + r[0] + ":" + r[1] + " with priority " + r[2]);
        show();
        notifyAll();
    }

    /*
    This method prints the current requests in the queue
     */
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

    public void add(Object o){
        data.add(o);
    }

    /*
    This method executes the requests
     */
    synchronized public Object get() {
        while (size() == 0) {
            try {
                Node.logToFile("C:mutex   Buffer empty - waiting for requests...");
                wait();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException("Buffer wait interrupted", e);
            }
        }

        long now = Instant.now().getEpochSecond();
        int selectedIndex = 0;

        String[] firstReq = (String[]) data.get(0); 
        int maxPriority = Integer.parseInt(firstReq[2]);
        long maxWait = now - timestamps.get(0);

        for (int i = 0; i < data.size(); i++) {
            String[] req = (String[]) data.get(i);
            int currentPriority = Integer.parseInt(req[2]);
            long waitTime = now - timestamps.get(i);

            if (waitTime > age) {
                currentPriority = 2;
            }

            if (currentPriority > maxPriority ||
                (currentPriority == maxPriority && waitTime > maxWait)) {
                selectedIndex = i;
                maxPriority = currentPriority;
                maxWait = waitTime;
            }
        }

        Object request = data.remove(selectedIndex);
        timestamps.remove(selectedIndex); 
        Node.logToFile("C:mutex   Request - remaining requests: " + size());
        return request;
    }
}
```

## C. C_Connection_r.java

```java

import java.net.*;
import java.util.Arrays;
import java.io.*;
// Reacts to a node request.
// Receives and records the node request in the buffer.
//

public class C_Connection_r extends Thread {

    // class variables
    C_buffer buffer;
    Socket s;
    InputStream in;
    BufferedReader bin;

    public C_Connection_r(Socket s, C_buffer b) {
        this.s = s;
        this.buffer = b;
    }

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
}
```

## D. Coordinator.java

```java
import java.net.*;

public class Coordinator {
    public static void main(String args[]) {
        int port = 7000;

        Coordinator c = new Coordinator();
        try {
            InetAddress c_addr = InetAddress.getLocalHost();
            String c_name = c_addr.getHostName();
            Node.logToFile("Coordinator address is " + c_addr);
            Node.logToFile("Coordinator host name is " + c_name +"\n\n");
        } catch (UnknownHostException e) {
            Node.logToFile("Error in coordinator " + e.getMessage());
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

## E. C_receiver.java

```java
import java.io.IOException;
import java.net.*;

public class C_receiver extends Thread {

    private C_buffer   buffer; 
    private int   port;
    private ServerSocket  s_socket; 
    private Socket  socketFromNode;
    private C_Connection_r connect;

    boolean running = true;

    public C_receiver(C_buffer b, int p) {
        buffer = b;
        port = p;
    }

    public void run() {

        // >>> create the socket the server will listen to
        try {
            s_socket = new ServerSocket(port);
        } catch (IOException e) {
            Node.logToFile("Exception creating server socket " + e);
        }
        Node.logToFile("C:receiver    Listening for requests on port " + port);

        while (running) {
            try {

                // >>> get a new connection
                socketFromNode = s_socket.accept();
                Node.logToFile("C:receiver    Coordinator has received a request ...");

                // >>> create a separate thread to service the request, a C_Connection_r thread.
                connect = new C_Connection_r(socketFromNode, buffer);
                connect.start();

            } catch (java.io.IOException e) {
                Node.logToFile("Exception when creating a connection " + e);
            }
        }
        Node.logToFile("C:receiver Stopped.");
    }//end run
} 
```

## F. C_mutex.java

```java
import java.io.IOException;
import java.net.*;

public class C_mutex extends Thread {

    C_buffer buffer;
    Socket   s;
    int port;

    // ip address and port number of the node requesting the token.
    // They will be fetched from the buffer
    String n_host;
    int n_port;

    boolean running = true;

    public C_mutex(C_buffer b, int p) {
        buffer = b;
        port = p;
    }

    public void run() {
        try (ServerSocket ss_back = new ServerSocket(port)) {
            // ip address and port number of the node requesting the token.
            // They will be fetched from the buffer
            Node.logToFile("C:mutex   Token manager started on port " + port);

            while (running) {
                // >>> Print some info on the current buffer content for debugging purposes.
                // >>> please look at the available methods in C_buffer
                Node.logToFile("C:mutex Buffer size is " + buffer.size());

                // >>> Getting the first (FIFO) node that is waiting for a TOKEN form the buffer
                // Type conversions may be needed.
                String[] request = (String[]) buffer.get();
                n_host = request[0];
                n_port = Integer.parseInt(request[1]);

                Node.logToFile("C:mutex   Processing token request from " + n_host + ":" + n_port);

                // >>> **** Granting the token
                //
                try (Socket s = new Socket(n_host, n_port)) {
                    Node.logToFile("C:mutex   Token granted to " + n_host + ":" + n_port);
                } catch (IOException e) {
                    Node.logToFile("C:mutex   Failed to grant token: " + e.getMessage());
                    continue;
                }

                // >>> **** Getting the token back
                Node.logToFile("C:mutex   Waiting for token return from " + n_host + ":" + n_port);
                try (Socket returnSocket = ss_back.accept()) {
                    Node.logToFile("C:mutex   Token returned from " + n_host + ":" + n_port);
                } catch (IOException e) {
                    Node.logToFile("C:mutex   Failed to receive token return: " + e.getMessage());
                }
            } // endwhile
        } catch (IOException e) {
            Node.logToFile("C:mutex   Fatal error: " + e.getMessage());
        }
        Node.logToFile("C:mutex Stopped.");
    }
}
```

2. Basic

## A. Node.java

```java

import java.net.*;
import java.io.*;
import java.util.*;

public class Node {

    private Random ra;
    private Socket s;
    private PrintWriter pout = null;
    private ServerSocket n_ss;
    private Socket n_token;
    String c_host = "127.0.0.1";
    int c_request_port = 7000;
    int c_return_port = 7001;
    static String n_host = "127.0.0.1";
    String n_host_name;
    int n_port;

    public Node(String nam, int por, int sec) throws InterruptedException {
        ra = new Random();
        n_host_name = nam;
        n_port = por;

        System.out.println("\n=== Node " + n_host + ":" + n_port + " started ===\n");

        // NODE sends n_host and n_port through a socket s to the coordinator
        // c_host:c_req_port
        // and immediately opens a server socket through which will receive
        // a TOKEN (actually just a synchronization).
        while (true) {
            // >>> sleep a random number of seconds linked to the initialisation sec value
            System.out.println("Node " + n_host + ":" + n_port + " waiting " + sec + "ms before next request");
            Thread.sleep(sec);
            try {
                // **** Send to the coordinator a token request.
                // send your ip address and port number
                requestToken();

                // **** Then Wait for the token
                // Print suitable messages
                // **** Sleep for a while
                // This is the critical session
                processCriticalSection();

                // **** Return the token
                // Print suitable messages - also considering communication failures
                returnToken();

            } catch (IOException e) {
                System.err.println("Node " + n_host + ":" + n_port + " encountered error: " + e.getMessage());
                Thread.sleep(1000);
            }
        }
    }

    /*
    This method is used to send the coordinator a token request
     */
    private void requestToken() throws IOException {
        try (Socket s = new Socket(c_host, c_request_port); PrintWriter pout = new PrintWriter(s.getOutputStream(), true)) {
            pout.println(n_host);
            pout.println(n_port);
            System.out.println("Node " + n_host + ":" + n_port + " sent token request");
        }
    }

    /*
    This method is used to processing the critical section
     */
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

    /*
    This method is used send the token back to the mutex or the coordinator
     */
    private void returnToken() throws IOException {
        try (Socket returnSocket = new Socket(c_host, c_return_port); PrintWriter returnPout = new PrintWriter(returnSocket.getOutputStream(), true)) {
            returnPout.println(n_host);
            returnPout.println(n_port);
            System.out.println("Node " + n_host + ":" + n_port + " returned token");
        }
    }

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

}
```

## B. C_buffer.java

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

## C. C_Connection_r.java

```java

import java.net.*;
import java.util.Arrays;
import java.io.*;
// Reacts to a node request.
// Receives and records the node request in the buffer.
//

public class C_Connection_r extends Thread {

    // class variables
    C_buffer buffer;
    Socket s;
    InputStream in;
    BufferedReader bin;

    public C_Connection_r(Socket s, C_buffer b) {
        this.s = s;
        this.buffer = b;
    }

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
            System.out.println("C:connection OUT    received and recorded request from " + request[NODE] + ":" + request[PORT] + "  (socket closed)");

        } catch (java.io.IOException e) {
            System.out.println(e);
            System.exit(1);
        }
        buffer.show();

    }
}
```

## D. Coordinator.java

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

## E. C_receiver.java

```java
 import java.io.IOException;
import java.net.*;

public class C_receiver extends Thread{
    
    private C_buffer   buffer; 
    private int   port;
    private ServerSocket  s_socket; 
    private Socket  socketFromNode;
    private C_Connection_r connect;
    
    public C_receiver (C_buffer b, int p){
  buffer = b;
  port = p;
    }
    
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
}
```

## F. C_mutex.java

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

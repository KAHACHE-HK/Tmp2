# Freelance Pattern with Heartbeating and Auto-Failover using PyZMQ

The Freelance pattern is a reliable request-reply pattern that provides automatic failover between multiple servers. Here's a complete implementation with heartbeating.

## Architecture Overview

The pattern consists of:
- **Clients** that can connect to multiple servers and automatically failover
- **Servers** that process requests and send heartbeats
- A **broker-less** design where clients connect directly to servers

## Server Implementation

```python
# server.py
import zmq
import time
import threading
import argparse

class FreelanceServer:
    """
    Freelance server that handles requests and sends heartbeats.
    """
    
    HEARTBEAT_INTERVAL = 1.0  # seconds
    
    def __init__(self, bind_address, server_id):
        self.bind_address = bind_address
        self.server_id = server_id
        self.context = zmq.Context()
        self.socket = self.context.socket(zmq.ROUTER)
        self.socket.setsockopt_string(zmq.IDENTITY, server_id)
        self.running = False
        
    def start(self):
        """Start the server."""
        self.socket.bind(self.bind_address)
        self.running = True
        print(f"[{self.server_id}] Server started on {self.bind_address}")
        
        # Start heartbeat thread
        heartbeat_thread = threading.Thread(target=self._heartbeat_loop, daemon=True)
        heartbeat_thread.start()
        
        # Main request processing loop
        self._process_requests()
        
    def _heartbeat_loop(self):
        """Send periodic heartbeats to all connected clients."""
        while self.running:
            time.sleep(self.HEARTBEAT_INTERVAL)
            # Heartbeats are sent as responses to PING requests
            
    def _process_requests(self):
        """Process incoming requests."""
        poller = zmq.Poller()
        poller.register(self.socket, zmq.POLLIN)
        
        while self.running:
            try:
                sockets = dict(poller.poll(1000))
                
                if self.socket in sockets:
                    message = self.socket.recv_multipart()
                    self._handle_message(message)
                    
            except zmq.ZMQError as e:
                if e.errno == zmq.ETERM:
                    break
                raise
                
    def _handle_message(self, message):
        """Handle incoming message."""
        if len(message) < 3:
            return
            
        client_id = message[0]
        empty = message[1]
        command = message[2].decode('utf-8')
        
        if command == "PING":
            # Respond to heartbeat ping
            self.socket.send_multipart([
                client_id,
                b"",
                b"PONG",
                self.server_id.encode('utf-8')
            ])
            print(f"[{self.server_id}] PING from {client_id.hex()[:8]}, sent PONG")
            
        elif command == "REQUEST":
            # Process actual request
            if len(message) > 3:
                request_data = message[3].decode('utf-8')
                print(f"[{self.server_id}] Processing request: {request_data}")
                
                # Simulate some work
                time.sleep(0.1)
                
                # Send response
                response = f"Response from {self.server_id}: processed '{request_data}'"
                self.socket.send_multipart([
                    client_id,
                    b"",
                    b"REPLY",
                    response.encode('utf-8')
                ])
                print(f"[{self.server_id}] Sent response")
                
    def stop(self):
        """Stop the server."""
        self.running = False
        self.socket.close()
        self.context.term()


def main():
    parser = argparse.ArgumentParser(description='Freelance Server')
    parser.add_argument('--port', type=int, default=5555, help='Port to bind to')
    parser.add_argument('--id', type=str, default='server-1', help='Server ID')
    args = parser.parse_args()
    
    server = FreelanceServer(f"tcp://*:{args.port}", args.id)
    
    try:
        server.start()
    except KeyboardInterrupt:
        print("\nShutting down...")
        server.stop()


if __name__ == "__main__":
    main()
```

## Client Implementation

```python
# client.py
import zmq
import time
import uuid
import threading
from collections import OrderedDict
from dataclasses import dataclass, field
from typing import Optional, List, Dict

@dataclass
class ServerInfo:
    """Information about a connected server."""
    endpoint: str
    alive: bool = True
    last_ping: float = field(default_factory=time.time)
    ping_sent: float = 0.0
    failures: int = 0


class FreelanceClient:
    """
    Freelance client with heartbeating and automatic failover.
    
    Features:
    - Connects to multiple servers
    - Automatic heartbeat monitoring
    - Failover to healthy servers
    - Round-robin load balancing among healthy servers
    """
    
    HEARTBEAT_INTERVAL = 1.0  # How often to send pings
    HEARTBEAT_TIMEOUT = 3.0   # How long before considering server dead
    MAX_FAILURES = 3          # Max failures before marking server dead
    REQUEST_TIMEOUT = 5.0     # Timeout for requests
    RECONNECT_INTERVAL = 10.0 # How often to try reconnecting dead servers
    
    def __init__(self):
        self.context = zmq.Context()
        self.client_id = uuid.uuid4().hex[:8]
        self.servers: Dict[str, ServerInfo] = OrderedDict()
        self.sockets: Dict[str, zmq.Socket] = {}
        self.running = False
        self.lock = threading.Lock()
        self.current_server_index = 0
        
    def connect(self, endpoints: List[str]):
        """Connect to multiple server endpoints."""
        for endpoint in endpoints:
            self._add_server(endpoint)
            
        self.running = True
        
        # Start heartbeat thread
        self.heartbeat_thread = threading.Thread(target=self._heartbeat_loop, daemon=True)
        self.heartbeat_thread.start()
        
        # Start receiver thread
        self.receiver_thread = threading.Thread(target=self._receive_loop, daemon=True)
        self.receiver_thread.start()
        
        print(f"[Client-{self.client_id}] Connected to {len(endpoints)} server(s)")
        
    def _add_server(self, endpoint: str):
        """Add a server connection."""
        socket = self.context.socket(zmq.DEALER)
        socket.setsockopt_string(zmq.IDENTITY, self.client_id)
        socket.setsockopt(zmq.LINGER, 0)
        socket.connect(endpoint)
        
        with self.lock:
            self.servers[endpoint] = ServerInfo(endpoint=endpoint)
            self.sockets[endpoint] = socket
            
        print(f"[Client-{self.client_id}] Added server: {endpoint}")
        
    def _remove_server(self, endpoint: str):
        """Remove a server connection."""
        with self.lock:
            if endpoint in self.sockets:
                self.sockets[endpoint].close()
                del self.sockets[endpoint]
            if endpoint in self.servers:
                del self.servers[endpoint]
                
    def _get_alive_servers(self) -> List[str]:
        """Get list of alive server endpoints."""
        with self.lock:
            return [ep for ep, info in self.servers.items() if info.alive]
            
    def _get_next_server(self) -> Optional[str]:
        """Get next alive server using round-robin."""
        alive = self._get_alive_servers()
        if not alive:
            return None
            
        with self.lock:
            self.current_server_index = self.current_server_index % len(alive)
            server = alive[self.current_server_index]
            self.current_server_index = (self.current_server_index + 1) % len(alive)
            return server
            
    def _heartbeat_loop(self):
        """Send periodic heartbeats to all servers."""
        last_reconnect = time.time()
        
        while self.running:
            current_time = time.time()
            
            # Send pings to all servers
            with self.lock:
                for endpoint, info in self.servers.items():
                    if endpoint in self.sockets:
                        try:
                            self.sockets[endpoint].send_multipart([
                                b"",
                                b"PING"
                            ], zmq.NOBLOCK)
                            info.ping_sent = current_time
                        except zmq.ZMQError:
                            pass
                            
            # Check for dead servers
            with self.lock:
                for endpoint, info in self.servers.items():
                    if info.alive and (current_time - info.last_ping) > self.HEARTBEAT_TIMEOUT:
                        info.failures += 1
                        if info.failures >= self.MAX_FAILURES:
                            info.alive = False
                            print(f"[Client-{self.client_id}] Server {endpoint} marked as DEAD")
                            
            # Try to reconnect dead servers periodically
            if current_time - last_reconnect > self.RECONNECT_INTERVAL:
                last_reconnect = current_time
                with self.lock:
                    for endpoint, info in self.servers.items():
                        if not info.alive:
                            # Reset and try again
                            info.failures = 0
                            info.alive = True
                            info.last_ping = current_time
                            print(f"[Client-{self.client_id}] Attempting reconnect to {endpoint}")
                            
            time.sleep(self.HEARTBEAT_INTERVAL)
            
    def _receive_loop(self):
        """Receive responses from all servers."""
        poller = zmq.Poller()
        
        with self.lock:
            for socket in self.sockets.values():
                poller.register(socket, zmq.POLLIN)
                
        while self.running:
            try:
                sockets = dict(poller.poll(100))
                
                with self.lock:
                    for endpoint, socket in self.sockets.items():
                        if socket in sockets:
                            try:
                                message = socket.recv_multipart(zmq.NOBLOCK)
                                self._handle_response(endpoint, message)
                            except zmq.ZMQError:
                                pass
                                
            except zmq.ZMQError as e:
                if e.errno == zmq.ETERM:
                    break
                    
    def _handle_response(self, endpoint: str, message: List[bytes]):
        """Handle response from server."""
        if len(message) < 2:
            return
            
        empty = message[0]
        command = message[1].decode('utf-8')
        
        if command == "PONG":
            # Heartbeat response
            with self.lock:
                if endpoint in self.servers:
                    self.servers[endpoint].last_ping = time.time()
                    self.servers[endpoint].failures = 0
                    if not self.servers[endpoint].alive:
                        self.servers[endpoint].alive = True
                        print(f"[Client-{self.client_id}] Server {endpoint} is back ALIVE")
                        
        elif command == "REPLY":
            # Request response - handled by request method
            if len(message) > 2:
                response_data = message[2].decode('utf-8')
                # Store response for the request method to pick up
                self._last_response = response_data
                self._response_received.set()
                
    def request(self, data: str, retries: int = 3) -> Optional[str]:
        """
        Send a request with automatic failover.
        
        Args:
            data: Request data to send
            retries: Number of retry attempts
            
        Returns:
            Response string or None if all servers failed
        """
        self._last_response = None
        self._response_received = threading.Event()
        
        for attempt in range(retries):
            server = self._get_next_server()
            
            if not server:
                print(f"[Client-{self.client_id}] No alive servers available!")
                time.sleep(1)
                continue
                
            print(f"[Client-{self.client_id}] Sending request to {server} (attempt {attempt + 1})")
            
            try:
                with self.lock:
                    if server in self.sockets:
                        self.sockets[server].send_multipart([
                            b"",
                            b"REQUEST",
                            data.encode('utf-8')
                        ])
                        
                # Wait for response
                if self._response_received.wait(timeout=self.REQUEST_TIMEOUT):
                    return self._last_response
                else:
                    print(f"[Client-{self.client_id}] Request timeout from {server}")
                    # Mark server as having a failure
                    with self.lock:
                        if server in self.servers:
                            self.servers[server].failures += 1
                            
            except zmq.ZMQError as e:
                print(f"[Client-{self.client_id}] Error sending to {server}: {e}")
                
        return None
        
    def get_server_status(self) -> Dict[str, dict]:
        """Get status of all servers."""
        with self.lock:
            return {
                ep: {
                    'alive': info.alive,
                    'failures': info.failures,
                    'last_ping': time.time() - info.last_ping
                }
                for ep, info in self.servers.items()
            }
            
    def stop(self):
        """Stop the client."""
        self.running = False
        
        with self.lock:
            for socket in self.sockets.values():
                socket.close()
            self.sockets.clear()
            
        self.context.term()


def main():
    # Example usage with multiple servers
    client = FreelanceClient()
    
    # Connect to multiple servers
    client.connect([
        "tcp://localhost:5555",
        "tcp://localhost:5556",
        "tcp://localhost:5557"
    ])
    
    # Wait for heartbeats to establish
    time.sleep(2)
    
    try:
        # Send some requests
        for i in range(10):
            print(f"\n--- Request {i + 1} ---")
            print(f"Server status: {client.get_server_status()}")
            
            response = client.request(f"Hello #{i + 1}")
            
            if response:
                print(f"Got response: {response}")
            else:
                print("Request failed - no response from any server")
                
            time.sleep(2)
            
    except KeyboardInterrupt:
        print("\nShutting down...")
    finally:
        client.stop()


if __name__ == "__main__":
    main()
```

## Demo Script

```python
# demo.py
"""
Demo script to run multiple servers and a client.
Run this to see the freelance pattern in action with failover.
"""

import subprocess
import time
import sys
import signal

def main():
    processes = []
    
    print("=" * 60)
    print("Freelance Pattern Demo with Heartbeating and Auto-Failover")
    print("=" * 60)
    print()
    
    # Start three servers
    print("Starting servers...")
    
    for i, port in enumerate([5555, 5556, 5557], 1):
        proc = subprocess.Popen(
            [sys.executable, "server.py", "--port", str(port), "--id", f"server-{i}"],
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            text=True
        )
        processes.append(proc)
        print(f"  Started server-{i} on port {port} (PID: {proc.pid})")
        
    time.sleep(1)
    
    print()
    print("Starting client...")
    print()
    print("-" * 60)
    print("Try killing one of the server processes to see failover!")
    print("Server PIDs:", [p.pid for p in processes])
    print("-" * 60)
    print()
    
    # Run the client
    try:
        client_proc = subprocess.Popen(
            [sys.executable, "client.py"],
            stdout=None,
            stderr=None
        )
        client_proc.wait()
        
    except KeyboardInterrupt:
        print("\nStopping demo...")
        
    finally:
        # Clean up
        for proc in processes:
            proc.terminate()
            proc.wait()
        print("All processes stopped.")


if __name__ == "__main__":
    main()
```

## Enhanced Version with Request Tracking

```python
# client_enhanced.py
"""
Enhanced freelance client with request tracking and callbacks.
"""

import zmq
import time
import uuid
import threading
from collections import OrderedDict
from dataclasses import dataclass, field
from typing import Optional, List, Dict, Callable
from queue import Queue, Empty

@dataclass
class PendingRequest:
    """Tracks a pending request."""
    request_id: str
    data: str
    sent_time: float
    server: str
    callback: Optional[Callable] = None
    response: Optional[str] = None
    completed: threading.Event = field(default_factory=threading.Event)


@dataclass
class ServerInfo:
    endpoint: str
    alive: bool = True
    last_ping: float = field(default_factory=time.time)
    failures: int = 0
    latency_ms: float = 0.0


class EnhancedFreelanceClient:
    """
    Enhanced Freelance client with:
    - Request tracking with unique IDs
    - Async callbacks
    - Latency tracking
    - Better failover logic
    """
    
    HEARTBEAT_INTERVAL = 1.0
    HEARTBEAT_TIMEOUT = 3.0
    MAX_FAILURES = 3
    REQUEST_TIMEOUT = 5.0
    
    def __init__(self):
        self.context = zmq.Context()
        self.client_id = uuid.uuid4().hex[:8]
        self.servers: Dict[str, ServerInfo] = OrderedDict()
        self.sockets: Dict[str, zmq.Socket] = {}
        self.pending_requests: Dict[str, PendingRequest] = {}
        self.running = False
        self.lock = threading.Lock()
        self.response_queue = Queue()
        
    def connect(self, endpoints: List[str]):
        """Connect to server endpoints."""
        for endpoint in endpoints:
            socket = self.context.socket(zmq.DEALER)
            socket.setsockopt_string(zmq.IDENTITY, self.client_id)
            socket.setsockopt(zmq.LINGER, 0)
            socket.connect(endpoint)
            
            with self.lock:
                self.servers[endpoint] = ServerInfo(endpoint=endpoint)
                self.sockets[endpoint] = socket
                
        self.running = True
        
        threading.Thread(target=self._heartbeat_loop, daemon=True).start()
        threading.Thread(target=self._receive_loop, daemon=True).start()
        
        print(f"[Client] Connected to {len(endpoints)} servers")
        
    def _get_best_server(self) -> Optional[str]:
        """Get the best available server based on latency and health."""
        with self.lock:
            alive_servers = [
                (ep, info) for ep, info in self.servers.items() 
                if info.alive
            ]
            
            if not alive_servers:
                return None
                
            # Sort by latency (lower is better) and failures (fewer is better)
            alive_servers.sort(key=lambda x: (x[1].failures, x[1].latency_ms))
            return alive_servers[0][0]
            
    def _heartbeat_loop(self):
        """Heartbeat monitoring loop."""
        while self.running:
            current_time = time.time()
            
            with self.lock:
                for endpoint, info in self.servers.items():
                    if endpoint in self.sockets:
                        try:
                            ping_id = uuid.uuid4().hex[:8]
                            self.sockets[endpoint].send_multipart([
                                b"",
                                b"PING",
                                ping_id.encode(),
                                str(current_time).encode()
                            ], zmq.NOBLOCK)
                        except zmq.ZMQError:
                            pass
                            
                    # Check for timeout
                    if info.alive and (current_time - info.last_ping) > self.HEARTBEAT_TIMEOUT:
                        info.failures += 1
                        if info.failures >= self.MAX_FAILURES:
                            info.alive = False
                            print(f"[Client] Server {endpoint} is DOWN")
                            
            time.sleep(self.HEARTBEAT_INTERVAL)
            
    def _receive_loop(self):
        """Receive and process responses."""
        poller = zmq.Poller()
        
        with self.lock:
            for socket in self.sockets.values():
                poller.register(socket, zmq.POLLIN)
                
        while self.running:
            try:
                events = dict(poller.poll(100))
                
                with self.lock:
                    for endpoint, socket in self.sockets.items():
                        if socket in events:
                            try:
                                msg = socket.recv_multipart(zmq.NOBLOCK)
                                self._handle_message(endpoint, msg)
                            except zmq.ZMQError:
                                pass
                                
            except zmq.ZMQError as e:
                if e.errno == zmq.ETERM:
                    break
                    
    def _handle_message(self, endpoint: str, message: List[bytes]):
        """Process incoming message."""
        if len(message) < 2:
            return
            
        command = message[1].decode()
        
        if command == "PONG" and len(message) >= 4:
            # Calculate latency
            sent_time = float(message[3].decode())
            latency = (time.time() - sent_time) * 1000
            
            if endpoint in self.servers:
                self.servers[endpoint].last_ping = time.time()
                self.servers[endpoint].failures = 0
                self.servers[endpoint].latency_ms = latency
                
                if not self.servers[endpoint].alive:
                    self.servers[endpoint].alive = True
                    print(f"[Client] Server {endpoint} is back UP (latency: {latency:.1f}ms)")
                    
        elif command == "REPLY" and len(message) >= 4:
            request_id = message[2].decode()
            response_data = message[3].decode()
            
            if request_id in self.pending_requests:
                req = self.pending_requests[request_id]
                req.response = response_data
                req.completed.set()
                
                if req.callback:
                    req.callback(response_data)
                    
    def request(self, data: str, callback: Optional[Callable] = None, 
                timeout: float = None) -> Optional[str]:
        """
        Send request with optional async callback.
        
        Args:
            data: Request payload
            callback: Optional callback function(response)
            timeout: Request timeout in seconds
            
        Returns:
            Response string or None on timeout/failure
        """
        timeout = timeout or self.REQUEST_TIMEOUT
        request_id = uuid.uuid4().hex
        
        for attempt in range(3):
            server = self._get_best_server()
            
            if not server:
                print("[Client] No servers available, waiting...")
                time.sleep(1)
                continue
                
            pending = PendingRequest(
                request_id=request_id,
                data=data,
                sent_time=time.time(),
                server=server,
                callback=callback
            )
            
            with self.lock:
                self.pending_requests[request_id] = pending
                
                try:
                    self.sockets[server].send_multipart([
                        b"",
                        b"REQUEST",
                        request_id.encode(),
                        data.encode()
                    ])
                except zmq.ZMQError as e:
                    print(f"[Client] Send error: {e}")
                    del self.pending_requests[request_id]
                    continue
                    
            # Wait for response
            if pending.completed.wait(timeout=timeout):
                with self.lock:
                    del self.pending_requests[request_id]
                return pending.response
            else:
                print(f"[Client] Timeout from {server}, trying next...")
                with self.lock:
                    if server in self.servers:
                        self.servers[server].failures += 1
                    if request_id in self.pending_requests:
                        del self.pending_requests[request_id]
                        
        return None
        
    def request_async(self, data: str, callback: Callable):
        """Send async request with callback."""
        threading.Thread(
            target=self.request,
            args=(data, callback),
            daemon=True
        ).start()
        
    def get_status(self) -> Dict:
        """Get detailed client status."""
        with self.lock:
            return {
                'servers': {
                    ep: {
                        'alive': info.alive,
                        'failures': info.failures,
                        'latency_ms': round(info.latency_ms, 2),
                        'last_seen': round(time.time() - info.last_ping, 1)
                    }
                    for ep, info in self.servers.items()
                },
                'pending_requests': len(self.pending_requests)
            }
            
    def stop(self):
        """Shutdown client."""
        self.running = False
        with self.lock:
            for socket in self.sockets.values():
                socket.close()
        self.context.term()


# Example usage
if __name__ == "__main__":
    client = EnhancedFreelanceClient()
    client.connect([
        "tcp://localhost:5555",
        "tcp://localhost:5556", 
        "tcp://localhost:5557"
    ])
    
    time.sleep(2)
    
    def on_response(resp):
        print(f"[Async] Got: {resp}")
    
    try:
        for i in range(20):
            print(f"\n=== Request {i+1} ===")
            print(f"Status: {client.get_status()}")
            
            # Sync request
            resp = client.request(f"sync-message-{i}")
            print(f"Response: {resp}")
            
            # Async request
            client.request_async(f"async-message-{i}", on_response)
            
            time.sleep(2)
            
    except KeyboardInterrupt:
        pass
    finally:
        client.stop()
```

## Updated Server for Enhanced Client

```python
# server_enhanced.py
"""
Enhanced server that works with the enhanced client.
"""

import zmq
import time
import argparse

class EnhancedServer:
    def __init__(self, bind_address: str, server_id: str):
        self.bind_address = bind_address
        self.server_id = server_id
        self.context = zmq.Context()
        self.socket = self.context.socket(zmq.ROUTER)
        self.socket.setsockopt_string(zmq.IDENTITY, server_id)
        
    def run(self):
        self.socket.bind(self.bind_address)
        print(f"[{self.server_id}] Listening on {self.bind_address}")
        
        poller = zmq.Poller()
        poller.register(self.socket, zmq.POLLIN)
        
        while True:
            try:
                if poller.poll(1000):
                    msg = self.socket.recv_multipart()
                    self._handle(msg)
            except KeyboardInterrupt:
                break
            except zmq.ZMQError:
                break
                
        self.socket.close()
        self.context.term()
        
    def _handle(self, msg):
        if len(msg) < 3:
            return
            
        client_id = msg[0]
        command = msg[2].decode()
        
        if command == "PING":
            # Echo back ping with timestamp for latency calc
            ping_id = msg[3] if len(msg) > 3 else b""
            timestamp = msg[4] if len(msg) > 4 else b""
            
            self.socket.send_multipart([
                client_id, b"", b"PONG", ping_id, timestamp
            ])
            
        elif command == "REQUEST":
            request_id = msg[3] if len(msg) > 3 else b""
            data = msg[4].decode() if len(msg) > 4 else ""
            
            print(f"[{self.server_id}] Request: {data}")
            time.sleep(0.05)  # Simulate work
            
            response = f"[{self.server_id}] Processed: {data}"
            self.socket.send_multipart([
                client_id, b"", b"REPLY", request_id, response.encode()
            ])


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument('--port', type=int, default=5555)
    parser.add_argument('--id', type=str, default='server-1')
    args = parser.parse_args()
    
    server = EnhancedServer(f"tcp://*:{args.port}", args.id)
    server.run()
```

## How to Run

1. **Start multiple servers** in separate terminals:
```bash
python server_enhanced.py --port 5555 --id server-1
python server_enhanced.py --port 5556 --id server-2
python server_enhanced.py --port 5557 --id server-3
```

2. **Run the client**:
```bash
python client_enhanced.py
```

3. **Test failover** by killing one of the server processes (Ctrl+C). The client will automatically detect the failure and route requests to the remaining healthy servers.

## Key Features

- **Heartbeat-based health monitoring**: Clients send periodic PING messages and track PONG responses
- **Automatic failover**: When a server becomes unresponsive, traffic is automatically routed to healthy servers
- **Latency-aware routing**: The enhanced client routes to servers with lower latency
- **Request tracking**: Each request has a unique ID for proper response matching
- **Graceful recovery**: Dead servers are periodically retried and restored when they come back online

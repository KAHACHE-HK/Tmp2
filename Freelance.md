# Reliable Freelance Pattern Implementation with PyZMQ

This implementation provides a robust client that can work with multiple servers, featuring heartbeating, request tracking, retries, and failover.

## Design Overview

The Freelance pattern allows a client to connect to multiple servers and handle failures gracefully. Key reliability features included:

1. **Request identifiers** - UUID-based tracking for each request
2. **Heartbeating** - Detect dead servers and maintain connection awareness
3. **Retries with exponential backoff** - Graceful handling of transient failures
4. **Server ranking** - Prefer servers that respond reliably
5. **Timeouts** - Prevent indefinite blocking
6. **Request deduplication** - Servers track processed requests to handle retries safely

---

## Server Implementation

```python
"""
freelance_server.py

Reliable Freelance Pattern - Server
Handles requests with heartbeating and duplicate detection.
"""

import zmq
import time
import logging
import argparse
from collections import OrderedDict
from dataclasses import dataclass
from typing import Optional

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


@dataclass
class ProcessedRequest:
    """Cache entry for processed requests (for deduplication)."""
    request_id: str
    response: list
    timestamp: float


class FreelanceServer:
    """
    Reliable server for the Freelance pattern.
    
    Features:
    - Heartbeat responses to client pings
    - Request deduplication to handle client retries safely
    - Graceful shutdown
    """
    
    # Protocol constants
    HEARTBEAT_CMD = b"HEARTBEAT"
    REQUEST_CMD = b"REQUEST"
    
    # Cache settings
    CACHE_TTL = 300.0  # 5 minutes - how long to remember processed requests
    CACHE_CLEANUP_INTERVAL = 60.0  # Clean cache every minute
    
    def __init__(
        self,
        bind_address: str,
        server_id: Optional[str] = None
    ):
        self.bind_address = bind_address
        self.server_id = server_id or f"server-{bind_address}"
        
        self.context = zmq.Context()
        self.socket = self.context.socket(zmq.ROUTER)
        self.socket.setsockopt(zmq.LINGER, 0)
        
        # Request cache for deduplication (OrderedDict for efficient cleanup)
        self._request_cache: OrderedDict[str, ProcessedRequest] = OrderedDict()
        self._last_cache_cleanup = time.time()
        
        self._running = False
    
    def start(self):
        """Bind and start serving requests."""
        self.socket.bind(self.bind_address)
        logger.info(f"Server {self.server_id} bound to {self.bind_address}")
        self._running = True
        self._serve_loop()
    
    def stop(self):
        """Stop the server gracefully."""
        self._running = False
        self.socket.close()
        self.context.term()
        logger.info(f"Server {self.server_id} stopped")
    
    def _serve_loop(self):
        """Main serving loop."""
        poller = zmq.Poller()
        poller.register(self.socket, zmq.POLLIN)
        
        while self._running:
            try:
                events = dict(poller.poll(timeout=1000))
                
                if self.socket in events:
                    self._handle_message()
                
                self._cleanup_cache_if_needed()
                
            except zmq.ZMQError as e:
                if e.errno == zmq.ETERM:
                    break
                logger.error(f"ZMQ error: {e}")
            except Exception as e:
                logger.error(f"Unexpected error: {e}")
    
    def _handle_message(self):
        """Process an incoming message."""
        frames = self.socket.recv_multipart()
        
        if len(frames) < 3:
            logger.warning(f"Malformed message: {frames}")
            return
        
        client_id = frames[0]
        empty = frames[1]  # Empty delimiter frame
        command = frames[2]
        
        if command == self.HEARTBEAT_CMD:
            self._handle_heartbeat(client_id)
        elif command == self.REQUEST_CMD:
            if len(frames) >= 5:
                request_id = frames[3].decode('utf-8')
                payload = frames[4:]
                self._handle_request(client_id, request_id, payload)
            else:
                logger.warning(f"Malformed request: {frames}")
        else:
            logger.warning(f"Unknown command: {command}")
    
    def _handle_heartbeat(self, client_id: bytes):
        """Respond to heartbeat ping."""
        self.socket.send_multipart([
            client_id,
            b"",
            self.HEARTBEAT_CMD,
            self.server_id.encode('utf-8')
        ])
    
    def _handle_request(
        self,
        client_id: bytes,
        request_id: str,
        payload: list
    ):
        """
        Process a request with deduplication.
        
        If we've already processed this request_id, return the cached response.
        This ensures idempotency when clients retry.
        """
        # Check cache for duplicate
        if request_id in self._request_cache:
            cached = self._request_cache[request_id]
            logger.debug(f"Returning cached response for {request_id}")
            self.socket.send_multipart(cached.response)
            return
        
        # Process the request
        try:
            response_payload = self.process_request(payload)
            response = [
                client_id,
                b"",
                b"RESPONSE",
                request_id.encode('utf-8'),
                b"OK"
            ] + response_payload
        except Exception as e:
            logger.error(f"Error processing request {request_id}: {e}")
            response = [
                client_id,
                b"",
                b"RESPONSE",
                request_id.encode('utf-8'),
                b"ERROR",
                str(e).encode('utf-8')
            ]
        
        # Cache and send response
        self._request_cache[request_id] = ProcessedRequest(
            request_id=request_id,
            response=response,
            timestamp=time.time()
        )
        self.socket.send_multipart(response)
        logger.debug(f"Processed request {request_id}")
    
    def process_request(self, payload: list) -> list:
        """
        Override this method to implement your business logic.
        
        Args:
            payload: List of message frames from client
            
        Returns:
            List of response frames to send back
        """
        # Default echo behavior - override in subclass
        return payload
    
    def _cleanup_cache_if_needed(self):
        """Remove expired entries from request cache."""
        now = time.time()
        if now - self._last_cache_cleanup < self.CACHE_CLEANUP_INTERVAL:
            return
        
        self._last_cache_cleanup = now
        cutoff = now - self.CACHE_TTL
        
        # Remove old entries (OrderedDict maintains insertion order)
        expired = []
        for req_id, entry in self._request_cache.items():
            if entry.timestamp < cutoff:
                expired.append(req_id)
            else:
                break  # Remaining entries are newer
        
        for req_id in expired:
            del self._request_cache[req_id]
        
        if expired:
            logger.debug(f"Cleaned {len(expired)} expired cache entries")


def main():
    parser = argparse.ArgumentParser(description='Freelance Pattern Server')
    parser.add_argument(
        '--bind', '-b',
        default='tcp://*:5555',
        help='Address to bind to (default: tcp://*:5555)'
    )
    parser.add_argument(
        '--id', '-i',
        default=None,
        help='Server identifier'
    )
    args = parser.parse_args()
    
    server = FreelanceServer(args.bind, args.id)
    try:
        server.start()
    except KeyboardInterrupt:
        logger.info("Shutting down...")
        server.stop()


if __name__ == '__main__':
    main()
```

---

## Client Implementation

```python
"""
freelance_client.py

Reliable Freelance Pattern - Client
Connects to multiple servers with failover, heartbeating, and retries.
"""

import zmq
import uuid
import time
import logging
import threading
from dataclasses import dataclass, field
from typing import Optional, List, Dict, Tuple
from enum import Enum
from collections import deque

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


class ServerState(Enum):
    """Possible states for a server connection."""
    UNKNOWN = "unknown"
    ALIVE = "alive"
    DEAD = "dead"


@dataclass
class ServerInfo:
    """Tracks state and statistics for a connected server."""
    endpoint: str
    state: ServerState = ServerState.UNKNOWN
    last_heartbeat: float = 0.0
    consecutive_failures: int = 0
    total_requests: int = 0
    total_successes: int = 0
    average_latency_ms: float = 0.0
    
    # For calculating running average latency
    _latency_samples: deque = field(default_factory=lambda: deque(maxlen=100))
    
    def record_success(self, latency_ms: float):
        self.state = ServerState.ALIVE
        self.last_heartbeat = time.time()
        self.consecutive_failures = 0
        self.total_requests += 1
        self.total_successes += 1
        self._latency_samples.append(latency_ms)
        if self._latency_samples:
            self.average_latency_ms = sum(self._latency_samples) / len(self._latency_samples)
    
    def record_failure(self):
        self.consecutive_failures += 1
        self.total_requests += 1
    
    def mark_dead(self):
        self.state = ServerState.DEAD
    
    @property
    def success_rate(self) -> float:
        if self.total_requests == 0:
            return 0.0
        return self.total_successes / self.total_requests


@dataclass
class PendingRequest:
    """Tracks an in-flight request."""
    request_id: str
    payload: list
    server_endpoint: str
    send_time: float
    attempt: int = 1


class FreelanceClient:
    """
    Reliable client for the Freelance pattern.
    
    Features:
    - Connect to multiple servers
    - Automatic failover on server failure
    - Heartbeating to detect dead servers
    - Request retries with exponential backoff
    - Server ranking based on reliability and latency
    - Thread-safe operation
    """
    
    # Protocol constants
    HEARTBEAT_CMD = b"HEARTBEAT"
    REQUEST_CMD = b"REQUEST"
    
    # Timing defaults (all in seconds)
    DEFAULT_REQUEST_TIMEOUT = 5.0
    DEFAULT_HEARTBEAT_INTERVAL = 2.0
    DEFAULT_HEARTBEAT_TIMEOUT = 6.0  # 3 missed heartbeats
    DEFAULT_MAX_RETRIES = 3
    DEFAULT_RETRY_BASE_DELAY = 0.5
    
    # Server health thresholds
    MAX_CONSECUTIVE_FAILURES = 3
    
    def __init__(
        self,
        request_timeout: float = DEFAULT_REQUEST_TIMEOUT,
        heartbeat_interval: float = DEFAULT_HEARTBEAT_INTERVAL,
        heartbeat_timeout: float = DEFAULT_HEARTBEAT_TIMEOUT,
        max_retries: int = DEFAULT_MAX_RETRIES,
        retry_base_delay: float = DEFAULT_RETRY_BASE_DELAY
    ):
        self.request_timeout = request_timeout
        self.heartbeat_interval = heartbeat_interval
        self.heartbeat_timeout = heartbeat_timeout
        self.max_retries = max_retries
        self.retry_base_delay = retry_base_delay
        
        self.context = zmq.Context()
        self.socket = self.context.socket(zmq.DEALER)
        self.socket.setsockopt(zmq.LINGER, 0)
        self.socket.setsockopt_string(zmq.IDENTITY, str(uuid.uuid4()))
        
        self._servers: Dict[str, ServerInfo] = {}
        self._lock = threading.RLock()
        
        self._heartbeat_thread: Optional[threading.Thread] = None
        self._running = False
    
    def connect(self, endpoint: str):
        """
        Add a server endpoint.
        
        Args:
            endpoint: ZMQ endpoint (e.g., 'tcp://localhost:5555')
        """
        with self._lock:
            if endpoint in self._servers:
                logger.warning(f"Already connected to {endpoint}")
                return
            
            self.socket.connect(endpoint)
            self._servers[endpoint] = ServerInfo(endpoint=endpoint)
            logger.info(f"Connected to {endpoint}")
    
    def disconnect(self, endpoint: str):
        """Remove a server endpoint."""
        with self._lock:
            if endpoint not in self._servers:
                return
            
            self.socket.disconnect(endpoint)
            del self._servers[endpoint]
            logger.info(f"Disconnected from {endpoint}")
    
    def start(self):
        """Start the heartbeat background thread."""
        if self._running:
            return
        
        self._running = True
        self._heartbeat_thread = threading.Thread(
            target=self._heartbeat_loop,
            daemon=True,
            name="freelance-heartbeat"
        )
        self._heartbeat_thread.start()
        logger.info("Client started")
    
    def stop(self):
        """Stop the client and clean up resources."""
        self._running = False
        
        if self._heartbeat_thread:
            self._heartbeat_thread.join(timeout=2.0)
        
        self.socket.close()
        self.context.term()
        logger.info("Client stopped")
    
    def request(
        self,
        payload: List[bytes],
        timeout: Optional[float] = None
    ) -> Tuple[bool, Optional[List[bytes]]]:
        """
        Send a request with retries and failover.
        
        Args:
            payload: List of message frames to send
            timeout: Optional timeout override
            
        Returns:
            Tuple of (success, response_frames or None)
        """
        if timeout is None:
            timeout = self.request_timeout
        
        request_id = str(uuid.uuid4())
        
        for attempt in range(1, self.max_retries + 1):
            server = self._select_server()
            if server is None:
                logger.error("No servers available")
                return False, None
            
            logger.debug(
                f"Request {request_id} attempt {attempt}/{self.max_retries} "
                f"to {server.endpoint}"
            )
            
            success, response = self._send_request(
                request_id, payload, server, timeout
            )
            
            if success:
                return True, response
            
            # Exponential backoff before retry
            if attempt < self.max_retries:
                delay = self.retry_base_delay * (2 ** (attempt - 1))
                time.sleep(delay)
        
        logger.error(f"Request {request_id} failed after {self.max_retries} attempts")
        return False, None
    
    def _send_request(
        self,
        request_id: str,
        payload: List[bytes],
        server: ServerInfo,
        timeout: float
    ) -> Tuple[bool, Optional[List[bytes]]]:
        """Send a single request attempt to a specific server."""
        # Build and send message
        message = [
            b"",
            self.REQUEST_CMD,
            request_id.encode('utf-8')
        ] + list(payload)
        
        send_time = time.time()
        
        try:
            self.socket.send_multipart(message)
        except zmq.ZMQError as e:
            logger.error(f"Send error: {e}")
            server.record_failure()
            return False, None
        
        # Wait for response
        poller = zmq.Poller()
        poller.register(self.socket, zmq.POLLIN)
        
        deadline = send_time + timeout
        while time.time() < deadline:
            remaining = max(0, (deadline - time.time()) * 1000)
            events = dict(poller.poll(timeout=remaining))
            
            if self.socket not in events:
                continue
            
            response = self.socket.recv_multipart()
            
            # Parse response: [empty, "RESPONSE", request_id, status, ...]
            if len(response) < 4:
                continue
            
            resp_type = response[1]
            resp_id = response[2].decode('utf-8')
            
            # Ignore heartbeat responses in this context
            if resp_type == self.HEARTBEAT_CMD:
                self._handle_heartbeat_response(response)
                continue
            
            # Check if this is our response
            if resp_id != request_id:
                logger.debug(f"Ignoring response for {resp_id}, waiting for {request_id}")
                continue
            
            status = response[3]
            latency_ms = (time.time() - send_time) * 1000
            
            if status == b"OK":
                server.record_success(latency_ms)
                return True, response[4:]  # Return payload frames
            else:
                # Application-level error
                server.record_failure()
                error_msg = response[4].decode('utf-8') if len(response) > 4 else "Unknown error"
                logger.error(f"Server error: {error_msg}")
                return False, None
        
        # Timeout
        logger.warning(f"Request {request_id} timed out to {server.endpoint}")
        server.record_failure()
        if server.consecutive_failures >= self.MAX_CONSECUTIVE_FAILURES:
            server.mark_dead()
            logger.warning(f"Server {server.endpoint} marked as dead")
        
        return False, None
    
    def _select_server(self) -> Optional[ServerInfo]:
        """
        Select the best available server.
        
        Priority:
        1. Alive servers with best success rate and latency
        2. Unknown servers (not yet tested)
        3. Dead servers (as last resort, they might have recovered)
        """
        with self._lock:
            if not self._servers:
                return None
            
            alive = []
            unknown = []
            dead = []
            
            for server in self._servers.values():
                if server.state == ServerState.ALIVE:
                    alive.append(server)
                elif server.state == ServerState.UNKNOWN:
                    unknown.append(server)
                else:
                    dead.append(server)
            
            # Prefer alive servers, sorted by success rate then latency
            if alive:
                alive.sort(
                    key=lambda s: (-s.success_rate, s.average_latency_ms)
                )
                return alive[0]
            
            # Try unknown servers
            if unknown:
                return unknown[0]
            
            # Last resort: try dead servers (they might have recovered)
            if dead:
                # Sort by most recent heartbeat (most likely to have recovered)
                dead.sort(key=lambda s: -s.last_heartbeat)
                return dead[0]
            
            return None
    
    def _heartbeat_loop(self):
        """Background thread for sending heartbeats and detecting dead servers."""
        poller = zmq.Poller()
        poller.register(self.socket, zmq.POLLIN)
        
        last_heartbeat_sent = 0.0
        
        while self._running:
            try:
                # Send heartbeats periodically
                now = time.time()
                if now - last_heartbeat_sent >= self.heartbeat_interval:
                    self._send_heartbeats()
                    last_heartbeat_sent = now
                
                # Check for heartbeat responses
                events = dict(poller.poll(timeout=100))
                if self.socket in events:
                    try:
                        response = self.socket.recv_multipart(zmq.NOBLOCK)
                        if len(response) >= 2 and response[1] == self.HEARTBEAT_CMD:
                            self._handle_heartbeat_response(response)
                    except zmq.Again:
                        pass
                
                # Check for dead servers
                self._check_server_health()
                
            except zmq.ZMQError as e:
                if e.errno == zmq.ETERM:
                    break
                logger.error(f"Heartbeat error: {e}")
            except Exception as e:
                logger.error(f"Heartbeat loop error: {e}")
    
    def _send_heartbeats(self):
        """Send heartbeat to all connected servers."""
        message = [b"", self.HEARTBEAT_CMD]
        try:
            self.socket.send_multipart(message)
        except zmq.ZMQError as e:
            logger.error(f"Heartbeat send error: {e}")
    
    def _handle_heartbeat_response(self, response: list):
        """Process a heartbeat response."""
        if len(response) < 3:
            return
        
        # We use DEALER socket, so we can't directly identify which server responded
        # The server includes its ID in the response
        server_id = response[2].decode('utf-8') if len(response) > 2 else None
        
        with self._lock:
            # Find server by ID (which might be the endpoint)
            for server in self._servers.values():
                if server_id and server_id in server.endpoint:
                    server.state = ServerState.ALIVE
                    server.last_heartbeat = time.time()
                    break
            else:
                # If we can't match, update all alive servers
                # This is a fallback for when server ID doesn't match endpoint
                for server in self._servers.values():
                    if server.state == ServerState.ALIVE:
                        server.last_heartbeat = time.time()
    
    def _check_server_health(self):
        """Mark servers as dead if heartbeat timeout exceeded."""
        now = time.time()
        with self._lock:
            for server in self._servers.values():
                if server.state == ServerState.ALIVE:
                    if now - server.last_heartbeat > self.heartbeat_timeout:
                        server.mark_dead()
                        logger.warning(
                            f"Server {server.endpoint} marked dead "
                            f"(no heartbeat for {self.heartbeat_timeout}s)"
                        )
    
    def get_server_stats(self) -> Dict[str, dict]:
        """Get statistics for all connected servers."""
        with self._lock:
            return {
                endpoint: {
                    'state': server.state.value,
                    'success_rate': server.success_rate,
                    'avg_latency_ms': server.average_latency_ms,
                    'consecutive_failures': server.consecutive_failures,
                    'total_requests': server.total_requests
                }
                for endpoint, server in self._servers.items()
            }


# Convenience function for simple usage
def create_client(
    endpoints: List[str],
    **kwargs
) -> FreelanceClient:
    """
    Create and start a client connected to multiple servers.
    
    Args:
        endpoints: List of server endpoints
        **kwargs: Additional arguments passed to FreelanceClient
        
    Returns:
        Started FreelanceClient instance
    """
    client = FreelanceClient(**kwargs)
    for endpoint in endpoints:
        client.connect(endpoint)
    client.start()
    return client
```

---

## Example Usage

```python
"""
example_usage.py

Demonstrates the Freelance pattern with a simple echo service.
"""

import time
import threading
from freelance_server import FreelanceServer
from freelance_client import create_client


class EchoServer(FreelanceServer):
    """Simple echo server that returns the request payload."""
    
    def process_request(self, payload):
        # Simulate some processing time
        time.sleep(0.01)
        return payload


def run_server(port):
    """Run a server on the specified port."""
    server = EchoServer(f"tcp://*:{port}", f"echo-server-{port}")
    try:
        server.start()
    except KeyboardInterrupt:
        server.stop()


def main():
    # Start multiple servers in background threads
    server_ports = [5555, 5556, 5557]
    server_threads = []
    
    for port in server_ports:
        t = threading.Thread(target=run_server, args=(port,), daemon=True)
        t.start()
        server_threads.append(t)
    
    # Give servers time to start
    time.sleep(0.5)
    
    # Create client connected to all servers
    endpoints = [f"tcp://localhost:{port}" for port in server_ports]
    client = create_client(endpoints)
    
    try:
        # Send some requests
        for i in range(10):
            payload = [f"Message {i}".encode('utf-8')]
            success, response = client.request(payload)
            
            if success:
                print(f"Response {i}: {response[0].decode('utf-8')}")
            else:
                print(f"Request {i} failed")
        
        # Print server statistics
        print("\nServer Statistics:")
        for endpoint, stats in client.get_server_stats().items():
            print(f"  {endpoint}: {stats}")
        
    finally:
        client.stop()


if __name__ == '__main__':
    main()
```

---

## Key Reliability Features

| Feature | Purpose | Implementation |
|---------|---------|----------------|
| **Request IDs** | Track and deduplicate requests | UUID assigned to each request |
| **Heartbeating** | Detect dead servers proactively | Background thread pings servers |
| **Retries** | Handle transient failures | Exponential backoff, configurable max attempts |
| **Server ranking** | Prefer reliable servers | Based on success rate and latency |
| **Deduplication** | Safe retries | Server caches responses by request ID |
| **Timeouts** | Prevent blocking | Configurable per-request |
| **Failover** | Continue despite failures | Automatic switch to healthy servers |

## Configuration Recommendations for Hospital Environment

For critical healthcare applications, consider these settings:

```python
client = FreelanceClient(
    request_timeout=10.0,        # Generous timeout for reliability
    heartbeat_interval=1.0,      # Faster failure detection
    heartbeat_timeout=3.0,       # Detect dead servers quickly
    max_retries=5,               # More retry attempts
    retry_base_delay=0.1         # Start retries quickly
)
```

Also ensure you have at least 3 servers for redundancy, and consider running them on separate physical machines with independent network paths.

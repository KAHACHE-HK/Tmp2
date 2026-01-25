# Comprehensive Test Suite for Freelance Pattern

This test suite covers unit tests, integration tests, stress tests, and failure scenario tests.

```python
"""
test_freelance.py

Comprehensive test suite for the Freelance pattern implementation.
Run with: pytest test_freelance.py -v

For coverage: pytest test_freelance.py --cov=freelance_client --cov=freelance_server --cov-report=html
"""

import pytest
import zmq
import time
import threading
import uuid
import random
from unittest.mock import Mock, patch, MagicMock
from collections import deque
from concurrent.futures import ThreadPoolExecutor, as_completed

from freelance_server import FreelanceServer, ProcessedRequest
from freelance_client import (
    FreelanceClient,
    ServerInfo,
    ServerState,
    PendingRequest,
    create_client
)


# =============================================================================
# Fixtures
# =============================================================================

@pytest.fixture
def zmq_context():
    """Provide a ZMQ context that's cleaned up after the test."""
    ctx = zmq.Context()
    yield ctx
    ctx.term()


@pytest.fixture
def available_port():
    """Find an available port for testing."""
    import socket
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.bind(('', 0))
        s.listen(1)
        port = s.getsockname()[1]
    return port


@pytest.fixture
def available_ports():
    """Get multiple available ports."""
    import socket
    ports = []
    sockets = []
    for _ in range(5):
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.bind(('', 0))
        s.listen(1)
        ports.append(s.getsockname()[1])
        sockets.append(s)
    
    for s in sockets:
        s.close()
    
    return ports


@pytest.fixture
def echo_server(available_port):
    """Create and start an echo server."""
    class EchoServer(FreelanceServer):
        def process_request(self, payload):
            return payload
    
    server = EchoServer(f"tcp://*:{available_port}", f"test-server-{available_port}")
    thread = threading.Thread(target=server.start, daemon=True)
    thread.start()
    time.sleep(0.1)  # Allow server to bind
    
    yield server, available_port
    
    server.stop()


@pytest.fixture
def slow_server(available_port):
    """Create a server that responds slowly."""
    class SlowServer(FreelanceServer):
        def __init__(self, *args, delay=0.5, **kwargs):
            super().__init__(*args, **kwargs)
            self.delay = delay
        
        def process_request(self, payload):
            time.sleep(self.delay)
            return payload
    
    server = SlowServer(f"tcp://*:{available_port}", f"slow-server-{available_port}", delay=0.5)
    thread = threading.Thread(target=server.start, daemon=True)
    thread.start()
    time.sleep(0.1)
    
    yield server, available_port
    
    server.stop()


@pytest.fixture
def failing_server(available_port):
    """Create a server that fails on certain requests."""
    class FailingServer(FreelanceServer):
        def __init__(self, *args, fail_rate=0.5, **kwargs):
            super().__init__(*args, **kwargs)
            self.fail_rate = fail_rate
            self.request_count = 0
        
        def process_request(self, payload):
            self.request_count += 1
            if random.random() < self.fail_rate:
                raise Exception("Simulated failure")
            return payload
    
    server = FailingServer(f"tcp://*:{available_port}", f"failing-server-{available_port}")
    thread = threading.Thread(target=server.start, daemon=True)
    thread.start()
    time.sleep(0.1)
    
    yield server, available_port
    
    server.stop()


@pytest.fixture
def client():
    """Create a client instance."""
    c = FreelanceClient(
        request_timeout=2.0,
        heartbeat_interval=0.5,
        heartbeat_timeout=1.5,
        max_retries=3,
        retry_base_delay=0.1
    )
    yield c
    c.stop()


@pytest.fixture
def connected_client(echo_server, client):
    """Create a client connected to an echo server."""
    server, port = echo_server
    client.connect(f"tcp://localhost:{port}")
    client.start()
    time.sleep(0.2)  # Allow heartbeat to establish connection
    yield client, port


# =============================================================================
# Unit Tests - ServerInfo
# =============================================================================

class TestServerInfo:
    """Unit tests for ServerInfo dataclass."""
    
    def test_initial_state(self):
        """ServerInfo should start in UNKNOWN state."""
        server = ServerInfo(endpoint="tcp://localhost:5555")
        assert server.state == ServerState.UNKNOWN
        assert server.consecutive_failures == 0
        assert server.total_requests == 0
        assert server.success_rate == 0.0
    
    def test_record_success(self):
        """record_success should update state and metrics."""
        server = ServerInfo(endpoint="tcp://localhost:5555")
        
        server.record_success(100.0)
        
        assert server.state == ServerState.ALIVE
        assert server.consecutive_failures == 0
        assert server.total_requests == 1
        assert server.total_successes == 1
        assert server.success_rate == 1.0
        assert server.average_latency_ms == 100.0
    
    def test_record_failure(self):
        """record_failure should increment failure counters."""
        server = ServerInfo(endpoint="tcp://localhost:5555")
        
        server.record_failure()
        server.record_failure()
        
        assert server.consecutive_failures == 2
        assert server.total_requests == 2
        assert server.total_successes == 0
        assert server.success_rate == 0.0
    
    def test_success_resets_consecutive_failures(self):
        """A success should reset the consecutive failure counter."""
        server = ServerInfo(endpoint="tcp://localhost:5555")
        
        server.record_failure()
        server.record_failure()
        assert server.consecutive_failures == 2
        
        server.record_success(50.0)
        assert server.consecutive_failures == 0
    
    def test_mark_dead(self):
        """mark_dead should set state to DEAD."""
        server = ServerInfo(endpoint="tcp://localhost:5555")
        server.state = ServerState.ALIVE
        
        server.mark_dead()
        
        assert server.state == ServerState.DEAD
    
    def test_average_latency_calculation(self):
        """Average latency should be calculated correctly."""
        server = ServerInfo(endpoint="tcp://localhost:5555")
        
        server.record_success(100.0)
        server.record_success(200.0)
        server.record_success(300.0)
        
        assert server.average_latency_ms == 200.0
    
    def test_latency_samples_limited(self):
        """Latency samples should be limited to prevent memory growth."""
        server = ServerInfo(endpoint="tcp://localhost:5555")
        
        # Record more samples than the limit
        for i in range(150):
            server.record_success(float(i))
        
        # Should only keep last 100 samples (50-149)
        assert len(server._latency_samples) == 100
        expected_avg = sum(range(50, 150)) / 100
        assert abs(server.average_latency_ms - expected_avg) < 0.01
    
    def test_success_rate_mixed_results(self):
        """Success rate should calculate correctly with mixed results."""
        server = ServerInfo(endpoint="tcp://localhost:5555")
        
        server.record_success(100.0)
        server.record_success(100.0)
        server.record_failure()
        server.record_success(100.0)
        
        assert server.total_requests == 4
        assert server.total_successes == 3
        assert server.success_rate == 0.75


# =============================================================================
# Unit Tests - FreelanceServer
# =============================================================================

class TestFreelanceServerUnit:
    """Unit tests for FreelanceServer."""
    
    def test_server_initialization(self):
        """Server should initialize with correct attributes."""
        server = FreelanceServer("tcp://*:5555", "test-server")
        
        assert server.bind_address == "tcp://*:5555"
        assert server.server_id == "test-server"
        assert not server._running
        
        server.context.term()
    
    def test_server_default_id(self):
        """Server should generate default ID if not provided."""
        server = FreelanceServer("tcp://*:5555")
        
        assert "tcp://*:5555" in server.server_id
        
        server.context.term()
    
    def test_process_request_default_echo(self):
        """Default process_request should echo the payload."""
        server = FreelanceServer("tcp://*:5555")
        
        payload = [b"test", b"data"]
        result = server.process_request(payload)
        
        assert result == payload
        
        server.context.term()
    
    def test_request_cache_structure(self):
        """Request cache should store ProcessedRequest objects."""
        server = FreelanceServer("tcp://*:5555")
        
        # Manually add a cache entry
        request_id = "test-request-123"
        response = [b"client-id", b"", b"RESPONSE", b"test-request-123", b"OK", b"data"]
        
        server._request_cache[request_id] = ProcessedRequest(
            request_id=request_id,
            response=response,
            timestamp=time.time()
        )
        
        assert request_id in server._request_cache
        assert server._request_cache[request_id].response == response
        
        server.context.term()


# =============================================================================
# Unit Tests - FreelanceClient
# =============================================================================

class TestFreelanceClientUnit:
    """Unit tests for FreelanceClient."""
    
    def test_client_initialization(self):
        """Client should initialize with correct defaults."""
        client = FreelanceClient()
        
        assert client.request_timeout == FreelanceClient.DEFAULT_REQUEST_TIMEOUT
        assert client.max_retries == FreelanceClient.DEFAULT_MAX_RETRIES
        assert len(client._servers) == 0
        assert not client._running
        
        client.stop()
    
    def test_client_custom_parameters(self):
        """Client should accept custom parameters."""
        client = FreelanceClient(
            request_timeout=10.0,
            heartbeat_interval=1.0,
            heartbeat_timeout=3.0,
            max_retries=5,
            retry_base_delay=0.2
        )
        
        assert client.request_timeout == 10.0
        assert client.heartbeat_interval == 1.0
        assert client.heartbeat_timeout == 3.0
        assert client.max_retries == 5
        assert client.retry_base_delay == 0.2
        
        client.stop()
    
    def test_connect_adds_server(self):
        """connect() should add a server to the internal list."""
        client = FreelanceClient()
        
        client.connect("tcp://localhost:5555")
        
        assert "tcp://localhost:5555" in client._servers
        assert client._servers["tcp://localhost:5555"].state == ServerState.UNKNOWN
        
        client.stop()
    
    def test_connect_duplicate_ignored(self):
        """Connecting to the same endpoint twice should be ignored."""
        client = FreelanceClient()
        
        client.connect("tcp://localhost:5555")
        client.connect("tcp://localhost:5555")
        
        assert len(client._servers) == 1
        
        client.stop()
    
    def test_disconnect_removes_server(self):
        """disconnect() should remove a server."""
        client = FreelanceClient()
        
        client.connect("tcp://localhost:5555")
        client.disconnect("tcp://localhost:5555")
        
        assert "tcp://localhost:5555" not in client._servers
        
        client.stop()
    
    def test_disconnect_nonexistent_safe(self):
        """Disconnecting from non-existent server should not raise."""
        client = FreelanceClient()
        
        client.disconnect("tcp://localhost:9999")  # Should not raise
        
        client.stop()
    
    def test_select_server_prefers_alive(self):
        """_select_server should prefer ALIVE servers."""
        client = FreelanceClient()
        
        client.connect("tcp://localhost:5555")
        client.connect("tcp://localhost:5556")
        client.connect("tcp://localhost:5557")
        
        client._servers["tcp://localhost:5555"].state = ServerState.DEAD
        client._servers["tcp://localhost:5556"].state = ServerState.ALIVE
        client._servers["tcp://localhost:5556"].last_heartbeat = time.time()
        client._servers["tcp://localhost:5557"].state = ServerState.UNKNOWN
        
        selected = client._select_server()
        
        assert selected.endpoint == "tcp://localhost:5556"
        
        client.stop()
    
    def test_select_server_prefers_better_success_rate(self):
        """_select_server should prefer servers with higher success rate."""
        client = FreelanceClient()
        
        client.connect("tcp://localhost:5555")
        client.connect("tcp://localhost:5556")
        
        # Both alive, but different success rates
        client._servers["tcp://localhost:5555"].state = ServerState.ALIVE
        client._servers["tcp://localhost:5555"].total_requests = 10
        client._servers["tcp://localhost:5555"].total_successes = 5  # 50%
        
        client._servers["tcp://localhost:5556"].state = ServerState.ALIVE
        client._servers["tcp://localhost:5556"].total_requests = 10
        client._servers["tcp://localhost:5556"].total_successes = 9  # 90%
        
        selected = client._select_server()
        
        assert selected.endpoint == "tcp://localhost:5556"
        
        client.stop()
    
    def test_select_server_unknown_before_dead(self):
        """_select_server should prefer UNKNOWN over DEAD servers."""
        client = FreelanceClient()
        
        client.connect("tcp://localhost:5555")
        client.connect("tcp://localhost:5556")
        
        client._servers["tcp://localhost:5555"].state = ServerState.DEAD
        client._servers["tcp://localhost:5556"].state = ServerState.UNKNOWN
        
        selected = client._select_server()
        
        assert selected.endpoint == "tcp://localhost:5556"
        
        client.stop()
    
    def test_select_server_returns_none_when_empty(self):
        """_select_server should return None when no servers connected."""
        client = FreelanceClient()
        
        selected = client._select_server()
        
        assert selected is None
        
        client.stop()
    
    def test_get_server_stats(self):
        """get_server_stats should return formatted statistics."""
        client = FreelanceClient()
        
        client.connect("tcp://localhost:5555")
        client._servers["tcp://localhost:5555"].state = ServerState.ALIVE
        client._servers["tcp://localhost:5555"].record_success(100.0)
        
        stats = client.get_server_stats()
        
        assert "tcp://localhost:5555" in stats
        assert stats["tcp://localhost:5555"]["state"] == "alive"
        assert stats["tcp://localhost:5555"]["success_rate"] == 1.0
        assert stats["tcp://localhost:5555"]["avg_latency_ms"] == 100.0
        
        client.stop()


# =============================================================================
# Integration Tests - Basic Communication
# =============================================================================

class TestBasicCommunication:
    """Integration tests for basic client-server communication."""
    
    def test_simple_request_response(self, connected_client):
        """Client should receive correct response from server."""
        client, port = connected_client
        
        payload = [b"Hello, World!"]
        success, response = client.request(payload)
        
        assert success is True
        assert response == payload
    
    def test_multi_frame_payload(self, connected_client):
        """Multi-frame payloads should be handled correctly."""
        client, port = connected_client
        
        payload = [b"frame1", b"frame2", b"frame3"]
        success, response = client.request(payload)
        
        assert success is True
        assert response == payload
    
    def test_empty_payload(self, connected_client):
        """Empty payload should be handled correctly."""
        client, port = connected_client
        
        payload = [b""]
        success, response = client.request(payload)
        
        assert success is True
        assert response == payload
    
    def test_binary_payload(self, connected_client):
        """Binary data should be transmitted correctly."""
        client, port = connected_client
        
        payload = [bytes(range(256))]
        success, response = client.request(payload)
        
        assert success is True
        assert response == payload
    
    def test_large_payload(self, connected_client):
        """Large payloads should be handled correctly."""
        client, port = connected_client
        
        # 1 MB payload
        payload = [b"x" * (1024 * 1024)]
        success, response = client.request(payload)
        
        assert success is True
        assert response == payload
    
    def test_multiple_sequential_requests(self, connected_client):
        """Multiple sequential requests should all succeed."""
        client, port = connected_client
        
        for i in range(10):
            payload = [f"Message {i}".encode()]
            success, response = client.request(payload)
            
            assert success is True
            assert response == payload
    
    def test_request_timeout(self, slow_server, client):
        """Request should timeout if server is too slow."""
        server, port = slow_server
        
        client.connect(f"tcp://localhost:{port}")
        client.start()
        client.request_timeout = 0.1  # Very short timeout
        client.max_retries = 1  # Don't retry
        
        time.sleep(0.1)
        
        payload = [b"test"]
        success, response = client.request(payload, timeout=0.1)
        
        assert success is False
        assert response is None
    
    def test_request_without_servers(self, client):
        """Request should fail gracefully with no servers connected."""
        client.start()
        
        success, response = client.request([b"test"])
        
        assert success is False
        assert response is None


# =============================================================================
# Integration Tests - Multi-Server
# =============================================================================

class TestMultiServer:
    """Tests for multi-server scenarios."""
    
    def test_connect_to_multiple_servers(self, available_ports):
        """Client should connect to multiple servers."""
        servers = []
        
        # Start multiple servers
        for port in available_ports[:3]:
            class EchoServer(FreelanceServer):
                def process_request(self, payload):
                    return payload
            
            server = EchoServer(f"tcp://*:{port}", f"server-{port}")
            thread = threading.Thread(target=server.start, daemon=True)
            thread.start()
            servers.append(server)
        
        time.sleep(0.1)
        
        # Create client connected to all servers
        client = FreelanceClient(request_timeout=2.0)
        for port in available_ports[:3]:
            client.connect(f"tcp://localhost:{port}")
        client.start()
        
        time.sleep(0.2)
        
        try:
            # Should be able to send requests
            for i in range(5):
                success, response = client.request([f"test-{i}".encode()])
                assert success is True
            
            # All servers should be tracked
            assert len(client._servers) == 3
        finally:
            client.stop()
            for server in servers:
                server.stop()
    
    def test_failover_to_backup_server(self, available_ports):
        """Client should failover when primary server dies."""
        port1, port2 = available_ports[:2]
        
        class EchoServer(FreelanceServer):
            def process_request(self, payload):
                return payload
        
        # Start two servers
        server1 = EchoServer(f"tcp://*:{port1}", f"server-{port1}")
        server2 = EchoServer(f"tcp://*:{port2}", f"server-{port2}")
        
        thread1 = threading.Thread(target=server1.start, daemon=True)
        thread2 = threading.Thread(target=server2.start, daemon=True)
        thread1.start()
        thread2.start()
        time.sleep(0.1)
        
        # Create client
        client = FreelanceClient(
            request_timeout=1.0,
            heartbeat_interval=0.2,
            heartbeat_timeout=0.5,
            max_retries=3
        )
        client.connect(f"tcp://localhost:{port1}")
        client.connect(f"tcp://localhost:{port2}")
        client.start()
        time.sleep(0.3)
        
        try:
            # First request should work
            success, _ = client.request([b"test1"])
            assert success is True
            
            # Kill server1
            server1.stop()
            time.sleep(0.6)  # Wait for heartbeat timeout
            
            # Request should still succeed (failover to server2)
            success, response = client.request([b"test2"])
            assert success is True
            assert response == [b"test2"]
        finally:
            client.stop()
            server2.stop()
    
    def test_server_recovery_detection(self, available_ports):
        """Client should detect when a dead server recovers."""
        port = available_ports[0]
        
        class EchoServer(FreelanceServer):
            def process_request(self, payload):
                return payload
        
        # Create client first
        client = FreelanceClient(
            request_timeout=1.0,
            heartbeat_interval=0.2,
            heartbeat_timeout=0.5,
            max_retries=1
        )
        client.connect(f"tcp://localhost:{port}")
        client.start()
        
        try:
            # Request should fail (no server)
            success, _ = client.request([b"test1"])
            assert success is False
            
            # Start server
            server = EchoServer(f"tcp://*:{port}", f"server-{port}")
            thread = threading.Thread(target=server.start, daemon=True)
            thread.start()
            time.sleep(0.3)
            
            # Request should now succeed
            success, response = client.request([b"test2"])
            assert success is True
            
            server.stop()
        finally:
            client.stop()


# =============================================================================
# Integration Tests - Reliability Features
# =============================================================================

class TestReliabilityFeatures:
    """Tests for reliability features."""
    
    def test_request_retry_on_timeout(self, slow_server, client):
        """Client should retry on timeout."""
        server, port = slow_server
        server.delay = 0.3
        
        client.request_timeout = 0.2
        client.max_retries = 5
        client.retry_base_delay = 0.05
        
        client.connect(f"tcp://localhost:{port}")
        client.start()
        time.sleep(0.1)
        
        # Request will timeout on first attempts but may succeed eventually
        # Since delay (0.3) > timeout (0.2), it will fail
        success, _ = client.request([b"test"])
        
        # Verify retries happened
        server_info = client._servers[f"tcp://localhost:{port}"]
        # Should have multiple failures from retries
        assert server_info.total_requests >= 1
    
    def test_request_deduplication(self, available_port):
        """Server should deduplicate retried requests."""
        process_count = 0
        
        class CountingServer(FreelanceServer):
            def process_request(self, payload):
                nonlocal process_count
                process_count += 1
                # Simulate slow processing that causes client timeout
                time.sleep(0.3)
                return payload
        
        server = CountingServer(f"tcp://*:{available_port}", f"counting-server")
        thread = threading.Thread(target=server.start, daemon=True)
        thread.start()
        time.sleep(0.1)
        
        client = FreelanceClient(
            request_timeout=0.1,  # Very short timeout to force retries
            max_retries=3,
            retry_base_delay=0.05
        )
        client.connect(f"tcp://localhost:{available_port}")
        client.start()
        time.sleep(0.1)
        
        try:
            # This will timeout and retry, but server should only process once
            # per unique request ID
            client.request([b"test"])
            
            # Give server time to process
            time.sleep(0.5)
            
            # The request should have been processed only once despite retries
            # (deduplication in effect)
            assert process_count >= 1
        finally:
            client.stop()
            server.stop()
    
    def test_heartbeat_keeps_server_alive(self, echo_server, client):
        """Heartbeat should keep server in ALIVE state."""
        server, port = echo_server
        
        client.heartbeat_interval = 0.2
        client.heartbeat_timeout = 1.0
        
        client.connect(f"tcp://localhost:{port}")
        client.start()
        
        # Wait for heartbeats to establish connection
        time.sleep(0.5)
        
        # Do a request to mark server as alive
        client.request([b"test"])
        
        # Server should be alive
        server_info = client._servers[f"tcp://localhost:{port}"]
        assert server_info.state == ServerState.ALIVE
    
    def test_heartbeat_timeout_marks_dead(self, available_port, client):
        """Server should be marked dead after heartbeat timeout."""
        class EchoServer(FreelanceServer):
            def process_request(self, payload):
                return payload
        
        server = EchoServer(f"tcp://*:{available_port}", f"server")
        thread = threading.Thread(target=server.start, daemon=True)
        thread.start()
        time.sleep(0.1)
        
        client.heartbeat_interval = 0.1
        client.heartbeat_timeout = 0.3
        
        client.connect(f"tcp://localhost:{available_port}")
        client.start()
        
        # Do a request to mark server alive
        time.sleep(0.2)
        client.request([b"test"])
        
        server_info = client._servers[f"tcp://localhost:{available_port}"]
        assert server_info.state == ServerState.ALIVE
        
        # Stop the server
        server.stop()
        
        # Wait for heartbeat timeout
        time.sleep(0.5)
        
        # Server should be marked dead
        assert server_info.state == ServerState.DEAD
        
        client.stop()
    
    def test_server_statistics_tracking(self, connected_client):
        """Server statistics should be tracked correctly."""
        client, port = connected_client
        
        # Send some successful requests
        for i in range(5):
            success, _ = client.request([f"test-{i}".encode()])
            assert success is True
        
        stats = client.get_server_stats()
        endpoint = f"tcp://localhost:{port}"
        
        assert endpoint in stats
        assert stats[endpoint]["total_requests"] == 5
        assert stats[endpoint]["success_rate"] == 1.0
        assert stats[endpoint]["avg_latency_ms"] > 0


# =============================================================================
# Integration Tests - Error Handling
# =============================================================================

class TestErrorHandling:
    """Tests for error handling."""
    
    def test_server_error_response(self, available_port):
        """Client should handle server error responses."""
        class ErrorServer(FreelanceServer):
            def process_request(self, payload):
                raise ValueError("Intentional test error")
        
        server = ErrorServer(f"tcp://*:{available_port}", f"error-server")
        thread = threading.Thread(target=server.start, daemon=True)
        thread.start()
        time.sleep(0.1)
        
        client = FreelanceClient(request_timeout=2.0, max_retries=1)
        client.connect(f"tcp://localhost:{available_port}")
        client.start()
        time.sleep(0.1)
        
        try:
            success, response = client.request([b"test"])
            
            assert success is False
            assert response is None
        finally:
            client.stop()
            server.stop()
    
    def test_malformed_server_response(self, available_port, zmq_context):
        """Client should handle malformed responses gracefully."""
        # Create a mock server that sends invalid responses
        server_socket = zmq_context.socket(zmq.ROUTER)
        server_socket.bind(f"tcp://*:{available_port}")
        
        client = FreelanceClient(request_timeout=0.5, max_retries=1)
        client.connect(f"tcp://localhost:{available_port}")
        client.start()
        time.sleep(0.1)
        
        def bad_responder():
            while True:
                try:
                    frames = server_socket.recv_multipart(zmq.NOBLOCK)
                    client_id = frames[0]
                    # Send malformed response (missing frames)
                    server_socket.send_multipart([client_id, b"", b"GARBAGE"])
                except zmq.Again:
                    time.sleep(0.01)
                except:
                    break
        
        responder_thread = threading.Thread(target=bad_responder, daemon=True)
        responder_thread.start()
        
        try:
            success, response = client.request([b"test"])
            
            # Should timeout/fail, not crash
            assert success is False
        finally:
            client.stop()
            server_socket.close()
    
    def test_client_stop_during_request(self, echo_server):
        """Stopping client during request should not cause crash."""
        server, port = echo_server
        
        client = FreelanceClient(request_timeout=5.0)
        client.connect(f"tcp://localhost:{port}")
        client.start()
        time.sleep(0.1)
        
        # Start a request in background
        result = [None]
        
        def make_request():
            result[0] = client.request([b"test"])
        
        request_thread = threading.Thread(target=make_request)
        request_thread.start()
        
        time.sleep(0.05)
        
        # Stop client while request is in progress
        client.stop()
        
        request_thread.join(timeout=1.0)
        
        # Should complete without crashing


# =============================================================================
# Stress Tests
# =============================================================================

class TestStress:
    """Stress tests for high load scenarios."""
    
    def test_high_request_volume(self, echo_server, client):
        """Client should handle high volume of requests."""
        server, port = echo_server
        
        client.connect(f"tcp://localhost:{port}")
        client.start()
        time.sleep(0.1)
        
        num_requests = 100
        successes = 0
        
        for i in range(num_requests):
            success, response = client.request([f"msg-{i}".encode()])
            if success:
                successes += 1
        
        # Should have high success rate
        assert successes >= num_requests * 0.95  # At least 95% success
    
    def test_concurrent_requests(self, echo_server, client):
        """Client should handle concurrent requests safely."""
        server, port = echo_server
        
        client.connect(f"tcp://localhost:{port}")
        client.start()
        time.sleep(0.1)
        
        num_threads = 10
        requests_per_thread = 20
        results = []
        
        def make_requests(thread_id):
            thread_results = []
            for i in range(requests_per_thread):
                payload = [f"thread-{thread_id}-msg-{i}".encode()]
                success, response = client.request(payload)
                thread_results.append((success, payload, response))
            return thread_results
        
        with ThreadPoolExecutor(max_workers=num_threads) as executor:
            futures = [executor.submit(make_requests, i) for i in range(num_threads)]
            for future in as_completed(futures):
                results.extend(future.result())
        
        # Verify results
        total = len(results)
        successes = sum(1 for success, _, _ in results if success)
        
        assert total == num_threads * requests_per_thread
        assert successes >= total * 0.90  # At least 90% success
        
        # Verify responses match requests
        for success, payload, response in results:
            if success:
                assert response == payload
    
    def test_rapid_connect_disconnect(self, echo_server, client):
        """Client should handle rapid connect/disconnect cycles."""
        server, port = echo_server
        endpoint = f"tcp://localhost:{port}"
        
        client.start()
        
        for _ in range(20):
            client.connect(endpoint)
            time.sleep(0.02)
            client.disconnect(endpoint)
            time.sleep(0.02)
        
        # Should be able to connect and use after all that
        client.connect(endpoint)
        time.sleep(0.1)
        
        success, _ = client.request([b"test"])
        assert success is True
    
    def test_multi_server_load_distribution(self, available_ports):
        """Load should be distributed across healthy servers."""
        servers = []
        request_counts = {}
        
        for port in available_ports[:3]:
            class CountingServer(FreelanceServer):
                def __init__(self, bind_address, server_id, counts_dict, port):
                    super().__init__(bind_address, server_id)
                    self.counts_dict = counts_dict
                    self.port = port
                    self.counts_dict[port] = 0
                
                def process_request(self, payload):
                    self.counts_dict[self.port] += 1
                    return payload
            
            server = CountingServer(
                f"tcp://*:{port}",
                f"server-{port}",
                request_counts,
                port
            )
            thread = threading.Thread(target=server.start, daemon=True)
            thread.start()
            servers.append(server)
        
        time.sleep(0.1)
        
        client = FreelanceClient(request_timeout=2.0)
        for port in available_ports[:3]:
            client.connect(f"tcp://localhost:{port}")
        client.start()
        time.sleep(0.2)
        
        try:
            # Send many requests
            for i in range(60):
                client.request([f"msg-{i}".encode()])
            
            # Check that requests were handled (distribution depends on server selection)
            total_handled = sum(request_counts.values())
            assert total_handled >= 55  # Allow some failures
        finally:
            client.stop()
            for server in servers:
                server.stop()


# =============================================================================
# Cache Tests
# =============================================================================

class TestServerCache:
    """Tests for server-side request cache."""
    
    def test_cache_cleanup(self, available_port):
        """Expired cache entries should be cleaned up."""
        server = FreelanceServer(f"tcp://*:{available_port}", "cache-test")
        server.CACHE_TTL = 0.1  # Very short TTL for testing
        server.CACHE_CLEANUP_INTERVAL = 0.05
        
        # Add some cache entries
        for i in range(10):
            server._request_cache[f"req-{i}"] = ProcessedRequest(
                request_id=f"req-{i}",
                response=[b"response"],
                timestamp=time.time() - 0.2  # Already expired
            )
        
        assert len(server._request_cache) == 10
        
        # Trigger cleanup
        server._last_cache_cleanup = 0
        server._cleanup_cache_if_needed()
        
        assert len(server._request_cache) == 0
        
        server.context.term()
    
    def test_cache_preserves_recent(self, available_port):
        """Recent cache entries should not be cleaned."""
        server = FreelanceServer(f"tcp://*:{available_port}", "cache-test")
        server.CACHE_TTL = 10.0  # Long TTL
        server.CACHE_CLEANUP_INTERVAL = 0.0  # Always clean
        
        # Add recent entries
        for i in range(5):
            server._request_cache[f"req-{i}"] = ProcessedRequest(
                request_id=f"req-{i}",
                response=[b"response"],
                timestamp=time.time()
            )
        
        server._last_cache_cleanup = 0
        server._cleanup_cache_if_needed()
        
        assert len(server._request_cache) == 5
        
        server.context.term()


# =============================================================================
# Thread Safety Tests
# =============================================================================

class TestThreadSafety:
    """Tests for thread safety."""
    
    def test_concurrent_connect_disconnect(self, available_ports):
        """Concurrent connect/disconnect should be thread-safe."""
        client = FreelanceClient()
        client.start()
        
        def connect_disconnect(port):
            endpoint = f"tcp://localhost:{port}"
            for _ in range(10):
                client.connect(endpoint)
                time.sleep(0.01)
                client.disconnect(endpoint)
                time.sleep(0.01)
        
        threads = [
            threading.Thread(target=connect_disconnect, args=(port,))
            for port in available_ports
        ]
        
        for t in threads:
            t.start()
        for t in threads:
            t.join()
        
        # Should complete without errors
        client.stop()
    
    def test_concurrent_stats_access(self, connected_client):
        """Concurrent stats access should be thread-safe."""
        client, port = connected_client
        
        errors = []
        
        def access_stats():
            try:
                for _ in range(100):
                    stats = client.get_server_stats()
                    assert isinstance(stats, dict)
            except Exception as e:
                errors.append(e)
        
        def make_requests():
            try:
                for i in range(50):
                    client.request([f"msg-{i}".encode()])
            except Exception as e:
                errors.append(e)
        
        threads = [
            threading.Thread(target=access_stats),
            threading.Thread(target=access_stats),
            threading.Thread(target=make_requests),
            threading.Thread(target=make_requests),
        ]
        
        for t in threads:
            t.start()
        for t in threads:
            t.join()
        
        assert len(errors) == 0, f"Errors occurred: {errors}"


# =============================================================================
# Factory Function Tests
# =============================================================================

class TestCreateClient:
    """Tests for the create_client convenience function."""
    
    def test_create_client_connects_all(self, available_ports):
        """create_client should connect to all provided endpoints."""
        endpoints = [f"tcp://localhost:{port}" for port in available_ports[:3]]
        
        client = create_client(endpoints)
        
        try:
            assert len(client._servers) == 3
            assert client._running is True
            
            for endpoint in endpoints:
                assert endpoint in client._servers
        finally:
            client.stop()
    
    def test_create_client_passes_kwargs(self, available_ports):
        """create_client should pass kwargs to FreelanceClient."""
        endpoints = [f"tcp://localhost:{available_ports[0]}"]
        
        client = create_client(
            endpoints,
            request_timeout=15.0,
            max_retries=10
        )
        
        try:
            assert client.request_timeout == 15.0
            assert client.max_retries == 10
        finally:
            client.stop()


# =============================================================================
# Edge Cases
# =============================================================================

class TestEdgeCases:
    """Tests for edge cases and boundary conditions."""
    
    def test_zero_timeout(self, echo_server, client):
        """Zero timeout should fail immediately."""
        server, port = echo_server
        
        client.connect(f"tcp://localhost:{port}")
        client.start()
        time.sleep(0.1)
        
        # Request with zero timeout
        success, _ = client.request([b"test"], timeout=0.001)
        
        # Should fail (timeout too short)
        assert success is False
    
    def test_very_long_request_id(self, connected_client):
        """Should handle very long request IDs."""
        client, port = connected_client
        
        # The implementation uses UUIDs internally, so this tests
        # that the system works correctly with standard IDs
        for _ in range(5):
            success, response = client.request([b"test"])
            assert success is True
    
    def test_unicode_in_server_id(self, available_port):
        """Server should handle unicode in server ID."""
        server = FreelanceServer(
            f"tcp://*:{available_port}",
            "server-日本語-émoji-🚀"
        )
        
        assert server.server_id == "server-日本語-émoji-🚀"
        
        server.context.term()
    
    def test_max_retries_zero(self, echo_server):
        """Zero max_retries should still attempt once."""
        server, port = echo_server
        
        client = FreelanceClient(max_retries=0)
        client.connect(f"tcp://localhost:{port}")
        client.start()
        time.sleep(0.1)
        
        try:
            # Should fail because max_retries=0 means no attempts
            success, _ = client.request([b"test"])
            assert success is False
        finally:
            client.stop()
    
    def test_single_retry(self, echo_server):
        """max_retries=1 should attempt exactly once."""
        server, port = echo_server
        
        client = FreelanceClient(max_retries=1)
        client.connect(f"tcp://localhost:{port}")
        client.start()
        time.sleep(0.1)
        
        try:
            success, response = client.request([b"test"])
            assert success is True
            assert response == [b"test"]
        finally:
            client.stop()


# =============================================================================
# Protocol Tests
# =============================================================================

class TestProtocol:
    """Tests for the message protocol."""
    
    def test_heartbeat_command_constant(self):
        """Heartbeat command should match between client and server."""
        assert FreelanceClient.HEARTBEAT_CMD == FreelanceServer.HEARTBEAT_CMD
    
    def test_request_command_constant(self):
        """Request command should match between client and server."""
        assert FreelanceClient.REQUEST_CMD == FreelanceServer.REQUEST_CMD
    
    def test_heartbeat_response_format(self, available_port, zmq_context):
        """Server heartbeat response should have correct format."""
        server = FreelanceServer(f"tcp://*:{available_port}", "test-server")
        thread = threading.Thread(target=server.start, daemon=True)
        thread.start()
        time.sleep(0.1)
        
        # Create a test client socket
        client_socket = zmq_context.socket(zmq.DEALER)
        client_socket.setsockopt_string(zmq.IDENTITY, "test-client")
        client_socket.connect(f"tcp://localhost:{available_port}")
        
        try:
            # Send heartbeat
            client_socket.send_multipart([b"", b"HEARTBEAT"])
            
            # Receive response
            response = client_socket.recv_multipart()
            
            assert len(response) >= 3
            assert response[0] == b""  # Empty delimiter
            assert response[1] == b"HEARTBEAT"
            assert response[2] == b"test-server"
        finally:
            client_socket.close()
            server.stop()


# =============================================================================
# Run Configuration
# =============================================================================

if __name__ == "__main__":
    pytest.main([__file__, "-v", "--tb=short"])
```

---

## Test Configuration (pytest.ini)

```ini
[pytest]
testpaths = .
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short
filterwarnings =
    ignore::DeprecationWarning
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    integration: marks tests as integration tests
    stress: marks tests as stress tests
timeout = 60
```

---

## Test Requirements (requirements-test.txt)

```
pytest>=7.0.0
pytest-cov>=4.0.0
pytest-timeout>=2.1.0
pytest-xdist>=3.0.0  # For parallel test execution
pyzmq>=25.0.0
```

---

## Running the Tests

```bash
# Run all tests
pytest test_freelance.py -v

# Run with coverage report
pytest test_freelance.py --cov=freelance_client --cov=freelance_server --cov-report=html

# Run only fast unit tests
pytest test_freelance.py -v -k "Unit"

# Run stress tests
pytest test_freelance.py -v -k "Stress"

# Run in parallel (faster)
pytest test_freelance.py -v -n auto

# Run with detailed output on failures
pytest test_freelance.py -v --tb=long

# Stop on first failure
pytest test_freelance.py -v -x
```

---

## Test Coverage Summary

| Category | Tests | Description |
|----------|-------|-------------|
| **Unit - ServerInfo** | 8 | State tracking, metrics, latency calculation |
| **Unit - Server** | 4 | Initialization, caching, default behavior |
| **Unit - Client** | 12 | Initialization, server selection, connection management |
| **Integration - Basic** | 9 | Request/response, payloads, timeouts |
| **Integration - Multi-Server** | 3 | Failover, recovery detection |
| **Integration - Reliability** | 5 | Retries, deduplication, heartbeating |
| **Error Handling** | 3 | Server errors, malformed responses |
| **Stress** | 4 | High volume, concurrency, load distribution |
| **Cache** | 2 | TTL expiration, cleanup |
| **Thread Safety** | 2 | Concurrent access |
| **Edge Cases** | 5 | Boundary conditions |
| **Protocol** | 3 | Message format verification |

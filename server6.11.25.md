#!/bin/bash

# Port Scanner Script
# A simple but effective port scanner written in shell
# Only use on systems you own or have permission to scan

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Default values
timeout_duration=1
start_port=1
end_port=1024
verbose=0

# Function to display usage
usage() {
    echo "Usage: $0 -h <host> [-p <start_port>-<end_port>] [-t <timeout>] [-v]"
    echo ""
    echo "Options:"
    echo "  -h <host>       Target hostname or IP address (required)"
    echo "  -p <range>      Port range (default: 1-1024)"
    echo "                  Examples: -p 80, -p 1-100, -p 8000-9000"
    echo "  -t <seconds>    Connection timeout in seconds (default: 1)"
    echo "  -v              Verbose mode (show closed ports)"
    echo "  -H              Display this help message"
    echo ""
    echo "Examples:"
    echo "  $0 -h 192.168.1.1"
    echo "  $0 -h example.com -p 80-443"
    echo "  $0 -h 10.0.0.1 -p 1-65535 -t 2 -v"
    exit 1
}

# Function to check if a port is open
scan_port() {
    local host=$1
    local port=$2
    
    # Try to connect using /dev/tcp (bash built-in)
    if command -v bash >/dev/null 2>&1; then
        timeout $timeout_duration bash -c "echo >/dev/tcp/$host/$port" 2>/dev/null
        if [ $? -eq 0 ]; then
            return 0
        else
            return 1
        fi
    # Fallback to nc (netcat) if available
    elif command -v nc >/dev/null 2>&1; then
        nc -z -w$timeout_duration $host $port 2>/dev/null
        return $?
    else
        echo -e "${RED}Error: Neither bash tcp support nor netcat (nc) is available${NC}"
        exit 1
    fi
}

# Function to get service name from port
get_service() {
    local port=$1
    case $port in
        21) echo "FTP" ;;
        22) echo "SSH" ;;
        23) echo "Telnet" ;;
        25) echo "SMTP" ;;
        53) echo "DNS" ;;
        80) echo "HTTP" ;;
        110) echo "POP3" ;;
        143) echo "IMAP" ;;
        443) echo "HTTPS" ;;
        445) echo "SMB" ;;
        3306) echo "MySQL" ;;
        3389) echo "RDP" ;;
        5432) echo "PostgreSQL" ;;
        5900) echo "VNC" ;;
        8080) echo "HTTP-Proxy" ;;
        8443) echo "HTTPS-Alt" ;;
        *) echo "" ;;
    esac
}

# Parse command line arguments
while getopts "h:p:t:vH" opt; do
    case $opt in
        h)
            target_host="$OPTARG"
            ;;
        p)
            if [[ "$OPTARG" =~ ^[0-9]+$ ]]; then
                # Single port
                start_port=$OPTARG
                end_port=$OPTARG
            elif [[ "$OPTARG" =~ ^([0-9]+)-([0-9]+)$ ]]; then
                # Port range
                start_port=${BASH_REMATCH[1]}
                end_port=${BASH_REMATCH[2]}
            else
                echo -e "${RED}Invalid port range format${NC}"
                usage
            fi
            ;;
        t)
            timeout_duration="$OPTARG"
            ;;
        v)
            verbose=1
            ;;
        H)
            usage
            ;;
        \?)
            echo -e "${RED}Invalid option: -$OPTARG${NC}"
            usage
            ;;
    esac
done

# Check if host is provided
if [ -z "$target_host" ]; then
    echo -e "${RED}Error: Host is required${NC}"
    usage
fi

# Validate port range
if [ $start_port -lt 1 ] || [ $end_port -gt 65535 ] || [ $start_port -gt $end_port ]; then
    echo -e "${RED}Error: Invalid port range. Ports must be between 1-65535${NC}"
    exit 1
fi

# Check if host is reachable
echo -e "${YELLOW}Checking if $target_host is reachable...${NC}"
if command -v ping >/dev/null 2>&1; then
    ping -c 1 -W 2 $target_host >/dev/null 2>&1
    if [ $? -ne 0 ]; then
        echo -e "${RED}Warning: Host may be unreachable or blocking ICMP${NC}"
    fi
fi

# Display scan information
echo -e "${GREEN}Starting port scan...${NC}"
echo "Target: $target_host"
echo "Port range: $start_port-$end_port"
echo "Timeout: ${timeout_duration}s"
echo ""

# Count open ports
open_count=0
closed_count=0

# Start scanning
echo -e "${YELLOW}Scanning ports...${NC}"
echo ""

# Main scanning loop
for port in $(seq $start_port $end_port); do
    # Show progress for large scans
    if [ $((port % 100)) -eq 0 ] && [ $port -ne $start_port ]; then
        echo -e "${YELLOW}Progress: Scanned up to port $port${NC}"
    fi
    
    scan_port $target_host $port
    result=$?
    
    if [ $result -eq 0 ]; then
        service=$(get_service $port)
        if [ -n "$service" ]; then
            echo -e "${GREEN}Port $port: OPEN ($service)${NC}"
        else
            echo -e "${GREEN}Port $port: OPEN${NC}"
        fi
        ((open_count++))
    else
        if [ $verbose -eq 1 ]; then
            echo -e "${RED}Port $port: CLOSED${NC}"
        fi
        ((closed_count++))
    fi
done

# Display summary
echo ""
echo -e "${YELLOW}Scan complete!${NC}"
echo "Open ports: $open_count"
echo "Closed/filtered ports: $closed_count"
echo "Total ports scanned: $((open_count + closed_count))"

# Security reminder
echo ""
echo -e "${YELLOW}Remember: Only scan systems you own or have permission to test!${NC}"

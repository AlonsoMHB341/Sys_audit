#!/bin/bash
# Sentinel-Audit - A lightweight Linux Security & Hardening Auditor
# Designed for IT & Cybersecurity system assessments.

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo -e "${YELLOW}==============================================${NC}"
echo -e "${YELLOW}      Sentinel-Audit: Linux Security Tool     ${NC}"
echo -e "${YELLOW}==============================================${NC}\n"

# 1. Root Check
if [ "$EUID" -ne 0 ]; then
  echo -e "${RED}[!] Please run as root (sudo) for a full audit.${NC}"
  exit 1
fi

echo -e "${GREEN}[*] Privilege Check: ROOT access confirmed.${NC}\n"

# 2. SSH Hardening Audit
echo -e "${YELLOW}--- [ SSH Hardening Audit ] ---${NC}"
SSHD_CONFIG="/etc/ssh/sshd_config"
if [ -f "$SSHD_CONFIG" ]; then
    # Check PermitRootLogin
    if grep -q "^PermitRootLogin no" "$SSHD_CONFIG"; then
        echo -e "${GREEN}[✓] Root SSH login is disabled.${NC}"
    else
        echo -e "${RED}[✗] VULNERABILITY: Root SSH login might be enabled. (Check PermitRootLogin)${NC}"
    fi
    
    # Check PasswordAuthentication
    if grep -q "^PasswordAuthentication no" "$SSHD_CONFIG"; then
        echo -e "${GREEN}[✓] SSH Password Auth is disabled (Key-based only).${NC}"
    else
        echo -e "${RED}[✗] VULNERABILITY: SSH Password Auth is active. Susceptible to brute force.${NC}"
    fi
else
    echo "SSH config not found at $SSHD_CONFIG."
fi
echo ""

# 3. Open Ports & Listening Services
echo -e "${YELLOW}--- [ Network Attack Surface ] ---${NC}"
echo "Listening TCP/UDP ports:"
ss -tulpn | grep LISTEN | awk '{print $5, $7}' | column -t
echo ""

# 4. Privilege Escalation Vectors (SUID Binaries)
echo -e "${YELLOW}--- [ SUID/SGID Privilege Escalation Vectors ] ---${NC}"
echo "Scanning for top-level SUID binaries (this may take a few seconds)..."
find /usr/bin -perm -4000 -exec ls -ldb {} \; 2>/dev/null | head -n 5
echo -e "${YELLOW}(Showing top 5 results. Review these carefully for exploits!)${NC}\n"

# 5. Basic Firewall Status
echo -e "${YELLOW}--- [ Firewall Status ] ---${NC}"
if command -v ufw >/dev/null 2>&1; then
    ufw status | head -n 1
elif command -v firewalld >/dev/null 2>&1; then
    systemctl status firewalld | grep Active
else
    echo -e "${RED}[!] Neither UFW nor Firewalld detected.${NC}"
fi
echo ""

echo -e "${GREEN}==============================================${NC}"
echo -e "${GREEN}             Audit Complete                   ${NC}"
echo -e "${GREEN}==============================================${NC}"

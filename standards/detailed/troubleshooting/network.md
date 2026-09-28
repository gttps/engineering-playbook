# Network Troubleshooting

Quick checks for local addressing, DNS resolution, and connectivity.

```bash
# Show local network configuration
# Windows
ipconfig
# macOS
ifconfig
# Linux
ip addr

# Test reachability (ICMP may be blocked even when the service is healthy)
ping -c 4 8.8.8.8       # macOS/Linux
ping -n 4 8.8.8.8       # Windows

# Trace the route to a destination
traceroute example.com  # macOS/Linux
tracert example.com     # Windows

# Resolve a hostname
dig example.com
nslookup example.com

# Test whether a TCP port is reachable
nc -vz example.com 443
```

An address in `169.254.0.0/16` is IPv4 link-local; on a network that uses DHCP, it often means the device did not receive a lease. Confirm the network's addressing setup before treating it as a fault. A successful ping confirms an ICMP response, not that a specific application port is reachable; a timeout may also mean ICMP is filtered.

References: [Cisco networking discussion](https://learningnetwork.cisco.com/s/question/0D5Kd0000BSFxp2KQD/how-do-you-troubleshoot-a-network-and-what-are-the-best-equipement-in-networking), [SolarWinds network troubleshooting](https://www.solarwinds.com/resources/it-glossary/network-troubleshooting).

## curl

Replace placeholders with environment variables or local test values. Avoid committing real credentials.

```bash
# Inspect an HTTPS request and response
curl -v https://example.com

# Add one or more headers
curl -H 'Accept: application/json' \
  -H 'X-Correlation-ID: request-123' https://api.example.com

# Bearer token
curl -H "Authorization: Bearer $TOKEN" https://api.example.com

# Basic authentication (curl prompts for the password)
curl --user "$API_USER" https://api.example.com

# API key in a header
curl -H "X-API-Key: $API_KEY" https://api.example.com

# API key in the query string (may be recorded in logs and history)
curl --get --data-urlencode "api_key=$API_KEY" https://api.example.com

# Cookie
curl -b 'session=SESSION_VALUE' https://api.example.com

# JSON request body
curl -X POST https://api.example.com/items \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $TOKEN" \
  --data '{"name":"example"}'

# Form fields
curl -X POST https://api.example.com/login \
  --data-urlencode "username=$API_USER" \
  --data-urlencode "password=$API_PASSWORD"

# Mutual TLS (client certificate and private key)
curl --cert client.crt --key client.key https://api.example.com

# Skip certificate verification for diagnosis only; do not use in production
curl --insecure https://example.com
```

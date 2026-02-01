# FortiGate Captive Portal Configuration Guide
## Freetown International Airport WiFi System

---

## Overview
This guide configures FortiGate to use your external captive portal with RADIUS authentication.

---

## Step 1: Configure RADIUS Server

### Via CLI:
```bash
config user radius
    edit "FNA_RADIUS"
        set server "YOUR_RADIUS_SERVER_IP"
        set secret "YOUR_RADIUS_SECRET"
        set radius-port 1812
        set auth-type auto
        set timeout 5
        set all-usergroup disable
    next
end
```

### Via GUI:
1. Go to **User & Authentication > RADIUS Servers**
2. Click **Create New**
3. Fill in:
   - Name: `FNA_RADIUS`
   - Primary Server IP: `YOUR_RADIUS_SERVER_IP`
   - Secret: `YOUR_RADIUS_SECRET`
   - Authentication Port: `1812`
   - Test Connectivity

---

## Step 2: Create User Group

### Via CLI:
```bash
config user group
    edit "FNA_Airport_Users"
        set member "FNA_RADIUS"
    next
end
```

### Via GUI:
1. Go to **User & Authentication > User Groups**
2. Click **Create New**
3. Name: `FNA_Airport_Users`
4. Type: `Firewall`
5. Add Remote Groups: Select `FNA_RADIUS`

---

## Step 3: Configure External Captive Portal

### Via CLI:
```bash
config system settings
    set gui-explicit-proxy enable
    set gui-captive-portal enable
end

config authentication setting
    set captive-portal-type external
    set captive-portal "https://YOUR_HOSTINGER_DOMAIN/index.html"
    set captive-portal-port 443
    set captive-portal-ssl-protocol tlsv1.2 tlsv1.3
end
```

### Via GUI:
1. Go to **System > Feature Visibility**
2. Enable **Explicit Proxy** and **Captive Portal**
3. Go to **Security Profiles > Captive Portal**
4. Configure:
   - Portal Type: `External`
   - Portal URL: `https://YOUR_HOSTINGER_DOMAIN/index.html`
   - Port: `443`

---

## Step 4: Create Firewall Policy for Authentication

### Via CLI:
```bash
config firewall policy
    edit 100
        set name "FNA_WiFi_Captive_Portal"
        set srcintf "wifi-interface"
        set dstintf "wan1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
        set identity-based-policy enable
        set auth-path captive-portal
        set groups "FNA_Airport_Users"
    next
end
```

### Via GUI:
1. Go to **Policy & Objects > Firewall Policy**
2. Click **Create New**
3. Configure:
   - Name: `FNA_WiFi_Captive_Portal`
   - Incoming Interface: `wifi-interface` (your WiFi interface)
   - Outgoing Interface: `wan1` (your internet interface)
   - Source: `all`
   - Destination: `all`
   - Service: `ALL`
   - Action: `ACCEPT`
   - NAT: `Enable`
   - **Security Profiles**: Enable **Authentication**
   - Authentication Portal: `Enable`
   - Identity-Based Policy: `Enable`
   - Groups: Select `FNA_Airport_Users`

---

## Step 5: Whitelist Captive Portal Domain

Users need to access your portal before authentication:

### Via CLI:
```bash
config firewall address
    edit "Captive_Portal_Server"
        set type fqdn
        set fqdn "YOUR_HOSTINGER_DOMAIN"
    next
end

config firewall policy
    edit 99
        set name "Allow_Captive_Portal_Access"
        set srcintf "wifi-interface"
        set dstintf "wan1"
        set srcaddr "all"
        set dstaddr "Captive_Portal_Server"
        set action accept
        set schedule "always"
        set service "HTTPS"
        set nat enable
    next
end
```

**Important:** This policy must be ABOVE the authentication policy (lower policy ID).

---

## Step 6: Configure Authentication Portal Mapping

This tells FortiGate how to handle the login form submission:

### Via CLI:
```bash
config authentication scheme
    edit "FNA_Portal_Scheme"
        set method basic
        set user-database "FNA_RADIUS"
    next
end

config authentication rule
    edit "FNA_Portal_Rule"
        set srcintf "wifi-interface"
        set srcaddr "all"
        set active-auth-scheme "FNA_Portal_Scheme"
    next
end
```

---

## Step 7: Configure Success/Failure Redirects

### Via CLI:
```bash
config authentication setting
    set auth-https enable
    set captive-portal-ip "0.0.0.0"
    set captive-portal-ip6 "::"
end

config system global
    set auth-http-port 1000
    set auth-https-port 1003
end
```

---

## Step 8: Test RADIUS Connectivity

### Via CLI:
```bash
# Test RADIUS server connectivity
execute user-device test-connectivity YOUR_RADIUS_SERVER_IP 1812

# Test authentication with sample credentials
execute user-device test-authentication FNA_RADIUS P1234567 testpassword
```

---

## How Authentication Works

### 1. User Connects to WiFi
- Device gets IP via DHCP
- User opens browser
- FortiGate intercepts HTTP/HTTPS traffic

### 2. FortiGate Redirects to Portal
- User sees your captive portal
- FortiGate replaces variables:
  - `$MAGIC$` → Unique session token
  - `$REDIR$` → Original URL user tried to access

### 3. User Submits Form
- Form POSTs to `/login` with:
  - `username` (Passport ID)
  - `password` (Access Code)
  - `magic` (Session token)
  - `redir` (Redirect URL)

### 4. FortiGate Validates
- Receives POST to `/login`
- Extracts username and password
- Sends RADIUS Access-Request to RADIUS server
- RADIUS checks credentials against database

### 5. RADIUS Responds
- **Access-Accept** → User authenticated
- **Access-Reject** → Invalid credentials

### 6. FortiGate Redirects
- **Success**: Redirects to `$REDIR$?auth=success`
- **Failure**: Redirects to portal with `?error=1`

### 7. Your Portal Shows Notification
- Success: SweetAlert2 popup → "Welcome to FNA WiFi!"
- Failure: Error toast → "Invalid access code"

---

## Important URLs FortiGate Uses

| Endpoint | Purpose |
|----------|---------|
| `/login` | Receives authentication POST |
| `/logout` | User logout |
| `/keepalive` | Session keep-alive |
| `/challenge` | Challenge-response auth (if needed) |

**Note:** These are handled by FortiGate, not your HTML page.

---

## RADIUS Server Requirements

Your RADIUS server must:

1. **Accept authentication requests** on port 1812
2. **Have user database** with:
   - Username: Passport ID (e.g., P1234567)
   - Password: Access Code from ticket/voucher
3. **Respond with**:
   - `Access-Accept` for valid credentials
   - `Access-Reject` for invalid credentials

### Example RADIUS User Entry (FreeRADIUS):
```
# /etc/freeradius/users
P1234567    Cleartext-Password := "ABC123XYZ"
            Reply-Message = "Welcome to FNA WiFi"
```

---

## Debugging Commands

### Check Authentication Status:
```bash
diagnose debug application fnbamd -1
diagnose debug enable
```

### View Active Sessions:
```bash
diagnose firewall auth list
```

### Clear User Session:
```bash
diagnose firewall auth clear
```

### View Authentication Logs:
```bash
execute log filter category 0
execute log filter field subtype authentication
execute log display
```

### Test RADIUS:
```bash
execute user-device test-connectivity YOUR_RADIUS_SERVER_IP 1812
execute user-device test-authentication FNA_RADIUS testuser testpass
```

---

## Troubleshooting

### Issue: Portal doesn't load
**Solution:** Check whitelist policy (Step 5) is above auth policy

### Issue: Form submits but nothing happens
**Solution:** 
- Verify RADIUS server is reachable
- Check RADIUS secret matches
- Review authentication logs

### Issue: Always shows error
**Solution:**
- Test RADIUS connectivity
- Verify user exists in RADIUS database
- Check RADIUS server logs

### Issue: Success but no internet
**Solution:**
- Verify firewall policy allows authenticated users
- Check NAT is enabled
- Review session table: `diagnose firewall auth list`

---

## Security Best Practices

1. **Use HTTPS only** for captive portal
2. **Strong RADIUS secret** (minimum 16 characters)
3. **Session timeout** (e.g., 8 hours for airport)
4. **Rate limiting** to prevent brute force
5. **Log all authentication attempts**
6. **Regular RADIUS password rotation**

---

## Session Timeout Configuration

```bash
config authentication setting
    set captive-portal-session-timeout-interval 480
end
```

This sets 8-hour timeout (480 minutes) - adjust as needed.

---

## Support

For FortiGate support:
- FortiGate Documentation: https://docs.fortinet.com
- FortiGate Support: https://support.fortinet.com

For RADIUS support:
- FreeRADIUS: https://freeradius.org
- Microsoft NPS: https://docs.microsoft.com/en-us/windows-server/networking/technologies/nps/

---

## Quick Reference

**Your Portal URL:** `https://YOUR_HOSTINGER_DOMAIN/index.html`

**Form Action:** `/login` (handled by FortiGate)

**Required Form Fields:**
- `username` (text input)
- `password` (password input)
- `magic` (hidden, value: `$MAGIC$`)
- `redir` (hidden, value: `$REDIR$`)

**Success Redirect:** `$REDIR$?auth=success`

**Failure Redirect:** `portal_url?error=1`

---

**Configuration Complete!** 🎉

Your captive portal is now ready to authenticate users via FortiGate + RADIUS.

# Podman WSL2 on Windows - Access Instructions

## Current Status ✓
Your Local Deep Research application is running successfully!

## How to Access

### Option 1: Direct IP Access (Currently Working)
```
http://172.31.252.236:3000
```
- This is the WSL2 machine IP where your Podman container is running
- Works immediately, no additional setup needed

### Option 2: Using localhost (Recommended for convenience)

Since Podman on Windows/WSL2 doesn't automatically forward `localhost` to the WSL IP, you have a few options:

#### Method A: Edit Windows Hosts File (Permanent)
1. Open Notepad as Administrator
2. Open: `C:\Windows\System32\drivers\etc\hosts`
3. Add this line:
   ```
   172.31.252.236  localhost
   ```
4. Save the file
5. Access: `http://localhost:3000`

**Note:** This replaces the standard localhost, which may affect other local services.

#### Method B: Use a Different Hostname (Alternative)
1. Open Notepad as Administrator
2. Open: `C:\Windows\System32\drivers\etc\hosts`
3. Add this line:
   ```
   172.31.252.236  research.local
   ```
4. Access: `http://research.local:3000`

#### Method C: Create a Browser Bookmark or Alias
Simply bookmark `http://172.31.252.236:3000` for quick access.

## Why Is This Needed?

Podman on Windows uses WSL2 to run containers. The WSL2 machine gets its own IP address (172.31.252.236 in your case), and port forwarding works through that IP address rather than the Windows `localhost`.

The Docker Desktop integration on Windows uses more sophisticated networking that isn't available with Podman's default setup.

## Container Information

- **Container Name:** deep-research
- **Internal Port:** 5000
- **Mapped Port:** 3000
- **WSL IP:** 172.31.252.236
- **Image:** localdeepresearch/local-deep-research:latest

## Troubleshooting

If the IP address changes when you restart Podman, you can find the current IP with:
```powershell
wsl ip addr | Select-String "inet\s" | Select-Object -First 2
```

## Docker Desktop Alternative

If you have Docker Desktop installed instead of standalone Podman, you can access via `localhost:3000` directly without any additional setup.

# Security Policy

## Reporting Vulnerabilities

If you discover a security vulnerability in SoundCloud Desktop, please report it
responsibly by opening a private issue or contacting the maintainer directly.

Do **not** open a public issue for security vulnerabilities.

## Known Security Considerations

- The binary is **not code-signed**. Verify the SHA-256 hash after downloading.
- The app registers itself in `Software\Microsoft\Windows\CurrentVersion\Run` when
  autostart is enabled.
- The app downloads the WebView2 runtime from Microsoft if not already installed.
- Discord Rich Presence integration uses a local IPC pipe (`\\?\pipe\discord-ipc-*`).

## Build Security

Release builds should:
- Strip PDB debug paths (`/PDBALTPATH:%_PDB%`)
- Enable Control Flow Guard (`/guard:cf`)
- Be code-signed with a valid certificate

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

WetCat Launcher is a modded Minecraft launcher built with Electron. It allows users to join modded servers without manually installing Java, Forge, or mods. The launcher handles authentication (Microsoft OAuth 2.0 and Mojang Yggdrasil), distribution management, automatic updates, and Discord Rich Presence integration.

## Development Commands

### Running the Launcher
```bash
npm start
```

### Building Installers
```bash
# Build for current platform
npm run dist

# Platform-specific builds
npm run dist:win    # Windows x64 (NSIS installer)
npm run dist:mac    # macOS (DMG for x64 and arm64)
npm run dist:linux  # Linux x64 (AppImage)
```

### Code Quality
```bash
npm run lint        # Run ESLint to check code style
```

Note: macOS builds may not work on Windows/Linux and vice-versa due to platform-specific dependencies.

## Architecture Overview

### Electron Process Model

**Main Process** ([index.js](index.js))
- Initializes Electron app and creates BrowserWindow
- Handles IPC communication between main and renderer processes
- Manages auto-updates via electron-updater
- Controls Microsoft OAuth login windows
- Handles shell operations (trash, etc.)

**Renderer Process** ([app/app.ejs](app/app.ejs))
- UI rendering with EJS templates
- Core UI logic in `app/assets/js/scripts/` directory
- Communicates with main process via IPC

**Preloader** ([app/assets/js/preloader.js](app/assets/js/preloader.js))
- Bridges main and renderer processes
- Loads ConfigManager and fetches distribution index on startup
- Validates game configuration before UI initialization

### Manager Architecture

The launcher uses specialized manager modules for separation of concerns:

**ConfigManager** ([app/assets/js/configmanager.js](app/assets/js/configmanager.js))
- Stores user settings in `.wetcatlauncher/config.json`
- Manages game resolution, RAM allocation, data directory
- Handles account authentication database
- Platform-specific data directory handling:
  - Windows: `%APPDATA%/.wetcatlauncher`
  - macOS: `~/Library/Application Support/.wetcatlauncher`
  - Linux: `~/.wetcatlauncher`

**AuthManager** ([app/assets/js/authmanager.js](app/assets/js/authmanager.js))
- Abstracts authentication logic for Microsoft OAuth 2.0 and Mojang Yggdrasil
- Handles auth error scenarios
- Note: Credentials are NEVER stored locally, transmitted directly to Mojang/Microsoft

**DistroManager** ([app/assets/js/distromanager.js](app/assets/js/distromanager.js))
- Fetches and caches distribution index from remote server
- Uses helios-core's DistributionAPI
- Distribution URL configurable via config
- See [docs/distro.md](docs/distro.md) for distribution spec

**ProcessBuilder** ([app/assets/js/processbuilder.js](app/assets/js/processbuilder.js))
- Constructs Java process arguments for Minecraft game launch
- Handles mod loading for Forge, Fabric, and LiteLoader
- Manages native libraries and JVM configuration
- Supports Minecraft versions from legacy to 1.20.4+

### UI State Management

**View-Based Routing** ([app/assets/js/scripts/uibinder.js](app/assets/js/scripts/uibinder.js))
- Manages view transitions between:
  - `welcome` - First-time user setup
  - `loginOptions` - Login method selection
  - `login` - Account login
  - `landing` - Main launcher interface (server selection, play button)
  - `settings` - Settings and configuration
  - `waiting` - Loading states

**Core UI Logic** ([app/assets/js/scripts/uicore.js](app/assets/js/scripts/uicore.js))
- Handles auto-update checks
- User authentication flows
- Game launch sequence

### IPC Communication

**IPC Constants** ([app/assets/js/ipcconstants.js](app/assets/js/ipcconstants.js))
- Defines opcodes for main ↔ renderer communication
- Microsoft Auth opcodes (login/logout)
- Shell operations (trash items)
- Auto-update notifications

Key IPC handlers in main process:
- `autoUpdateAction` - Update management
- `distributionIndexDone` - Distribution loading completion
- `SHELL_OPCODE.TRASH_ITEM` - File operations
- Microsoft OAuth window management

### Templating System

**EJS Templates** - All views are `.ejs` files in `/app` directory
- [app.ejs](app/app.ejs) - Main container with embedded views
- Dynamic data injection with `lang()` function from LangLoader
- Random background image selection (8 backgrounds available)
- CSP (Content Security Policy) restricts inline scripts for security

### Data Flow

**Startup Sequence:**
1. Main process (index.js:360 lines) initializes Electron
2. Creates BrowserWindow with preload script
3. Loads app.ejs template with random background
4. Preloader runs: loads ConfigManager, fetches distribution index, validates config
5. UIBinder determines user state (first launch/logged in/logged out)
6. Shows appropriate view (welcome/login/landing)
7. UICore handles interactions and auto-update checks

**Game Launch Flow:**
1. User selects server and clicks "Play"
2. ProcessBuilder constructs JVM arguments
3. Validates/downloads required mods and assets (via helios-core)
4. Ensures correct Java version installed
5. Spawns game process with authentication token
6. Monitors game process for completion

## Key Technologies

- **Electron 33.2.1** - Desktop application framework
- **Node.js 20.x.x** - Required runtime version
- **helios-core ~2.2.4** - Core launcher library (handles Minecraft game mechanics)
- **helios-distribution-types ^1.3.0** - Distribution schema definitions
- **electron-updater ^6.3.9** - Auto-update functionality
- **discord-rpc-patch ^4.0.1** - Discord Rich Presence integration
- **EJS 3.1.10** - Template engine for dynamic HTML
- **jQuery 3.7.1** - DOM manipulation
- **got ^11.8.5** - HTTP client for API requests

## Code Style (ESLint)

The project enforces strict code style via ESLint ([.eslintrc.json](.eslintrc.json)):
- **No semicolons** (semi: "never")
- **Single quotes** for strings
- **4-space indentation**
- **Windows line endings** (linebreak-style: "windows")
- **No var declarations** (use const/let)
- ES2022 syntax support
- Special override: `app/assets/js/scripts/*.js` has relaxed no-unused-vars and no-undef rules

## Security Considerations

- **Context Isolation:** Disabled (allows direct module access in renderer)
- **Node Integration:** Enabled (gives renderer full Node.js access)
- **Hardware Acceleration:** Disabled for security
- **Access Token Masking:** Auth tokens are masked in logs
- **No Local Credential Storage:** Credentials transmitted directly to auth providers
- **CSP Headers:** Strict content security policy on main template
- **DevTools Warning:** User-friendly scam prevention warning in console

## Important Files

### Configuration
- [package.json](package.json) - Dependencies and scripts
- [electron-builder.yml](electron-builder.yml) - Build configuration for all platforms
- [.eslintrc.json](.eslintrc.json) - Code style rules

### Main Process
- [index.js](index.js) - Main Electron process entry point (360 lines)

### Renderer Process Core
- [app/app.ejs](app/app.ejs) - Main HTML template
- [app/assets/js/preloader.js](app/assets/js/preloader.js) - Initialization bridge
- [app/assets/js/scripts/uicore.js](app/assets/js/scripts/uicore.js) - Core UI logic
- [app/assets/js/scripts/uibinder.js](app/assets/js/scripts/uibinder.js) - View state management

### Managers
- [app/assets/js/configmanager.js](app/assets/js/configmanager.js) - Configuration persistence
- [app/assets/js/authmanager.js](app/assets/js/authmanager.js) - Authentication
- [app/assets/js/distromanager.js](app/assets/js/distromanager.js) - Distribution API
- [app/assets/js/processbuilder.js](app/assets/js/processbuilder.js) - Game launch

### Documentation
- [README.md](README.md) - Project overview and setup instructions
- [docs/distro.md](docs/distro.md) - Distribution index specification
- [docs/MicrosoftAuth.md](docs/MicrosoftAuth.md) - Microsoft authentication setup

## Distribution Management

The launcher fetches a distribution index (JSON) from a remote server that defines:
- Available servers and their configurations
- Minecraft version per server
- Required mods and their download URLs
- Server icons, descriptions, and metadata
- Discord Rich Presence configuration

Use [Nebula](https://github.com/raykoshima/Nebula) to automate distribution index generation. See [helios-distribution-types](https://github.com/raykoshima/helios-distribution-types) for the complete schema specification.

## Build System

**Electron Builder** ([electron-builder.yml](electron-builder.yml)):
- Multi-platform support: Windows (NSIS), macOS (DMG), Linux (AppImage)
- ASAR packaging enabled
- Maximum compression
- Extra resources: `/libraries` directory (native Java tools)
- Output: `/dist` directory

**CI/CD**: GitHub Actions workflow (`.github/workflows/build.yml`) builds on all platforms for every push.

## Debugging

### VS Code Debug Configurations

**Debug Main Process:**
```json
{
  "name": "Debug Main Process",
  "type": "node",
  "request": "launch",
  "cwd": "${workspaceFolder}",
  "program": "${workspaceFolder}/node_modules/electron/cli.js",
  "args": ["."],
  "outputCapture": "std"
}
```

**Debug Renderer Process:**
```json
{
  "name": "Debug Renderer Process",
  "type": "chrome",
  "request": "launch",
  "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron",
  "windows": {
    "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron.cmd"
  },
  "runtimeArgs": [
    "${workspaceFolder}/.",
    "--remote-debugging-port=9222"
  ],
  "webRoot": "${workspaceFolder}"
}
```

Note: Requires [Debugger for Chrome](https://marketplace.visualstudio.com/items?itemName=msjsdiag.debugger-for-chrome) extension for renderer process debugging. You cannot open DevTools while using renderer debug config.

### Console Access

Open DevTools with `Ctrl+Shift+I`. Export console output by right-clicking and selecting "Save as..."

## Common Patterns

### Adding a New Setting
1. Add setting to `DEFAULT_CONFIG` in ConfigManager
2. Create getter/setter functions in ConfigManager
3. Add UI controls in [app/settings.ejs](app/settings.ejs)
4. Wire up UI events in [app/assets/js/scripts/settings.js](app/assets/js/scripts/settings.js)

### Modifying Authentication Flow
- All auth logic is in [app/assets/js/authmanager.js](app/assets/js/authmanager.js)
- Microsoft OAuth flow uses main process windows (see index.js)
- Account storage managed by ConfigManager
- UI views: [app/loginOptions.ejs](app/loginOptions.ejs) and [app/login.ejs](app/login.ejs)

### Changing Distribution Source
- Default URL set in ConfigManager
- Fetched by DistroManager on startup
- Cached locally after first fetch
- Schema defined in helios-distribution-types package

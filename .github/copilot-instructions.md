# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.2
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

## Project Context

You are working on an ioBroker adapter. ioBroker is an integration platform for the Internet of Things, focused on building smart home and industrial IoT solutions. Adapters are plugins that connect ioBroker to external systems, devices, or services.

### Adapter-Specific Context: euSec (Eufy Security)

This adapter integrates Eufy Security cameras and stations with ioBroker. Key characteristics:

- **Purpose**: Support for Eufy-Security cameras with stations, enabling control and monitoring of Eufy security devices
- **Primary Function**: Connects to Eufy Cloud servers to manage security cameras, doorbells, and stations
- **Target Devices**: Eufy security cameras (EufyCam), video doorbells, HomeBase stations, and other Eufy Security devices
- **Key Dependencies**: 
  - `eufy-security-client` (v3.1.1): Core library for communication with Eufy devices
  - `@bropat/fluent-ffmpeg`: Video stream processing
  - `go2rtc-static`: Real-time streaming capabilities
  - `ffmpeg-for-homebridge`: Video encoding/decoding
- **Connection Type**: Cloud-based with local/remote P2P connection support to stations
- **Data Source**: Polling-based with configurable interval (default: 10 seconds)
- **Configuration Requirements**:
  - Eufy Cloud account credentials (username/password)
  - Country selection
  - Polling interval configuration
  - P2P connection type settings
  - Go2RTC streaming configuration (API, RTSP, SRTP, WebRTC ports)
  - Livestream duration limits
  - Event recording duration
  - Alarm sound duration
- **Special Considerations**:
  - Requires Node.js >= 20
  - Requires js-controller >= 6.0.11
  - Requires admin >= 7.6.17
  - Encrypted native properties for passwords (password, go2rtc_rtsp_password)
  - Message box support for user interactions
  - Adapter must be stopped before updates
  - TypeScript-based implementation (source in `src/`, built to `build/`)

## Testing

### Unit Testing
- Use Jest as the primary testing framework for ioBroker adapters
- Create tests for all adapter main functions and helper methods
- Test error handling scenarios and edge cases
- Mock external API calls and hardware dependencies
- For adapters connecting to APIs/devices not reachable by internet, provide example data files to allow testing of functionality without live connections
- Example test structure:
  ```javascript
  describe('AdapterName', () => {
    let adapter;
    
    beforeEach(() => {
      // Setup test adapter instance
    });
    
    test('should initialize correctly', () => {
      // Test adapter initialization
    });
  });
  ```

### Integration Testing

**IMPORTANT**: Use the official `@iobroker/testing` framework for all integration tests. This is the ONLY correct way to test ioBroker adapters.

**Official Documentation**: https://github.com/ioBroker/testing

#### Framework Structure
Integration tests MUST follow this exact pattern:

```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');

// Define test coordinates or configuration
const TEST_COORDINATES = '52.520008,13.404954'; // Berlin
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

// Use tests.integration() with defineAdditionalTests
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Test adapter with specific configuration', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should configure and start adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        harness = getHarness();
                        
                        // Get adapter object using promisified pattern
                        const obj = await new Promise((res, rej) => {
                            harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                                if (err) return rej(err);
                                res(o);
                            });
                        });
                        
                        if (!obj) {
                            return reject(new Error('Adapter object not found'));
                        }

                        // Configure adapter properties
                        Object.assign(obj.native, {
                            position: TEST_COORDINATES,
                            createCurrently: true,
                            createHourly: true,
                            createDaily: true,
                            // Add other configuration as needed
                        });

                        // Set the updated configuration
                        harness.objects.setObject(obj._id, obj);

                        console.log('✅ Step 1: Configuration written, starting adapter...');
                        
                        // Start adapter and wait
                        await harness.startAdapterAndWait();
                        
                        console.log('✅ Step 2: Adapter started');

                        // Wait for adapter to process data
                        const waitMs = 15000;
                        await wait(waitMs);

                        console.log('🔍 Step 3: Checking states after adapter run...');
                        
                        // Get all states created by adapter
                        const stateIds = await harness.dbConnection.getStateIDs('your-adapter.0.*');
                        
                        console.log(`📊 Found ${stateIds.length} states`);

                        if (stateIds.length > 0) {
                            console.log('✅ Adapter successfully created states');
                            
                            // Show sample of created states
                            const allStates = await new Promise((res, rej) => {
                                harness.states.getStates(stateIds, (err, states) => {
                                    if (err) return rej(err);
                                    res(states);
                                });
                            });

                            console.log(`📋 Sample states created:`);
                            Object.keys(allStates).slice(0, 5).forEach((id) => {
                                const state = allStates[id];
                                console.log(`   ${id}: ${JSON.stringify(state)}`);
                            });
                        }

                        resolve();
                    } catch (err) {
                        reject(err);
                    }
                });
            }).timeout(60000);
        });
    }
});
```

#### Key Requirements

1. **Never use `harness.startAdapter()`** - ALWAYS use `harness.startAdapterAndWait()` to ensure proper synchronization
2. **Object and State Access**: Use proper callback or promise patterns with harness.objects and harness.states
3. **Timeout Configuration**: Set realistic timeouts for operations that fetch remote data (typically 30-120 seconds)
4. **Error Messages**: Provide specific, actionable error messages that include:
   - What was expected
   - What actually happened
   - How to fix the issue (check credentials, API endpoints, network settings, etc.)

#### Common Integration Test Patterns

##### Testing Adapter with Demo/Test Credentials

For adapters that connect to external APIs, provide demo credentials or test data:

```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');
const crypto = require('crypto');

// Helper to encrypt passwords for secure native storage
async function encryptPassword(harness, password) {
    return new Promise((resolve, reject) => {
        harness.sendTo('system.adapter.your-adapter.0', 'encrypt', { data: password }, (result) => {
            if (result && result.encrypted) {
                resolve(result.encrypted);
            } else {
                reject(new Error('Password encryption failed'));
            }
        });
    });
}

// Run integration tests with demo credentials
tests.integration(path.join(__dirname, ".."), {
    defineAdditionalTests({ suite }) {
        suite("API Testing with Demo Credentials", (getHarness) => {
            let harness;
            
            before(() => {
                harness = getHarness();
            });

            it("Should connect to API and initialize with demo credentials", async () => {
                console.log("Setting up demo credentials...");
                
                if (harness.isAdapterRunning()) {
                    await harness.stopAdapter();
                }
                
                const encryptedPassword = await encryptPassword(harness, "demo_password");
                
                await harness.changeAdapterConfig("your-adapter", {
                    native: {
                        username: "demo@provider.com",
                        password: encryptedPassword,
                        // other config options
                    }
                });

                console.log("Starting adapter with demo credentials...");
                await harness.startAdapter();
                
                // Wait for API calls and initialization
                await new Promise(resolve => setTimeout(resolve, 60000));
                
                const connectionState = await harness.states.getStateAsync("your-adapter.0.info.connection");
                
                if (connectionState && connectionState.val === true) {
                    console.log("✅ SUCCESS: API connection established");
                    return true;
                } else {
                    throw new Error("API Test Failed: Expected API connection to be established with demo credentials. " +
                        "Check logs above for specific API errors (DNS resolution, 401 Unauthorized, network issues, etc.)");
                }
            }).timeout(120000);
        });
    }
});
```

### euSec-Specific Testing Considerations

When working with this Eufy Security adapter:
- Mock the `eufy-security-client` library for unit tests since it requires actual Eufy Cloud credentials
- Integration tests should verify P2P connection handling
- Test video stream lifecycle (start, stop, timeout after maxLivestreamDuration)
- Verify proper cleanup in the `unload()` method (close streams, disconnect P2P)
- Test message box interactions for user confirmations
- Consider testing with encrypted vs. unencrypted password configurations

## Code Style and Structure

### TypeScript Usage
- This adapter uses TypeScript for all source code
- Source files are in `src/` directory
- Built JavaScript output goes to `build/` directory
- Use `npm run build` to compile TypeScript to JavaScript
- Use `npm run watch` during development for automatic recompilation
- Follow strict TypeScript typing - avoid `any` when possible

### Linting and Code Quality
- ESLint is configured via `eslint.config.mjs`
- Prettier is configured via `prettier.config.mjs`
- Run `npm run lint` to check code style
- Run `npm run check` to verify TypeScript compilation without emitting files
- All pull requests must pass linting and type checking

### Project Structure
```
├── src/               # TypeScript source files
│   └── main.ts        # Main adapter file
├── build/             # Compiled JavaScript (gitignored)
├── admin/             # Admin UI configuration
├── test/              # Test files
│   ├── package/       # Package tests
│   └── integration/   # Integration tests
├── docs/              # Documentation
└── www/               # Static web assets
```

## ioBroker Adapter Development Guidelines

### Adapter Structure

**Main Adapter File (main.ts):**
```typescript
import * as utils from '@iobroker/adapter-core';

class YourAdapter extends utils.Adapter {
    public constructor(options: Partial<utils.AdapterOptions> = {}) {
        super({
            ...options,
            name: 'your-adapter',
        });
        this.on('ready', this.onReady.bind(this));
        this.on('stateChange', this.onStateChange.bind(this));
        this.on('message', this.onMessage.bind(this));
        this.on('unload', this.onUnload.bind(this));
    }

    private async onReady(): Promise<void> {
        // Initialize your adapter here
        this.log.info('Adapter started');
    }

    private onStateChange(id: string, state: ioBroker.State | null | undefined): void {
        if (state) {
            this.log.info(`state ${id} changed: ${state.val} (ack = ${state.ack})`);
        } else {
            this.log.info(`state ${id} deleted`);
        }
    }

    private onMessage(obj: ioBroker.Message): void {
        if (typeof obj === 'object' && obj.message) {
            // Handle message box messages
        }
    }

    private onUnload(callback: () => void): void {
        try {
            // Clean up resources here
            this.log.info('Cleaned everything up...');
            callback();
        } catch (e) {
            callback();
        }
    }
}

if (require.main !== module) {
    module.exports = (options: Partial<utils.AdapterOptions> | undefined) => new YourAdapter(options);
} else {
    (() => new YourAdapter())();
}
```

### State Management

**Creating States:**
```typescript
await this.setObjectNotExistsAsync('testVariable', {
    type: 'state',
    common: {
        name: 'Test Variable',
        type: 'string',
        role: 'text',
        read: true,
        write: true,
    },
    native: {},
});
```

**Reading States:**
```typescript
const state = await this.getStateAsync('testVariable');
if (state) {
    this.log.info(`Current value: ${state.val}`);
}
```

**Writing States:**
```typescript
await this.setStateAsync('testVariable', { val: 'Hello World', ack: true });
```

**Subscribing to States:**
```typescript
this.subscribeStates('testVariable');
```

### Logging Best Practices

Use appropriate log levels:
- `this.log.error()` - Critical errors that prevent functionality
- `this.log.warn()` - Warning conditions that might need attention
- `this.log.info()` - General informational messages
- `this.log.debug()` - Detailed debugging information

Example:
```typescript
this.log.debug('Fetching data from API...');
this.log.info('Successfully connected to device');
this.log.warn('Connection timeout, retrying...');
this.log.error('Failed to authenticate: invalid credentials');
```

### Error Handling

**Always handle errors gracefully:**
```typescript
try {
    await this.someAsyncOperation();
} catch (error) {
    this.log.error(`Operation failed: ${error}`);
    // Don't crash the adapter - handle the error appropriately
}
```

**For external API calls:**
```typescript
try {
    const response = await fetch(url);
    if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
    }
    const data = await response.json();
} catch (error) {
    this.log.error(`API call failed: ${error instanceof Error ? error.message : error}`);
}
```

### Configuration Handling

**Accessing Configuration:**
```typescript
const username = this.config.username;
const pollingInterval = this.config.pollingInterval || 10;
```

**Validating Configuration:**
```typescript
private validateConfig(): boolean {
    if (!this.config.username || !this.config.password) {
        this.log.error('Username and password are required');
        return false;
    }
    return true;
}
```

### Admin Configuration (JSON Config)

This adapter uses JSON Config for the admin interface (adminUI.config: "json" in io-package.json).

**Structure:**
- Configuration is defined in `admin/jsonConfig.json`
- Supports responsive layouts, tabs, panels, and various input types
- Multilingual labels using translation files in `admin/i18n/`

### Lifecycle Management

**Clean Shutdown:**
```typescript
private onUnload(callback: () => void): void {
    try {
        // Stop all timers
        if (this.pollingInterval) {
            clearInterval(this.pollingInterval);
        }
        
        // Close all connections
        if (this.apiClient) {
            this.apiClient.disconnect();
        }
        
        // Clean up resources
        this.log.info('Adapter stopped and cleaned up');
        callback();
    } catch (e) {
        callback();
    }
}
```

### Timers and Intervals

**Using Adapter-Safe Timers:**
```typescript
// Store timeout/interval references for cleanup
private pollingInterval?: NodeJS.Timeout;

private startPolling(): void {
    this.pollingInterval = setInterval(() => {
        this.pollData();
    }, this.config.pollingInterval * 1000);
}

private onUnload(callback: () => void): void {
    if (this.pollingInterval) {
        clearInterval(this.pollingInterval);
    }
    callback();
}
```

### Message Box Communication

**Handling Messages (for adapter instance communication):**
```typescript
private onMessage(obj: ioBroker.Message): void {
    if (typeof obj === 'object' && obj.message) {
        if (obj.command === 'testConnection') {
            this.testConnection()
                .then(result => {
                    if (obj.callback) {
                        this.sendTo(obj.from, obj.command, result, obj.callback);
                    }
                })
                .catch(error => {
                    if (obj.callback) {
                        this.sendTo(obj.from, obj.command, { error: error.message }, obj.callback);
                    }
                });
        }
    }
}
```

## Package and Dependency Management

### Package.json Requirements

**Essential Fields:**
```json
{
    "name": "iobroker.your-adapter",
    "version": "x.y.z",
    "description": "ioBroker adapter for...",
    "author": {
        "name": "Your Name",
        "email": "your.email@example.com"
    },
    "keywords": ["ioBroker", "adapter", "..."],
    "engines": {
        "node": ">=20.0.0"
    },
    "dependencies": {
        "@iobroker/adapter-core": "^3.3.2"
    }
}
```

### io-package.json Configuration

**Key Sections:**
- `common.name` - Adapter name (without "iobroker." prefix)
- `common.version` - Must match package.json version
- `common.mode` - Adapter execution mode (daemon, schedule, etc.)
- `common.type` - Adapter category (alarm, climate, etc.)
- `common.connectionType` - Connection method (cloud, local)
- `common.dataSource` - Data retrieval method (poll, push)
- `native` - Default configuration values
- `encryptedNative` - Configuration fields to encrypt (passwords, tokens)

### Release Management

This adapter uses `@alcalzone/release-script` for releases:
```bash
npm run release
```

The release script:
- Updates version numbers
- Updates CHANGELOG
- Creates git tags
- Publishes to npm (if configured)

## Common ioBroker Patterns

### Device Discovery
```typescript
private async discoverDevices(): Promise<void> {
    const devices = await this.api.getDevices();
    
    for (const device of devices) {
        await this.createDeviceObjects(device);
    }
}

private async createDeviceObjects(device: Device): Promise<void> {
    await this.setObjectNotExistsAsync(device.id, {
        type: 'device',
        common: {
            name: device.name,
        },
        native: device,
    });
    
    await this.setObjectNotExistsAsync(`${device.id}.temperature`, {
        type: 'state',
        common: {
            name: 'Temperature',
            type: 'number',
            role: 'value.temperature',
            unit: '°C',
            read: true,
            write: false,
        },
        native: {},
    });
}
```

### Data Polling
```typescript
private async pollData(): Promise<void> {
    try {
        const data = await this.api.fetchData();
        await this.updateStates(data);
    } catch (error) {
        this.log.error(`Polling failed: ${error}`);
    }
}

private async updateStates(data: DataObject): Promise<void> {
    for (const [key, value] of Object.entries(data)) {
        await this.setStateAsync(key, { val: value, ack: true });
    }
}
```

### Handling Controllable States
```typescript
private async onStateChange(id: string, state: ioBroker.State | null | undefined): Promise<void> {
    if (state && !state.ack) {
        // User changed a state, send command to device
        try {
            await this.sendCommandToDevice(id, state.val);
            // Acknowledge the change
            await this.setStateAsync(id, { val: state.val, ack: true });
        } catch (error) {
            this.log.error(`Failed to control ${id}: ${error}`);
        }
    }
}
```

## Security Best Practices

### Credential Handling
- NEVER log passwords or tokens
- Use `encryptedNative` in io-package.json for sensitive data
- Validate credentials before attempting connections
- Clear sensitive data from memory when no longer needed

### Input Validation
```typescript
private validateInput(value: any, expectedType: string): boolean {
    if (typeof value !== expectedType) {
        this.log.error(`Invalid input: expected ${expectedType}, got ${typeof value}`);
        return false;
    }
    return true;
}
```

### API Communication
- Use HTTPS for all external API calls
- Implement proper timeout handling
- Add retry logic with exponential backoff
- Handle rate limiting appropriately

## euSec-Specific Coding Standards

When working on this Eufy Security adapter:

### eufy-security-client Integration
- Always properly initialize and close the eufy-security-client connections
- Handle P2P connection states and errors gracefully
- Implement proper event listeners for device events
- Clean up all event listeners in the `unload()` method

### Video Stream Management
- Respect the `maxLivestreamDuration` configuration
- Properly stop streams when they're no longer needed
- Handle go2rtc lifecycle (start, stop, restart)
- Clean up FFmpeg processes on adapter shutdown

### State Structure
- Follow the existing state hierarchy for devices and stations
- Use consistent naming conventions for camera states
- Properly handle encrypted configuration (passwords)
- Set appropriate roles for states (e.g., 'value.temperature', 'switch.power')

### Error Recovery
- Implement reconnection logic for lost cloud connections
- Handle P2P connection failures gracefully
- Retry failed stream starts with appropriate delays
- Log detailed error information for troubleshooting

### Performance Considerations
- Optimize polling intervals to balance responsiveness and API load
- Cache device information to reduce API calls
- Implement efficient state update mechanisms
- Consider memory usage when handling video streams

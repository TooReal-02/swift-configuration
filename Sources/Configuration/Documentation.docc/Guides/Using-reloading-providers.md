# Using reloading providers

Automatically reload configuration from files when they change.

## Overview

A reloading provider monitors configuration files for changes and automatically updates your application's configuration without requiring restarts. Swift Configuration provides:

- ``ReloadingJSONProvider`` for JSON configuration files.
- ``ReloadingYAMLProvider`` for YAML configuration files.

### Basic usage

#### Creating and running providers

Reloading providers run in a [`ServiceGroup`](https://swiftpackageindex.com/swift-server/swift-service-lifecycle/documentation/servicelifecycle/servicegroup):

```swift
import ServiceLifecycle

let provider = try await ReloadingJSONProvider(
    filePath: "/etc/config.json",
    pollInterval: .seconds(15)
)

let serviceGroup = ServiceGroup(
    services: [provider],
    logger: logger
)

try await serviceGroup.run()
```

#### Reading configuration

Use a reloading provider in the same fashion as a static provider, pass it to a ``ConfigReader``:

```swift
let config = ConfigReader(provider: provider)
let host = config.string(
    forKey: "database.host",
    default: "localhost"
)
```

#### Poll interval considerations

Choose poll intervals based on how quickly you need to detect changes:

```swift
// Development: Quick feedback
pollInterval: .seconds(1)

// Production: Balanced performance (default)
pollInterval: .seconds(15)

// Batch processing: Resource efficient
pollInterval: .seconds(300)
```

### Watching for changes

The following sections provide examples of watching for changes in configuration from a reloading provider.

#### Individual values

The example below watches for updates in a single key, `database.host`:

```swift
try await config.watchString(
    forKey: "database.host"
) { updates in
    for await host in updates {
        print("Database host updated: \(host)")
    }
}
```

#### Configuration snapshots

The following example reads the `database.host` and `database.password` key with the guarantee that they are read from the same update of the reloading file:

```swift
try await config.watchSnapshot { updates in
    for await snapshot in updates {
        let host = snapshot.string(forKey: "database.host")
        let password = snapshot.string(forKey: "database.password", isSecret: true)
        print("Configuration updated - Database: \(host)")
    }
}
```

### Comparison with static providers

| Feature | Static providers | Reloading providers |
|---------|------------------|---------------------|
| **File reading** | Load once at startup | Reloading on change |
| **Service lifecycle** | Not required | Conforms to `Service` and must run in a `ServiceGroup` |
| **Configuration updates** | Require restart | Automatic reload |

### Advanced features

#### Configuration-driven setup

The following example sets up an environment variable provider to select the path
and interval to watch for a JSON file that contains the configuration for your app:

```swift
let envProvider = EnvironmentVariablesProvider()
let envConfig = ConfigReader(provider: envProvider)

let jsonProvider = try await ReloadingJSONProvider(
    config: envConfig.scoped(to: "json")  
    // Reads JSON_FILE_PATH and JSON_POLL_INTERVAL_SECONDS
)
```

### Migration from static providers

1. **Replace initialization**:
   ```swift
   // Before
   let provider = try await JSONProvider(filePath: "/etc/config.json")

   // After
   let provider = try await ReloadingJSONProvider(filePath: "/etc/config.json")
   ```

2. **Add the provider to a ServiceGroup**:
   ```swift
   let serviceGroup = ServiceGroup(services: [provider], logger: logger)
   try await serviceGroup.run()
   ```

3. **Use ConfigReader**:
   ```swift
   let config = ConfigReader(provider: provider)

   // Live updates.
   try await config.watchDouble(forKey: "timeout") { updates in
       // Handle changes
   }

   // On-demand reads - returns the current value, so might change over time.
   let timeout = config.double(forKey: "timeout", default: 60.0)
   ```

For guidance on choosing between get, fetch, and watch access patterns with reloading providers, see <doc:Choosing-access-patterns>. For troubleshooting reloading provider issues, check out <doc:Troubleshooting>. To learn about in-memory providers as an alternative, see <doc:Using-in-memory-providers>.


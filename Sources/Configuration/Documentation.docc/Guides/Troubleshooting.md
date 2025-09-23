# Troubleshooting and access reporting

Check out some techniques to debug unexpected issues and to increase visibility into accessed config values.

## Overview

### Debugging configuration issues

If your configuration values aren't being read correctly, check:

1. **Key formatting**: Make sure the key format matches your `ConfigKeyDecoder`. The default is dot-notation (for example, `database.url`).

2. **Environment variable naming**: When using `EnvironmentVariablesProvider`, keys are automatically converted to uppercase with dots replaced by underscores. For example, `database.url` becomes `DATABASE_URL`.

3. **Provider ordering**: When using multiple providers, remember they're checked in order, and the first one that returns a value wins.

4. **Debug with an access reporter**: Use the access reporting feature to see which keys are being queried and what values (if any) are being returned. See the next section for details.

For guidance on selecting the right configuration access patterns and reader methods, check out <doc:Choosing-access-patterns> and <doc:Choosing-reader-methods>.

### Access reporting

Configuration access reporting can help you debug issues and understand which configuration values your application is using. Swift Configuration provides two built-in ways to log access (``AccessLogger`` and ``FileAccessLogger``), and you can also implement your own ``AccessReporter``.

#### Using AccessLogger

``AccessLogger`` integrates with Swift Log and records all configuration accesses:

```swift
let logger = Logger(label: "...")
let accessLogger = AccessLogger(logger: logger)
let config = ConfigReader(provider: provider, accessReporter: accessLogger)

// Each access will now be logged.
let timeout = config.double(forKey: "http.timeout", default: 30.0)
```

This produces log entries showing:
- Which configuration keys were accessed.
- What values were returned (with secret values redacted).
- Which provider supplied the value.
- Whether default values were used.
- The location of the code reading the config value.
- The timestamp of the access.

#### Using FileAccessLogger

For writing access events to a file, especially useful during ad-hoc debugging, use ``FileAccessLogger``:

```swift
let fileLogger = try FileAccessLogger(filePath: "/var/log/myapp/config-access.log")
let config = ConfigReader(provider: provider, accessReporter: fileLogger)

// Each access will now be logged.
let timeout = config.double(forKey: "http.timeout", default: 30.0)
```

You can also enable file access logging for the whole application, without recompiling your code, by setting an environment variable:

```bash
export CONFIG_ACCESS_LOG_FILE=/var/log/myapp/config-access.log
```

And then read from the file to see one line per config access:

```bash
tail -f /var/log/myapp/config-access.log
```

### Error handling

#### Provider errors
If any provider throws an error during lookup:
- **Required methods** (`getRequired*`): Error is immediately thrown to the caller.
- **Optional methods** (`get*` with or without defaults): Error is handled gracefully, nil or default value is returned.

> Tip: Even when an error gets handled gracefully, you can log it using an ``AccessReporter``.

#### Missing values
When no provider has the requested value:
- **Methods with defaults**: Return the provided default value.
- **Methods without defaults**: Return nil.
- **Required methods**: Throw an error.

### Reloading provider troubleshooting

#### Configuration not updating

If your reloading provider isn't detecting file changes:

1. **Check ServiceGroup**: Ensure the provider is running in a ServiceGroup.
2. **Enable verbose logging**: The built-in providers use Swift Log for detailed logging, which can help spot the underlying issue.
3. **Verify file path**: Confirm the file path is correct, the file exists, and file permissions are correct.
4. **Check poll interval**: Consider if your poll interval is appropriate for your use case.

#### ServiceGroup integration issues

Common ServiceGroup problems:

```swift
// Incorrect: Provider not included in ServiceGroup
let provider = try await ReloadingJSONProvider(filePath: "/etc/config.json")
let config = ConfigReader(provider: provider)
// File monitoring won't work

// Correct: Provider runs in ServiceGroup
let provider = try await ReloadingJSONProvider(filePath: "/etc/config.json")
let serviceGroup = ServiceGroup(services: [provider], logger: logger)
try await serviceGroup.run()
```

For more details about reloading providers and ServiceLifecycle integration, see <doc:Using-reloading-providers>. To learn about proper configuration practices that can prevent common issues, check out <doc:Best-practices>.

# Exercise 2: JSON Configuration Manager

**Difficulty**: Intermediate  
**Estimated Time**: 30-45 minutes  
**Concepts Covered**: JSON encoding/decoding, file I/O, error handling, maps, struct tags, pointers

---

## Objective

Build a configuration file manager that can:
1. Read configuration from a JSON file
2. Modify configuration values in memory
3. Validate configuration
4. Save updated configuration back to JSON file
5. Support nested configuration structures

This is directly applicable to API development where you'll frequently read config files, environment variables, and work with JSON.

---

## Requirements

### Configuration Structure

Create structs to represent application configuration:

```go
type DatabaseConfig struct {
    Host     string `json:"host"`
    Port     int    `json:"port"`
    Username string `json:"username"`
    Password string `json:"password"`
    Database string `json:"database"`
    MaxConns int    `json:"max_connections"`
}

type ServerConfig struct {
    Host         string `json:"host"`
    Port         int    `json:"port"`
    ReadTimeout  int    `json:"read_timeout"`
    WriteTimeout int    `json:"write_timeout"`
}

type APIConfig struct {
    Version     string `json:"version"`
    RateLimit   int    `json:"rate_limit"`
    EnableCache bool   `json:"enable_cache"`
}

type Config struct {
    AppName  string          `json:"app_name"`
    Env      string          `json:"environment"`
    Server   ServerConfig    `json:"server"`
    Database DatabaseConfig  `json:"database"`
    API      APIConfig       `json:"api"`
}
```

### Functions to Implement

#### 1. `loadConfig`

`loadConfig(filename string) (*Config, error)`

Read JSON file. Parse into Config struct. Return error if file doesn't exist or JSON is invalid.

#### 2. `saveConfig`

`saveConfig(config *Config, filename string) error`

Convert Config to JSON. Write to file with proper formatting (indentation). Return error if write fails.

#### 3. `validateConfig`

`validateConfig(config *Config) []string`

Check for invalid values: Server port must be between 1024 and 65535. Database port must be between 1024 and 65535. RateLimit must be positive. MaxConns must be positive. Return slice of error messages (empty if valid).

#### 4. `updateServerPort`

`updateServerPort(config *Config, newPort int) error`

Update server port. Validate the new port. Return error if invalid.

#### 5. `displayConfig`

`displayConfig(config *Config)`

Print configuration in a readable format.

#### 6. `createDefaultConfig`

`createDefaultConfig() *Config`

Create a default configuration with sensible values.

---

## Example JSON File

Create a file named `config.json`:

```json
{
  "app_name": "MyAPI",
  "environment": "development",
  "server": {
    "host": "localhost",
    "port": 8080,
    "read_timeout": 15,
    "write_timeout": 15
  },
  "database": {
    "host": "localhost",
    "port": 5432,
    "username": "admin",
    "password": "secret123",
    "database": "myapp_db",
    "max_connections": 100
  },
  "api": {
    "version": "v1",
    "rate_limit": 100,
    "enable_cache": true
  }
}
```

---

## Example Output

```
=== Configuration Manager ===

Loading configuration from config.json...
✓ Configuration loaded successfully

Current Configuration:
==================================================
App Name: MyAPI
Environment: development

Server:
  Host: localhost
  Port: 8080
  Read Timeout: 15s
  Write Timeout: 15s

Database:
  Host: localhost
  Port: 5432
  Username: admin
  Database: myapp_db
  Max Connections: 100

API:
  Version: v1
  Rate Limit: 100 req/min
  Cache Enabled: true
==================================================

Validating configuration...
✓ Configuration is valid

Updating server port to 9090...
✓ Server port updated successfully

Saving configuration to config_updated.json...
✓ Configuration saved successfully

Testing validation with invalid values...
Invalid configuration detected:
  - Server port must be between 1024 and 65535
  - Database max_connections must be positive
```

---

## Starter Code

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
    "io"
)

type DatabaseConfig struct {
    Host     string `json:"host"`
    Port     int    `json:"port"`
    Username string `json:"username"`
    Password string `json:"password"`
    Database string `json:"database"`
    MaxConns int    `json:"max_connections"`
}

type ServerConfig struct {
    Host         string `json:"host"`
    Port         int    `json:"port"`
    ReadTimeout  int    `json:"read_timeout"`
    WriteTimeout int    `json:"write_timeout"`
}

type APIConfig struct {
    Version     string `json:"version"`
    RateLimit   int    `json:"rate_limit"`
    EnableCache bool   `json:"enable_cache"`
}

type Config struct {
    AppName  string          `json:"app_name"`
    Env      string          `json:"environment"`
    Server   ServerConfig    `json:"server"`
    Database DatabaseConfig  `json:"database"`
    API      APIConfig       `json:"api"`
}

// TODO: Implement loadConfig
func loadConfig(filename string) (*Config, error) {
    // Your code here
}

// TODO: Implement saveConfig
func saveConfig(config *Config, filename string) error {
    // Your code here
}

// TODO: Implement validateConfig
func validateConfig(config *Config) []string {
    // Your code here
}

// TODO: Implement updateServerPort
func updateServerPort(config *Config, newPort int) error {
    // Your code here
}

// TODO: Implement displayConfig
func displayConfig(config *Config) {
    // Your code here
}

// TODO: Implement createDefaultConfig
func createDefaultConfig() *Config {
    // Your code here
}

func main() {
    fmt.Println("=== Configuration Manager ===\n")
    
    // Load existing config
    fmt.Println("Loading configuration from config.json...")
    config, err := loadConfig("config.json")
    if err != nil {
        fmt.Println("Error loading config:", err)
        fmt.Println("Creating default configuration...")
        config = createDefaultConfig()
    } else {
        fmt.Println("✓ Configuration loaded successfully")
    }
    
    // Display config
    fmt.Println("\nCurrent Configuration:")
    displayConfig(config)
    
    // Validate
    fmt.Println("\nValidating configuration...")
    errors := validateConfig(config)
    if len(errors) == 0 {
        fmt.Println("✓ Configuration is valid")
    } else {
        fmt.Println("Invalid configuration:")
        for _, err := range errors {
            fmt.Println("  -", err)
        }
    }
    
    // Update server port
    fmt.Println("\nUpdating server port to 9090...")
    err = updateServerPort(config, 9090)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("✓ Server port updated successfully")
    }
    
    // Save to new file
    fmt.Println("\nSaving configuration to config_updated.json...")
    err = saveConfig(config, "config_updated.json")
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("✓ Configuration saved successfully")
    }
    
    // Test validation with invalid config
    fmt.Println("\nTesting validation with invalid values...")
    config.Server.Port = 70000  // Invalid
    config.Database.MaxConns = -5  // Invalid
    
    errors = validateConfig(config)
    if len(errors) > 0 {
        fmt.Println("Invalid configuration detected:")
        for _, err := range errors {
            fmt.Println("  -", err)
        }
    }
}
```

---

## Hints

### Hint 1: Reading JSON from file
```go
func loadConfig(filename string) (*Config, error) {
    // Open file
    file, err := os.Open(filename)
    if err != nil {
        return nil, err
    }
    defer file.Close()
    
    // Read file contents
    data, err := io.ReadAll(file)
    if err != nil {
        return nil, err
    }
    
    // Parse JSON
    var config Config
    err = json.Unmarshal(data, &config)
    if err != nil {
        return nil, err
    }
    
    return &config, nil
}
```

### Hint 2: Writing JSON to file
```go
func saveConfig(config *Config, filename string) error {
    // Convert to pretty JSON
    data, err := json.MarshalIndent(config, "", "  ")
    if err != nil {
        return err
    }
    
    // Write to file
    return os.WriteFile(filename, data, 0644)
}
```

### Hint 3: Validation
```go
func validateConfig(config *Config) []string {
    errors := []string{}
    
    // Check server port
    if config.Server.Port < 1024 || config.Server.Port > 65535 {
        errors = append(errors, "Server port must be between 1024 and 65535")
    }
    
    // Add more validation checks...
    
    return errors
}
```

---

## Bonus Challenges

### Bonus 1: Environment Variable Override
Add a function that allows overriding config values with environment variables:

```go
func applyEnvOverrides(config *Config) {
    // Check for environment variables like:
    // SERVER_PORT, DATABASE_HOST, etc.
    // Use os.Getenv() and strconv.Atoi()
}
```

### Bonus 2: Config Merging
Create a function that merges two configs, with the second one taking precedence:

```go
func mergeConfigs(base *Config, override *Config) *Config {
    // Create new config with values from both
}
```

### Bonus 3: Config Diff
Show what changed between two configs:

```go
func configDiff(old *Config, new *Config) []string {
    // Return list of changes like:
    // "Server.Port changed from 8080 to 9090"
}
```

### Bonus 4: Secure Password Display
Modify `displayConfig` to mask the database password:

```go
func maskPassword(password string) string {
    if len(password) <= 2 {
        return "***"
    }
    return password[:2] + "***"
}
```

### Bonus 5: Config Templates
Create different config templates for different environments:

```go
func createProductionConfig() *Config {
    // Production-ready defaults
}

func createDevelopmentConfig() *Config {
    // Development defaults
}
```

---

## Testing Your Code

1. **Create the initial config.json file** with the JSON provided above
2. **Run the program** and verify:
   - Config loads correctly
   - Validation passes
   - Updates work
   - File saves properly
3. **Test error cases**:
   - Delete config.json and verify default creation works
   - Create invalid JSON and verify error handling
   - Try invalid port numbers
4. **Check the generated file**:
   - Verify config_updated.json has proper formatting
   - Verify the port was updated

---

## What You're Learning

- ✓ JSON encoding and decoding
- ✓ Struct tags for JSON mapping
- ✓ File I/O (reading and writing files)
- ✓ Error handling and propagation
- ✓ Pointer receivers (modifying structs)
- ✓ Validation logic
- ✓ Working with nested structs
- ✓ Using defer for cleanup

This directly applies to API development where you'll:
- Read configuration files
- Parse JSON request bodies
- Validate input data
- Generate JSON responses
- Handle file operations
- Manage application settings

---

## Extension Ideas

If you want to go further:

1. **Watch for config changes**: Use `fsnotify` package to reload config when file changes
2. **Remote config**: Fetch config from a URL instead of local file
3. **Encrypted secrets**: Encrypt sensitive values like passwords
4. **Config validation schema**: Use JSON Schema to validate config structure
5. **Hot reload**: Allow config updates without restarting the application

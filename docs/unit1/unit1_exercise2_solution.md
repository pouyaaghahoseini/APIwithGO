# Exercise 2 Solution: JSON Configuration Manager

**Complete implementation with explanations**

---

## Full Solution Code

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "os"
    "strconv"
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
    AppName  string         `json:"app_name"`
    Env      string         `json:"environment"`
    Server   ServerConfig   `json:"server"`
    Database DatabaseConfig `json:"database"`
    API      APIConfig      `json:"api"`
}

// loadConfig reads and parses a JSON configuration file
func loadConfig(filename string) (*Config, error) {
    // Open the file
    file, err := os.Open(filename)
    if err != nil {
        return nil, fmt.Errorf("failed to open config file: %w", err)
    }
    defer file.Close()
    
    // Read file contents
    data, err := io.ReadAll(file)
    if err != nil {
        return nil, fmt.Errorf("failed to read config file: %w", err)
    }
    
    // Parse JSON into Config struct
    var config Config
    err = json.Unmarshal(data, &config)
    if err != nil {
        return nil, fmt.Errorf("failed to parse JSON: %w", err)
    }
    
    return &config, nil
}

// saveConfig writes configuration to a JSON file with formatting
func saveConfig(config *Config, filename string) error {
    // Convert to pretty JSON with 2-space indentation
    data, err := json.MarshalIndent(config, "", "  ")
    if err != nil {
        return fmt.Errorf("failed to marshal config: %w", err)
    }
    
    // Write to file with read/write permissions for owner
    err = os.WriteFile(filename, data, 0644)
    if err != nil {
        return fmt.Errorf("failed to write config file: %w", err)
    }
    
    return nil
}

// validateConfig checks configuration values and returns validation errors
func validateConfig(config *Config) []string {
    errors := []string{}
    
    // Validate server port
    if config.Server.Port < 1024 || config.Server.Port > 65535 {
        errors = append(errors, "Server port must be between 1024 and 65535")
    }
    
    // Validate database port
    if config.Database.Port < 1024 || config.Database.Port > 65535 {
        errors = append(errors, "Database port must be between 1024 and 65535")
    }
    
    // Validate rate limit
    if config.API.RateLimit <= 0 {
        errors = append(errors, "API rate_limit must be positive")
    }
    
    // Validate max connections
    if config.Database.MaxConns <= 0 {
        errors = append(errors, "Database max_connections must be positive")
    }
    
    // Validate timeouts
    if config.Server.ReadTimeout <= 0 {
        errors = append(errors, "Server read_timeout must be positive")
    }
    
    if config.Server.WriteTimeout <= 0 {
        errors = append(errors, "Server write_timeout must be positive")
    }
    
    // Validate required string fields
    if config.AppName == "" {
        errors = append(errors, "app_name is required")
    }
    
    if config.Env == "" {
        errors = append(errors, "environment is required")
    }
    
    if config.Database.Database == "" {
        errors = append(errors, "Database name is required")
    }
    
    return errors
}

// updateServerPort updates the server port with validation
func updateServerPort(config *Config, newPort int) error {
    // Validate the new port
    if newPort < 1024 || newPort > 65535 {
        return fmt.Errorf("port must be between 1024 and 65535, got %d", newPort)
    }
    
    // Update the port
    config.Server.Port = newPort
    return nil
}

// displayConfig prints configuration in a readable format
func displayConfig(config *Config) {
    fmt.Println("==================================================")
    fmt.Printf("App Name: %s\n", config.AppName)
    fmt.Printf("Environment: %s\n", config.Env)
    
    fmt.Println("\nServer:")
    fmt.Printf("  Host: %s\n", config.Server.Host)
    fmt.Printf("  Port: %d\n", config.Server.Port)
    fmt.Printf("  Read Timeout: %ds\n", config.Server.ReadTimeout)
    fmt.Printf("  Write Timeout: %ds\n", config.Server.WriteTimeout)
    
    fmt.Println("\nDatabase:")
    fmt.Printf("  Host: %s\n", config.Database.Host)
    fmt.Printf("  Port: %d\n", config.Database.Port)
    fmt.Printf("  Username: %s\n", config.Database.Username)
    fmt.Printf("  Database: %s\n", config.Database.Database)
    fmt.Printf("  Max Connections: %d\n", config.Database.MaxConns)
    
    fmt.Println("\nAPI:")
    fmt.Printf("  Version: %s\n", config.API.Version)
    fmt.Printf("  Rate Limit: %d req/min\n", config.API.RateLimit)
    fmt.Printf("  Cache Enabled: %t\n", config.API.EnableCache)
    fmt.Println("==================================================")
}

// createDefaultConfig creates a configuration with sensible defaults
func createDefaultConfig() *Config {
    return &Config{
        AppName: "MyAPI",
        Env:     "development",
        Server: ServerConfig{
            Host:         "localhost",
            Port:         8080,
            ReadTimeout:  15,
            WriteTimeout: 15,
        },
        Database: DatabaseConfig{
            Host:     "localhost",
            Port:     5432,
            Username: "admin",
            Password: "changeme",
            Database: "myapp_db",
            MaxConns: 100,
        },
        API: APIConfig{
            Version:     "v1",
            RateLimit:   100,
            EnableCache: true,
        },
    }
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
        
        // Save default config
        err = saveConfig(config, "config.json")
        if err != nil {
            fmt.Println("Error saving default config:", err)
        } else {
            fmt.Println("✓ Default configuration created and saved")
        }
    } else {
        fmt.Println("✓ Configuration loaded successfully")
    }
    
    // Display config
    fmt.Println("\nCurrent Configuration:")
    displayConfig(config)
    
    // Validate
    fmt.Println("\nValidating configuration...")
    validationErrors := validateConfig(config)
    if len(validationErrors) == 0 {
        fmt.Println("✓ Configuration is valid")
    } else {
        fmt.Println("Invalid configuration:")
        for _, err := range validationErrors {
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
    config.Server.Port = 70000        // Invalid
    config.Database.MaxConns = -5     // Invalid
    config.API.RateLimit = 0          // Invalid
    config.Server.ReadTimeout = -10   // Invalid
    
    validationErrors = validateConfig(config)
    if len(validationErrors) > 0 {
        fmt.Println("Invalid configuration detected:")
        for _, err := range validationErrors {
            fmt.Println("  -", err)
        }
    }
    
    // Test error handling for invalid port
    fmt.Println("\nTesting error handling for invalid port update...")
    err = updateServerPort(config, 100)
    if err != nil {
        fmt.Printf("✓ Correctly caught error: %v\n", err)
    }
}
```

---

## Key Concepts Explained

### 1. JSON Struct Tags

```go
type DatabaseConfig struct {
    Host     string `json:"host"`        // Maps to "host" in JSON
    Password string `json:"password"`    // Maps to "password" in JSON
    MaxConns int    `json:"max_connections"`  // Different name in JSON
}
```

**Why use tags?** JSON keys often use snake_case or different names than Go's PascalCase. Tags allow you to control the mapping.

### 2. Reading Files

```go
file, err := os.Open(filename)
if err != nil {
    return nil, fmt.Errorf("failed to open: %w", err)
}
defer file.Close()  // Ensures file is closed when function exits

data, err := io.ReadAll(file)
```

**Why defer?** Guarantees cleanup happens even if there's an error later.

### 3. Error Wrapping

```go
return nil, fmt.Errorf("failed to open config file: %w", err)
```

**Why `%w`?** Wraps the original error, preserving the error chain. Better than `%v` for debugging.

### 4. JSON Marshaling

```go
// Pretty print with indentation
data, err := json.MarshalIndent(config, "", "  ")
// prefix: ""
// indent: "  " (2 spaces)

// Compact JSON (no formatting)
data, err := json.Marshal(config)
```

### 5. Pointer Receivers for Modification

```go
func updateServerPort(config *Config, newPort int) error {
    config.Server.Port = newPort  // Modifies original
    return nil
}
```

**Why pointer?** We want to modify the actual config, not a copy.

### 6. Building Error Lists

```go
errors := []string{}  // Start with empty slice

if someCondition {
    errors = append(errors, "error message")
}

return errors  // Return all errors at once
```

This pattern is great for validation - collect all errors instead of failing on first one.

---

## Bonus Solutions

### Bonus 1: Environment Variable Override

```go
func applyEnvOverrides(config *Config) {
    // Server port
    if port := os.Getenv("SERVER_PORT"); port != "" {
        if p, err := strconv.Atoi(port); err == nil {
            config.Server.Port = p
            fmt.Println("✓ Overrode SERVER_PORT from environment")
        }
    }
    
    // Database host
    if host := os.Getenv("DATABASE_HOST"); host != "" {
        config.Database.Host = host
        fmt.Println("✓ Overrode DATABASE_HOST from environment")
    }
    
    // Database port
    if port := os.Getenv("DATABASE_PORT"); port != "" {
        if p, err := strconv.Atoi(port); err == nil {
            config.Database.Port = p
            fmt.Println("✓ Overrode DATABASE_PORT from environment")
        }
    }
    
    // Database username
    if username := os.Getenv("DATABASE_USERNAME"); username != "" {
        config.Database.Username = username
        fmt.Println("✓ Overrode DATABASE_USERNAME from environment")
    }
    
    // Database password
    if password := os.Getenv("DATABASE_PASSWORD"); password != "" {
        config.Database.Password = password
        fmt.Println("✓ Overrode DATABASE_PASSWORD from environment")
    }
    
    // Environment
    if env := os.Getenv("ENVIRONMENT"); env != "" {
        config.Env = env
        fmt.Println("✓ Overrode ENVIRONMENT from environment")
    }
    
    // Enable cache
    if cache := os.Getenv("ENABLE_CACHE"); cache != "" {
        config.API.EnableCache = cache == "true" || cache == "1"
        fmt.Println("✓ Overrode ENABLE_CACHE from environment")
    }
}

// Usage in main
func main() {
    config, err := loadConfig("config.json")
    // ...
    
    fmt.Println("\nApplying environment variable overrides...")
    applyEnvOverrides(config)
}

// Test it:
// SERVER_PORT=9000 DATABASE_HOST=prod-db.example.com go run main.go
```

### Bonus 2: Config Merging

```go
func mergeConfigs(base *Config, override *Config) *Config {
    // Create a copy of base
    merged := *base
    
    // Override non-zero values from override config
    if override.AppName != "" {
        merged.AppName = override.AppName
    }
    
    if override.Env != "" {
        merged.Env = override.Env
    }
    
    // Server config
    if override.Server.Host != "" {
        merged.Server.Host = override.Server.Host
    }
    if override.Server.Port != 0 {
        merged.Server.Port = override.Server.Port
    }
    if override.Server.ReadTimeout != 0 {
        merged.Server.ReadTimeout = override.Server.ReadTimeout
    }
    if override.Server.WriteTimeout != 0 {
        merged.Server.WriteTimeout = override.Server.WriteTimeout
    }
    
    // Database config
    if override.Database.Host != "" {
        merged.Database.Host = override.Database.Host
    }
    if override.Database.Port != 0 {
        merged.Database.Port = override.Database.Port
    }
    if override.Database.Username != "" {
        merged.Database.Username = override.Database.Username
    }
    if override.Database.Password != "" {
        merged.Database.Password = override.Database.Password
    }
    if override.Database.Database != "" {
        merged.Database.Database = override.Database.Database
    }
    if override.Database.MaxConns != 0 {
        merged.Database.MaxConns = override.Database.MaxConns
    }
    
    // API config
    if override.API.Version != "" {
        merged.API.Version = override.API.Version
    }
    if override.API.RateLimit != 0 {
        merged.API.RateLimit = override.API.RateLimit
    }
    // For bool, we can't distinguish false from unset, so always override
    merged.API.EnableCache = override.API.EnableCache
    
    return &merged
}

// Usage
baseConfig := createDefaultConfig()
overrideConfig, _ := loadConfig("override.json")
finalConfig := mergeConfigs(baseConfig, overrideConfig)
```

### Bonus 3: Config Diff

```go
func configDiff(old *Config, new *Config) []string {
    changes := []string{}
    
    if old.AppName != new.AppName {
        changes = append(changes, fmt.Sprintf("AppName changed from '%s' to '%s'", old.AppName, new.AppName))
    }
    
    if old.Env != new.Env {
        changes = append(changes, fmt.Sprintf("Environment changed from '%s' to '%s'", old.Env, new.Env))
    }
    
    // Server changes
    if old.Server.Host != new.Server.Host {
        changes = append(changes, fmt.Sprintf("Server.Host changed from '%s' to '%s'", old.Server.Host, new.Server.Host))
    }
    if old.Server.Port != new.Server.Port {
        changes = append(changes, fmt.Sprintf("Server.Port changed from %d to %d", old.Server.Port, new.Server.Port))
    }
    if old.Server.ReadTimeout != new.Server.ReadTimeout {
        changes = append(changes, fmt.Sprintf("Server.ReadTimeout changed from %d to %d", old.Server.ReadTimeout, new.Server.ReadTimeout))
    }
    if old.Server.WriteTimeout != new.Server.WriteTimeout {
        changes = append(changes, fmt.Sprintf("Server.WriteTimeout changed from %d to %d", old.Server.WriteTimeout, new.Server.WriteTimeout))
    }
    
    // Database changes
    if old.Database.Host != new.Database.Host {
        changes = append(changes, fmt.Sprintf("Database.Host changed from '%s' to '%s'", old.Database.Host, new.Database.Host))
    }
    if old.Database.Port != new.Database.Port {
        changes = append(changes, fmt.Sprintf("Database.Port changed from %d to %d", old.Database.Port, new.Database.Port))
    }
    if old.Database.Username != new.Database.Username {
        changes = append(changes, fmt.Sprintf("Database.Username changed from '%s' to '%s'", old.Database.Username, new.Database.Username))
    }
    if old.Database.Password != new.Database.Password {
        changes = append(changes, "Database.Password changed")
    }
    if old.Database.Database != new.Database.Database {
        changes = append(changes, fmt.Sprintf("Database.Database changed from '%s' to '%s'", old.Database.Database, new.Database.Database))
    }
    if old.Database.MaxConns != new.Database.MaxConns {
        changes = append(changes, fmt.Sprintf("Database.MaxConns changed from %d to %d", old.Database.MaxConns, new.Database.MaxConns))
    }
    
    // API changes
    if old.API.Version != new.API.Version {
        changes = append(changes, fmt.Sprintf("API.Version changed from '%s' to '%s'", old.API.Version, new.API.Version))
    }
    if old.API.RateLimit != new.API.RateLimit {
        changes = append(changes, fmt.Sprintf("API.RateLimit changed from %d to %d", old.API.RateLimit, new.API.RateLimit))
    }
    if old.API.EnableCache != new.API.EnableCache {
        changes = append(changes, fmt.Sprintf("API.EnableCache changed from %t to %t", old.API.EnableCache, new.API.EnableCache))
    }
    
    return changes
}

// Usage
oldConfig, _ := loadConfig("config_old.json")
newConfig, _ := loadConfig("config_new.json")

changes := configDiff(oldConfig, newConfig)
if len(changes) == 0 {
    fmt.Println("No changes detected")
} else {
    fmt.Println("Configuration changes:")
    for _, change := range changes {
        fmt.Println("  -", change)
    }
}
```

### Bonus 4: Secure Password Display

```go
func maskPassword(password string) string {
    if len(password) == 0 {
        return ""
    }
    if len(password) <= 2 {
        return "***"
    }
    return password[:2] + strings.Repeat("*", len(password)-2)
}

// Updated displayConfig
func displayConfig(config *Config) {
    fmt.Println("==================================================")
    fmt.Printf("App Name: %s\n", config.AppName)
    fmt.Printf("Environment: %s\n", config.Env)
    
    fmt.Println("\nServer:")
    fmt.Printf("  Host: %s\n", config.Server.Host)
    fmt.Printf("  Port: %d\n", config.Server.Port)
    fmt.Printf("  Read Timeout: %ds\n", config.Server.ReadTimeout)
    fmt.Printf("  Write Timeout: %ds\n", config.Server.WriteTimeout)
    
    fmt.Println("\nDatabase:")
    fmt.Printf("  Host: %s\n", config.Database.Host)
    fmt.Printf("  Port: %d\n", config.Database.Port)
    fmt.Printf("  Username: %s\n", config.Database.Username)
    fmt.Printf("  Password: %s\n", maskPassword(config.Database.Password))  // Masked!
    fmt.Printf("  Database: %s\n", config.Database.Database)
    fmt.Printf("  Max Connections: %d\n", config.Database.MaxConns)
    
    fmt.Println("\nAPI:")
    fmt.Printf("  Version: %s\n", config.API.Version)
    fmt.Printf("  Rate Limit: %d req/min\n", config.API.RateLimit)
    fmt.Printf("  Cache Enabled: %t\n", config.API.EnableCache)
    fmt.Println("==================================================")
}
```

### Bonus 5: Config Templates

```go
func createProductionConfig() *Config {
    return &Config{
        AppName: "MyAPI",
        Env:     "production",
        Server: ServerConfig{
            Host:         "0.0.0.0",  // Listen on all interfaces
            Port:         8080,
            ReadTimeout:  30,  // Longer timeouts for production
            WriteTimeout: 30,
        },
        Database: DatabaseConfig{
            Host:     "prod-db.example.com",
            Port:     5432,
            Username: "prod_user",
            Password: "",  // Should be set from environment
            Database: "myapp_production",
            MaxConns: 500,  // More connections for production
        },
        API: APIConfig{
            Version:     "v1",
            RateLimit:   1000,  // Higher rate limit
            EnableCache: true,  // Always cache in production
        },
    }
}

func createDevelopmentConfig() *Config {
    return &Config{
        AppName: "MyAPI",
        Env:     "development",
        Server: ServerConfig{
            Host:         "localhost",
            Port:         8080,
            ReadTimeout:  15,
            WriteTimeout: 15,
        },
        Database: DatabaseConfig{
            Host:     "localhost",
            Port:     5432,
            Username: "dev_user",
            Password: "dev_password",
            Database: "myapp_dev",
            MaxConns: 10,  // Fewer connections for dev
        },
        API: APIConfig{
            Version:     "v1",
            RateLimit:   100,
            EnableCache: false,  // Disable cache for dev
        },
    }
}

func createTestConfig() *Config {
    return &Config{
        AppName: "MyAPI",
        Env:     "test",
        Server: ServerConfig{
            Host:         "localhost",
            Port:         8081,  // Different port
            ReadTimeout:  5,
            WriteTimeout: 5,
        },
        Database: DatabaseConfig{
            Host:     "localhost",
            Port:     5432,
            Username: "test_user",
            Password: "test_password",
            Database: "myapp_test",
            MaxConns: 5,
        },
        API: APIConfig{
            Version:     "v1",
            RateLimit:   1000,  // No rate limiting in tests
            EnableCache: false,
        },
    }
}

// Usage
func main() {
    env := os.Getenv("ENVIRONMENT")
    var config *Config
    
    switch env {
    case "production":
        config = createProductionConfig()
    case "test":
        config = createTestConfig()
    default:
        config = createDevelopmentConfig()
    }
    
    // Apply environment overrides
    applyEnvOverrides(config)
    
    // Display and use config
    displayConfig(config)
}
```

---

## Common Mistakes and How to Avoid Them

### Mistake 1: Not Using Defer for File Cleanup

```go
// WRONG - file might not be closed on error
file, err := os.Open(filename)
data, err := io.ReadAll(file)
file.Close()

// RIGHT - defer ensures cleanup
file, err := os.Open(filename)
if err != nil {
    return nil, err
}
defer file.Close()
data, err := io.ReadAll(file)
```

### Mistake 2: Forgetting to Check Marshal Errors

```go
// WRONG
data, _ := json.Marshal(config)  // Ignoring error

// RIGHT
data, err := json.Marshal(config)
if err != nil {
    return fmt.Errorf("marshal failed: %w", err)
}
```

### Mistake 3: Using Wrong JSON Tags

```go
// WRONG - won't match JSON
type Config struct {
    MaxConns int `json:"maxConns"`  // JSON has "max_connections"
}

// RIGHT
type Config struct {
    MaxConns int `json:"max_connections"`
}
```

### Mistake 4: Not Validating Before Using Config

```go
// WRONG - use config without validation
config, _ := loadConfig("config.json")
server := startServer(config.Server.Port)  // Might be invalid!

// RIGHT
config, err := loadConfig("config.json")
errors := validateConfig(config)
if len(errors) > 0 {
    log.Fatal("Invalid config:", errors)
}
server := startServer(config.Server.Port)
```

---

## What You've Learned

✅ Reading and writing JSON files  
✅ Struct tags for JSON mapping  
✅ File I/O with proper error handling  
✅ Error wrapping with `%w`  
✅ Using defer for cleanup  
✅ Validation patterns  
✅ Working with nested structs  
✅ Environment variable overrides  
✅ Configuration management patterns  

These skills directly apply to API development where you'll constantly work with:
- Configuration files
- JSON request/response bodies
- Input validation
- Environment-based configuration
- Error handling

You're now ready to handle configuration in real API projects!

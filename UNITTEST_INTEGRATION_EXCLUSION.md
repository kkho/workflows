# 🧪 Unit Test Action - Integration Test Exclusion

## ✅ **Updated Behavior:**

The `unittest/action.yaml` now automatically excludes test projects ending with `Integration.csproj` from unit test runs.

### **🔍 What Gets Excluded:**

```
✅ Will run unit tests:
- ProjectName.Tests.csproj
- ProjectName.UnitTests.csproj  
- ProjectName.Tests.Unit.csproj
- ProjectName.ComponentTests.csproj

🚫 Will exclude from unit tests:
- ProjectName.Integration.csproj
- ProjectName.Tests.Integration.csproj
- ProjectName.IntegrationTests.Integration.csproj
```

### **📋 Example Output:**

```
🔍 Searching for test projects matching pattern: **/*Tests/*.csproj
🚫 Excluding integration test projects (ending with Integration.csproj)
🚫 Excluding integration test: ./tests/Conferenti.Api.Tests.Integration.csproj
🚫 Excluding integration test: ./tests/Conferenti.Services.Integration.csproj
✅ Running unit tests on projects:
  📦 ./tests/Conferenti.Api.Tests.csproj
  📦 ./tests/Conferenti.Services.Tests.csproj
  📦 ./tests/Conferenti.Domain.UnitTests.csproj
🧪 Running tests for: ./tests/Conferenti.Api.Tests.csproj
🧪 Running tests for: ./tests/Conferenti.Services.Tests.csproj
🧪 Running tests for: ./tests/Conferenti.Domain.UnitTests.csproj
```

### **🎯 Usage Example:**

```yaml
- name: Run Unit Tests Only
  uses: ./unittest
  with:
    solution-path: './Conferenti.sln'
    test-projects: '**/*Tests/*.csproj'  # Finds all test projects
    collect-coverage: 'true'
    # Integration.csproj files are automatically excluded
```

### **🔧 How It Works:**

1. **🔍 Discovery**: Finds all projects matching the `test-projects` pattern
2. **🚫 Filtering**: Removes any project files ending with `Integration.csproj`
3. **📋 Logging**: Shows which projects are excluded and which will run
4. **🧪 Execution**: Runs tests only on the remaining unit test projects

### **🏗️ Project Structure Support:**

```
tests/
├── Conferenti.Api.Tests.csproj              ✅ Runs (unit tests)
├── Conferenti.Api.Tests.Integration.csproj  🚫 Excluded (integration)
├── Conferenti.Services.Tests.csproj         ✅ Runs (unit tests) 
├── Conferenti.Services.Integration.csproj   🚫 Excluded (integration)
└── Conferenti.Domain.UnitTests.csproj       ✅ Runs (unit tests)
```

### **💡 Benefits:**

- **⚡ Faster Builds**: Unit tests run separately from integration tests
- **🎯 Separation**: Clear distinction between unit and integration test execution
- **🔍 Visibility**: Clear logging of which projects are included/excluded
- **🛠️ Flexible**: Still uses the same `test-projects` pattern input

### **🔗 Integration Tests:**

Use the separate `integrationtest/action.yaml` for running integration tests:

```yaml
- name: Run Integration Tests
  uses: ./integrationtest
  with:
    system_name: "conferenti"
    test-projects: '**/*Integration*.csproj'
    environment: "test"
```

**Unit tests and integration tests are now properly separated!** 🚀
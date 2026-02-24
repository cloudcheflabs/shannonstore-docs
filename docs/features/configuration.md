# Configuration Management

ShannonStore supports flexible configuration through a priority-based resolution chain.

- **Priority order** (highest to lowest): System properties → Environment variables → External config file → Default classpath properties.
- Supports variable substitution (`${variable}` syntax) for DRY configuration.
- Over 150 configurable parameters covering cluster connectivity, storage, security, performance tuning, and more.

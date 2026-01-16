# Nyx Plugin: Masking

## Overview
NyxPluginMasking provides advanced data obfuscation and masking capabilities for sensitive information.

## Features
- Data anonymization
- Pattern-based masking
- Reversible masking (when authorized)

## Usage Example
```python
from olympitech.plugins import NyxMasking

masker = NyxMasking()
sensitive_data = "Sensitive information"
masked = masker.mask(sensitive_data)
```

## Configuration
```yaml
masking:
  patterns:
    - pattern: '\\d{4}-\\d{4}-\\d{4}-\\d{4}'
      mask_char: 'X'
    - pattern: '\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}\\b'
      mask_char: '*'
```

## Security Considerations
- Always use strong masking algorithms
- Store masking keys securely
- Audit access to masked data

# Contributing to OpenUMI

Thank you for your interest in contributing to OpenUMI! This document provides guidelines and information for contributors.

## Getting Started

1. Fork the repository
2. Create a feature branch from `main`
3. Make your changes
4. Submit a pull request

## Development Setup

### Prerequisites

- **Firmware**: ESP-IDF v5.x, Python 3.10+
- **iOS App**: Xcode 15+, iOS 16.0+ device
- **PCB Design**: KiCad 8
- **Mechanical Design**: FreeCAD 0.21+
- **Data Processing**: Python 3.10+, VINS-Fusion

### MCP Tools (Optional)

This project uses AI-assisted development. See [docs/design/system-design.md](docs/design/system-design.md#7-ai-driven-development-toolchain) for MCP server setup.

## Project Structure

```
openumi/
├── firmware/          # ESP-IDF firmware for ESP32-S3
├── ios-app/           # SwiftUI iOS application
├── hardware/
│   ├── pcb/           # KiCad PCB design files
│   └── mechanical/    # FreeCAD 3D model files
├── scripts/           # Data processing and conversion scripts
└── docs/
    └── design/        # Design specifications
```

## Contribution Areas

### Hardware
- PCB layout improvements
- Mechanical design iterations
- Component alternatives and optimizations
- Assembly documentation

### Firmware
- ESP-IDF sensor drivers
- WiFi streaming optimization
- Power management
- New device configurations

### iOS App
- UI/UX improvements
- Network reliability
- Data export features

### Data Processing
- VIO pipeline improvements
- LeRobot format conversion
- Calibration tools

## Code Style

- **C (firmware)**: Follow ESP-IDF coding style
- **Swift (iOS)**: Follow Swift API Design Guidelines
- **Python (scripts)**: Follow PEP 8, use type hints

## Pull Request Process

1. Ensure your changes build without errors
2. Update documentation if you changed interfaces or behavior
3. Add a clear description of what your PR does and why
4. Reference any related issues

## Reporting Issues

- Use GitHub Issues for bug reports and feature requests
- Include device/OS version, steps to reproduce, and expected vs actual behavior
- For hardware issues, include photos if relevant

## License

By contributing to OpenUMI, you agree that your contributions will be licensed under the [CC BY-NC-SA 4.0](LICENSE) license.

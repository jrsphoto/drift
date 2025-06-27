# Contributing to DRIFT

Thank you for your interest in contributing to DRIFT (Distributed RF Intelligence Network)! This guide will help you get started with contributing to the project.

## Table of Contents

- [Getting Started](#getting-started)
- [How to Contribute](#how-to-contribute)
- [Development Setup](#development-setup)
- [Coding Standards](#coding-standards)
- [Testing Guidelines](#testing-guidelines)
- [Documentation](#documentation)
- [Community Guidelines](#community-guidelines)
- [Recognition](#recognition)

## Getting Started

### Before You Begin

DRIFT is currently in the planning and early development phase. We welcome contributions in several areas:

- **Architecture Design**: Help refine system architecture and component interfaces
- **Hardware Integration**: Expertise with SDR hardware and SoapySDR
- **Core Development**: Python backend development
- **Frontend Development**: Web dashboard and visualization
- **DevOps**: Containerization, deployment automation, monitoring
- **Documentation**: User guides, API documentation, tutorials
- **Testing**: Test automation, hardware testing, integration testing

### Ways to Contribute

1. **Code Contributions**: Bug fixes, new features, optimizations
2. **Documentation**: Improve existing docs or create new guides
3. **Testing**: Write tests, report bugs, test on different hardware
4. **Design**: UI/UX improvements, system architecture feedback
5. **Hardware Support**: Add support for new SDR devices
6. **Community**: Help other users, participate in discussions

## How to Contribute

### 1. Find Something to Work On

- **Browse Issues**: Check [GitHub Issues](https://github.com/your-username/drift/issues) for open tasks
- **Good First Issues**: Look for issues labeled `good first issue` for newcomers
- **Feature Requests**: Check issues labeled `enhancement` for new features
- **Bug Reports**: Issues labeled `bug` need investigation and fixes
- **Documentation**: Issues labeled `documentation` need writing help

### 2. Discuss Your Idea

Before starting work on significant changes:

- **Open an Issue**: Describe what you'd like to work on
- **Join Discussions**: Participate in [GitHub Discussions](https://github.com/your-username/drift/discussions)
- **Ask Questions**: Don't hesitate to ask for clarification or guidance

### 3. Fork and Create a Branch

```bash
# Fork the repository on GitHub, then clone your fork
git clone https://github.com/your-username/drift.git
cd drift

# Add the upstream repository
git remote add upstream https://github.com/original-username/drift.git

# Create a new branch for your changes
git checkout -b feature/your-feature-name
```

### 4. Make Your Changes

- Follow the [coding standards](#coding-standards)
- Write tests for new functionality
- Update documentation as needed
- Keep commits focused and atomic

### 5. Test Your Changes

```bash
# Run the test suite
python -m pytest tests/

# Run linting
flake8 src/
black --check src/

# Test with different hardware (if applicable)
python scripts/test-hardware.py
```

### 6. Submit a Pull Request

- Push your changes to your fork
- Create a pull request against the main repository
- Provide a clear description of your changes
- Link any related issues
- Be responsive to code review feedback

## Development Setup

### Prerequisites

- Python 3.8 or higher
- Git
- Docker (for containerized testing)
- SDR hardware (for hardware-specific development)

### Environment Setup

```bash
# Clone the repository
git clone https://github.com/your-username/drift.git
cd drift

# Create a virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install development dependencies
pip install -r requirements-dev.txt

# Install DRIFT in development mode
pip install -e .

# Set up pre-commit hooks (optional but recommended)
pre-commit install
```

### Hardware Setup (Optional)

For testing with actual SDR hardware:

```bash
# Install SoapySDR and device drivers
# Ubuntu/Debian:
sudo apt-get install libsoapysdr-dev soapysdr-tools

# Install device-specific drivers
sudo apt-get install soapysdr-module-hackrf
sudo apt-get install soapysdr-module-rtlsdr
# ... other drivers as needed

# Test your setup
SoapySDRUtil --find
```

## Coding Standards

### Python Style Guide

We follow [PEP 8](https://pep8.org/) with some specific guidelines:

- **Line Length**: 88 characters (Black formatter default)
- **Imports**: Use absolute imports, group standard/third-party/local
- **Type Hints**: Use type hints for all public functions
- **Docstrings**: Use Google-style docstrings

### Code Formatting

We use automated code formatting:

```bash
# Format code with Black
black src/ tests/

# Sort imports with isort
isort src/ tests/

# Check style with flake8
flake8 src/ tests/
```

### Example Code Style

```python
from typing import Dict, List, Optional
import logging

from soapysdr import Device
from drift.core.exceptions import DeviceError


class SDRDevice:
    """Represents a Software Defined Radio device.
    
    This class provides a unified interface for different SDR hardware
    through the SoapySDR abstraction layer.
    
    Args:
        device_args: SoapySDR device arguments
        
    Raises:
        DeviceError: If device cannot be initialized
    """
    
    def __init__(self, device_args: Dict[str, str]) -> None:
        self._device: Optional[Device] = None
        self._logger = logging.getLogger(__name__)
        
        try:
            self._device = Device(device_args)
        except Exception as e:
            raise DeviceError(f"Failed to initialize device: {e}")
    
    def get_frequency_range(self, channel: int = 0) -> List[float]:
        """Get the supported frequency range for a channel.
        
        Args:
            channel: Channel index (default: 0)
            
        Returns:
            List of [min_freq, max_freq] in Hz
        """
        if not self._device:
            raise DeviceError("Device not initialized")
            
        return self._device.getFrequencyRange("RX", channel)
```

## Testing Guidelines

### Test Structure

```
tests/
├── unit/           # Unit tests for individual components
├── integration/    # Integration tests across components
├── hardware/       # Hardware-specific tests
├── fixtures/       # Test data and fixtures
└── conftest.py     # Pytest configuration
```

### Writing Tests

```python
import pytest
from unittest.mock import Mock, patch

from drift.core.device import SDRDevice
from drift.core.exceptions import DeviceError


class TestSDRDevice:
    """Test cases for SDRDevice class."""
    
    def test_device_initialization_success(self):
        """Test successful device initialization."""
        device_args = {"driver": "rtlsdr"}
        
        with patch('soapysdr.Device') as mock_device:
            device = SDRDevice(device_args)
            assert device is not None
            mock_device.assert_called_once_with(device_args)
    
    def test_device_initialization_failure(self):
        """Test device initialization failure handling."""
        with patch('soapysdr.Device', side_effect=RuntimeError("Device not found")):
            with pytest.raises(DeviceError):
                SDRDevice({"driver": "invalid"})
    
    @pytest.mark.hardware
    def test_real_hardware_detection(self):
        """Test with real hardware (requires SDR connected)."""
        # This test runs only when --hardware flag is used
        devices = SDRDevice.discover_devices()
        assert len(devices) > 0
```

### Running Tests

```bash
# Run all tests
pytest

# Run only unit tests
pytest tests/unit/

# Run with coverage
pytest --cov=drift

# Run hardware tests (requires actual SDR hardware)
pytest --hardware tests/hardware/

# Run specific test
pytest tests/unit/test_device.py::TestSDRDevice::test_device_initialization_success
```

## Documentation

### Documentation Types

1. **API Documentation**: Docstrings in code (auto-generated with Sphinx)
2. **User Guides**: Step-by-step tutorials in `docs/guides/`
3. **Technical Documentation**: Architecture and design docs in `docs/technical/`
4. **README Files**: Project overview and quick start

### Writing Documentation

- Use clear, concise language
- Include code examples where helpful
- Keep documentation up-to-date with code changes
- Use proper Markdown formatting
- Include diagrams for complex concepts

### Building

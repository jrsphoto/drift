# DRIFT - Distributed RF Intelligence Network

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![Status](https://img.shields.io/badge/status-planning-orange.svg)](https://github.com/your-username/drift)

## What is DRIFT?

DRIFT (Distributed RF Intelligence Network) is an open-source framework for coordinating multiple Software Defined Radio (SDR) devices across geographic locations to function as a unified, intelligent RF sensing and analysis system.

Think of it as "Kubernetes for SDRs" - where you can dynamically allocate radio resources across a distributed network, perform coordinated spectrum analysis, and execute sophisticated RF measurement campaigns that would be impossible with a single radio.

## Key Features (Planned)

- **Dynamic Resource Allocation**: Treat distributed SDRs as a pool of resources that can be allocated to jobs on demand
- **Coherent Operation**: Synchronize multiple radios using GPS disciplined oscillators or precision timing
- **Hardware Agnostic**: Support multiple SDR platforms through unified APIs
- **Geographic Distribution**: Coordinate radios across different locations for wide-area coverage
- **Application Agnostic**: Framework supports various RF applications from spectrum monitoring to propagation studies
- **Web-based Coordination**: Centralized scheduling and monitoring interface
- **Automated Deployment**: Infrastructure-as-code approach for easy node deployment

## Use Cases

- **Distributed Spectrum Monitoring**: Coordinate spectrum scans across multiple geographic locations
- **Propagation Studies**: Synchronized transmit/receive testing across wide areas
- **Interference Hunting**: Triangulate interference sources using multiple receivers
- **Band Planning**: Analyze spectrum usage patterns across different regions
- **Emergency Communications**: Rapid deployment of RF monitoring networks
- **Research & Development**: Academic and commercial RF research requiring distributed measurements

## Architecture Overview

DRIFT follows a client-server architecture with three main components:

### Coordinator (Server)
- Job scheduling and resource allocation
- Node health monitoring and discovery
- Data aggregation and storage
- Web dashboard for monitoring and control
- RESTful API for job submission

### Node Agents (Clients)
- Local SDR device management
- Job execution and data collection
- Clock synchronization management
- Hardware abstraction layer
- Automated deployment and updates

### Synchronization Layer
- GPS disciplined oscillator support
- Precision Time Protocol (PTP) coordination
- Software-based frequency correction
- Phase coherence management

## Quick Start

*Note: DRIFT is currently in the planning phase. This section will be updated as development progresses.*

```bash
# Clone the repository
git clone https://github.com/your-username/drift.git
cd drift

# Install dependencies
pip install -r requirements.txt

# Deploy a node
./deploy-node.sh --config node-config.yaml

# Start the coordinator
python drift-coordinator.py --config coordinator-config.yaml
```

## Hardware Support

DRIFT aims to support a wide range of SDR hardware through the SoapySDR abstraction layer:

- **HackRF One**: Wideband coverage, fast spectrum sweeping
- **ADALM-Pluto**: High-resolution narrowband analysis
- **Airspy Series**: High-performance receivers
- **USRP Series**: Research-grade precision
- **RTL-SDR**: Cost-effective wide deployment
- **HermesLite 2**: HF coverage with transmit capabilities
- **BladeRF**: High-performance transceiver
- **And many more via SoapySDR**

## Project Status

🚧 **Currently in Planning Phase** 🚧

We are actively designing the architecture and gathering requirements. See our [Project Roadmap](ROADMAP.md) for current progress and upcoming milestones.

## Contributing

We welcome contributions from the SDR and RF community! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on how to get involved.

## Inspiration

DRIFT draws inspiration from several excellent open-source projects:
- [SatNOGS](https://satnogs.org/) - Distributed satellite ground station network
- [KrakenSDR](https://github.com/krakenrf/krakensdr_doa) - Coherent multi-channel SDR processing
- [REDHAWK](https://github.com/redhawksdr/redhawk) - Software-defined radio framework
- [GNU Radio](https://www.gnuradio.org/) - Signal processing framework

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Contact

- **Project Lead**: [Your Name]
- **Discussion**: [GitHub Discussions](https://github.com/your-username/drift/discussions)
- **Issues**: [GitHub Issues](https://github.com/your-username/drift/issues)

## Acknowledgments

Special thanks to the open-source SDR community and the projects that inspire DRIFT's development.

---

*"Enabling distributed RF intelligence, one node at a time."*

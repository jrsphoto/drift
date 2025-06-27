# DRIFT Architecture

This document outlines the system architecture for DRIFT (Distributed RF Intelligence Network).

## System Overview

DRIFT implements a distributed computing model where Software Defined Radio (SDR) devices are treated as compute resources that can be dynamically allocated to RF processing jobs. The system is designed to scale from a few local nodes to hundreds of geographically distributed radios.

## Core Architecture

### Three-Tier Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    COORDINATOR LAYER                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │ Job Scheduler   │  │ Resource Manager│  │ Web Dashboard│ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │ Data Aggregator │  │ Node Discovery  │  │ REST API     │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────────┘
                                │
                          ┌─────┴─────┐
                          │ SYNC PLANE│
                          └─────┬─────┘
                                │
┌─────────────────────────────────────────────────────────────┐
│                      NODE AGENT LAYER                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │ Task Executor   │  │ Device Manager  │  │ Sync Manager │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │ Data Collector  │  │ Health Monitor  │  │ Config Mgmt  │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└─────────────────────────────────────────────────────────────┘
                                │
┌─────────────────────────────────────────────────────────────┐
│                    HARDWARE LAYER                          │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐ │
│  │ SDR Devices  │  │ Clock Sync   │  │ Compute Resources   │ │
│  │ (SoapySDR)   │  │ (GPS/PTP)    │  │ (CPU/GPU/Storage)   │ │
│  └──────────────┘  └──────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Component Details

### Coordinator Layer

The coordinator runs as a centralized service responsible for orchestrating the entire network.

#### Job Scheduler
- Receives job requests via REST API or web interface
- Analyzes resource requirements (frequency coverage, geographic distribution, timing)
- Selects optimal nodes and devices for job execution
- Handles job queuing, priority management, and conflict resolution
- Monitors job progress and handles failures

#### Resource Manager
- Maintains real-time inventory of available SDR devices across all nodes
- Tracks device capabilities, current status, and health metrics
- Implements resource allocation algorithms
- Handles device reservations and releases
- Manages resource contention and load balancing

#### Node Discovery Service
- Automatically discovers new nodes joining the network
- Maintains node registry with capabilities and status
- Handles node authentication and authorization
- Monitors node health and connectivity
- Implements node heartbeat and failure detection

### Node Agent Layer

Each node runs an agent responsible for local device management and job execution.

#### Device Manager
- Abstracts hardware through SoapySDR interface
- Discovers and initializes local SDR devices
- Manages device configurations and state
- Implements device health monitoring
- Handles device-specific optimizations

#### Task Executor
- Receives job assignments from coordinator
- Configures devices according to job parameters
- Executes spectrum scans, data collection, or signal processing
- Handles multi-device coordination for complex jobs
- Reports job progress and results

#### Synchronization Manager
- Manages local clock synchronization (GPS/PTP)
- Coordinates with other nodes for coherent operation
- Handles frequency reference distribution
- Implements phase alignment algorithms
- Monitors synchronization quality

### Data Flow Architecture

```
Job Submission → Resource Allocation → Node Assignment → 
Task Execution → Data Collection → Aggregation → Results
```

#### Data Collection Pipeline
1. **Local Collection**: Node agents collect raw RF data
2. **Local Processing**: Initial filtering, FFT, or feature extraction
3. **Data Packaging**: Format data with metadata (SigMF standard)
4. **Transmission**: Send processed data to coordinator
5. **Aggregation**: Coordinator combines data from multiple nodes
6. **Storage**: Persistent storage with indexing for analysis
7. **Visualization**: Web dashboard or API access to results

## Synchronization Architecture

### Clock Distribution
- **Primary Reference**: GPS disciplined oscillator (GPSDO)
- **Local Distribution**: 10MHz reference to all SDR devices
- **Software Correction**: NTP/PTP for timestamp synchronization
- **Quality Monitoring**: Continuous sync quality assessment

### Coherent Operation Modes
1. **Frequency Coherent**: Common 10MHz reference
2. **Time Coherent**: Synchronized sampling start times
3. **Phase Coherent**: Additional phase alignment (advanced)

## Network Architecture

### Communication Protocols
- **Control Plane**: HTTPS/WebSocket for real-time coordination
- **Data Plane**: Configurable (TCP/UDP/ZeroMQ) based on requirements
- **Management Plane**: SSH/Ansible for deployment and maintenance

### Security Model
- **Authentication**: Certificate-based node authentication
- **Authorization**: Role-based access control
- **Encryption**: TLS for all communications
- **Network Isolation**: VPN recommended for distributed deployments

## Deployment Architecture

### Containerized Deployment
```
┌─────────────────────────────────────────┐
│              Docker Host                │
│  ┌─────────────────────────────────────┐ │
│  │          Node Agent                 │ │
│  │  ┌─────────────┐ ┌─────────────────┐ │ │
│  │  │Device Mgr   │ │ Sync Manager    │ │ │
│  │  └─────────────┘ └─────────────────┘ │ │
│  └─────────────────────────────────────┘ │
│  ┌─────────────────────────────────────┐ │
│  │        Supporting Services          │ │
│  │  ┌─────────────┐ ┌─────────────────┐ │ │
│  │  │Monitoring   │ │ Log Aggregation │ │ │
│  │  └─────────────┘ └─────────────────┘ │ │
│  └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

### Infrastructure as Code
- **Ansible Playbooks**: Automated node deployment and configuration
- **Docker Compose**: Service orchestration for complex nodes
- **Configuration Management**: Centralized config with local overrides
- **Monitoring Stack**: Prometheus/Grafana for system metrics

## Scalability Considerations

### Horizontal Scaling
- **Node Scaling**: Add nodes linearly without coordinator changes
- **Device Scaling**: Multiple devices per node with automatic discovery
- **Geographic Scaling**: Region-aware job scheduling
- **Load Distribution**: Intelligent job distribution based on node capabilities

### Performance Optimization
- **Local Processing**: Minimize data transmission through edge processing
- **Caching**: Cache frequently accessed spectrum data
- **Compression**: Efficient data compression for network transmission
- **Parallel Processing**: Multi-threaded job execution on capable nodes

## Technology Stack

### Core Technologies
- **Language**: Python 3.8+ for primary implementation
- **SDR Abstraction**: SoapySDR for hardware interface
- **Web Framework**: FastAPI for REST API and web interface
- **Database**: PostgreSQL for metadata, InfluxDB for time-series data
- **Message Queue**: Redis for job queuing and caching
- **Containerization**: Docker for deployment
- **Orchestration**: Docker Compose (simple) or Kubernetes (advanced)

### Supporting Technologies
- **Monitoring**: Prometheus + Grafana
- **Logging**: ELK Stack (Elasticsearch, Logstash, Kibana)
- **Configuration**: YAML-based configuration files
- **Documentation**: Sphinx for API documentation
- **Testing**: pytest for unit and integration testing

## Future Architecture Considerations

### Advanced Features
- **Machine Learning Integration**: Automated interference detection and classification
- **Real-time Processing**: Stream processing for time-critical applications
- **Edge Computing**: Local AI processing on powerful nodes
- **Mesh Networking**: Peer-to-peer coordination for resilient operation
- **Cloud Integration**: Hybrid cloud/edge deployment models

### Research Directions
- **Adaptive Sampling**: Dynamic adjustment of sampling parameters
- **Predictive Scheduling**: ML-based job scheduling optimization
- **Federated Learning**: Distributed learning across nodes
- **Quantum-Safe Security**: Future-proof cryptographic protocols

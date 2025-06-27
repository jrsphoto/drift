# DRIFT Technical Overview

This document provides a detailed technical overview of the DRIFT (Distributed RF Intelligence Network) system, including implementation details, technology choices, and integration strategies.

## Technology Stack

### Core Languages and Frameworks

**Python 3.8+** - Primary implementation language
- **Rationale**: Excellent SDR ecosystem, scientific computing libraries, and rapid development
- **Libraries**: SciPy/NumPy for DSP, AsyncIO for concurrency, Pydantic for data validation

**FastAPI** - Web framework for REST API and web interface
- **Rationale**: High performance, automatic API documentation, excellent async support
- **Features**: WebSocket support for real-time updates, dependency injection, type validation

**PostgreSQL** - Primary database for metadata and configuration
- **Rationale**: ACID compliance, JSON support, excellent performance
- **Usage**: Node registration, job scheduling, user management, system configuration

**InfluxDB** - Time-series database for spectrum data
- **Rationale**: Optimized for time-series data, excellent compression, powerful query language
- **Usage**: Spectrum measurements, device metrics, performance data

**Redis** - Caching and message queuing
- **Rationale**: High performance, pub/sub messaging, distributed locking
- **Usage**: Job queues, caching, real-time coordination between services

### SDR Integration Layer

**SoapySDR** - Hardware abstraction layer
- **Rationale**: Unified API across hardware vendors, extensive device support, network transparency
- **Integration**: Python bindings, custom device modules, remote device access

**Supported Hardware Matrix**:

| Device | Frequency Range | Max Bandwidth | TX Capable | Sync Support | Status |
|--------|----------------|---------------|------------|--------------|---------|
| HackRF One | 1MHz - 6GHz | 20MHz | Yes | 10MHz Ref | ✅ Tier 1 |
| ADALM-Pluto | 325MHz - 3.8GHz | 56MHz | Yes | External | ✅ Tier 1 |
| RTL-SDR v3 | 24MHz - 1.7GHz | 2.4MHz | No | Crystal | ✅ Tier 1 |
| Airspy R2 | 24MHz - 1.8GHz | 10MHz | No | 10MHz Ref | 🔄 Tier 2 |
| HermesLite 2 | 10kHz - 30MHz | 768kHz | Yes (5W) | GPS | 🔄 Tier 2 |
| USRP B210 | 70MHz - 6GHz | 56MHz | Yes | GPSDO | 📋 Planned |

### Containerization and Deployment

**Docker** - Application containerization
- **Base Images**: Multi-stage builds for minimal production images
- **Security**: Non-root execution, minimal attack surface
- **Hardware Access**: USB device pass-through, privileged access management

**Ansible** - Infrastructure automation
- **Rationale**: Agentless deployment, idempotent operations, extensive module ecosystem
- **Usage**: Node deployment, configuration management, updates, monitoring setup

## System Architecture Details

### Node Agent Implementation

```python
# Simplified node agent structure
class NodeAgent:
    def __init__(self, config: NodeConfig):
        self.device_manager = DeviceManager(config.devices)
        self.task_executor = TaskExecutor()
        self.sync_manager = SynchronizationManager(config.sync)
        self.coordinator_client = CoordinatorClient(config.coordinator)
        self.health_monitor = HealthMonitor()
    
    async def start(self):
        await self.device_manager.initialize()
        await self.register_with_coordinator()
        await self.start_task_loop()
    
    async def register_with_coordinator(self):
        capabilities = await self.device_manager.get_capabilities()
        registration = NodeRegistration(
            node_id=self.config.node_id,
            location=self.config.location,
            capabilities=capabilities,
            sync_quality=await self.sync_manager.get_quality()
        )
        await self.coordinator_client.register(registration)
```

### Device Abstraction Layer

**Unified Device Interface**:
```python
from abc import ABC, abstractmethod
from typing import Dict, List, Optional
from dataclasses import dataclass

@dataclass
class DeviceCapabilities:
    frequency_ranges: List[tuple[float, float]]
    sample_rates: List[float]
    gain_ranges: Dict[str, tuple[float, float]]
    has_transmit: bool
    sync_sources: List[str]
    max_bandwidth: float

class SDRDevice(ABC):
    @abstractmethod
    async def configure(self, config: DeviceConfig) -> None:
        """Configure device for operation."""
        pass
    
    @abstractmethod
    async def start_streaming(self) -> AsyncIterator[np.ndarray]:
        """Start streaming samples."""
        pass
    
    @abstractmethod
    async def get_capabilities(self) -> DeviceCapabilities:
        """Get device capabilities."""
        pass
    
    @abstractmethod
    async def set_sync_source(self, source: str) -> None:
        """Configure synchronization source."""
        pass
```

### Job Scheduling Architecture

**Resource Allocation Algorithm**:
```python
class ResourceAllocator:
    def __init__(self, resource_manager: ResourceManager):
        self.resource_manager = resource_manager
        self.allocation_strategies = {
            'spectrum_scan': self._allocate_for_spectrum_scan,
            'direction_finding': self._allocate_for_direction_finding,
            'propagation_test': self._allocate_for_propagation_test
        }
    
    async def allocate_resources(self, job: Job) -> ResourceAllocation:
        strategy = self.allocation_strategies[job.type]
        return await strategy(job)
    
    async def _allocate_for_spectrum_scan(self, job: SpectrumScanJob) -> ResourceAllocation:
        # Consider: frequency coverage, geographic distribution, 
        # device availability, sync requirements
        available_nodes = await self.resource_manager.get_available_nodes()
        
        # Filter nodes by frequency capability
        capable_nodes = [
            node for node in available_nodes 
            if self._can_cover_frequency_range(node, job.frequency_range)
        ]
        
        # Select optimal subset based on geographic distribution
        selected_nodes = self._select_geographically_distributed(
            capable_nodes, job.required_nodes
        )
        
        return ResourceAllocation(
            job_id=job.id,
            allocated_nodes=selected_nodes,
            allocation_time=datetime.utcnow()
        )
```

### Synchronization Implementation

**Clock Synchronization Strategy**:
```python
class SynchronizationManager:
    def __init__(self, config: SyncConfig):
        self.sync_sources = {
            'gps': GPSSynchronizer(),
            'ntp': NTPSynchronizer(),
            'ptp': PTPSynchronizer(),
            'manual': ManualSynchronizer()
        }
        self.active_source = config.primary_source
        self.fallback_sources = config.fallback_sources
    
    async def get_time_reference(self) -> TimeReference:
        """Get current time reference with uncertainty estimate."""
        try:
            return await self.sync_sources[self.active_source].get_time()
        except SyncError:
            # Fallback to secondary sources
            for fallback in self.fallback_sources:
                try:
                    return await self.sync_sources[fallback].get_time()
                except SyncError:
                    continue
            raise SyncError("No time reference available")
    
    async def synchronize_devices(self, devices: List[SDRDevice]) -> None:
        """Synchronize all devices to common time reference."""
        time_ref = await self.get_time_reference()
        
        # Configure 10MHz reference
        for device in devices:
            await device.set_frequency_reference(10e6)
        
        # Coordinate start times
        start_time = time_ref.time + timedelta(seconds=2)  # 2 second lead time
        for device in devices:
            await device.schedule_start(start_time)
```

### Data Pipeline Architecture

**Spectrum Data Processing**:
```python
class SpectrumProcessor:
    def __init__(self, config: ProcessingConfig):
        self.fft_size = config.fft_size
        self.overlap = config.overlap
        self.window = signal.get_window(config.window_type, self.fft_size)
        self.calibration = CalibrationManager(config.calibration)
    
    async def process_samples(self, samples: np.ndarray, metadata: SampleMetadata) -> SpectrumData:
        """Process raw samples into spectrum data."""
        # Apply calibration corrections
        calibrated_samples = await self.calibration.apply(samples, metadata)
        
        # Compute spectrum with overlap-save
        spectrum = self._compute_spectrum(calibrated_samples)
        
        # Apply additional processing (noise floor estimation, peak detection, etc.)
        processed_spectrum = await self._post_process(spectrum, metadata)
        
        return SpectrumData(
            frequency_bins=self._get_frequency_bins(metadata.center_freq, metadata.sample_rate),
            power_spectrum=processed_spectrum,
            timestamp=metadata.timestamp,
            node_id=metadata.node_id,
            device_id=metadata.device_id
        )
    
    def _compute_spectrum(self, samples: np.ndarray) -> np.ndarray:
        """Compute power spectral density using Welch's method."""
        f, psd = signal.welch(
            samples, 
            fs=self.sample_rate,
            window=self.window,
            nperseg=self.fft_size,
            noverlap=int(self.fft_size * self.overlap),
            return_onesided=False
        )
        return 10 * np.log10(psd + 1e-12)  # Convert to dB
```

### API Design

**RESTful API Structure**:
```yaml
# OpenAPI specification excerpt
paths:
  /api/v1/jobs:
    post:
      summary: Submit a new job
      requestBody:
        content:
          application/json:
            schema:
              oneOf:
                - $ref: '#/components/schemas/SpectrumScanJob'
                - $ref: '#/components/schemas/DirectionFindingJob'
                - $ref: '#/components/schemas/PropagationTestJob'
    get:
      summary: List jobs with filtering
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [pending, running, completed, failed]
        - name: node_id
          in: query
          schema:
            type: string

  /api/v1/nodes:
    get:
      summary: List registered nodes
      responses:
        '200':
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/NodeInfo'

  /api/v1/spectrum/{job_id}:
    get:
      summary: Get spectrum data for a job
      parameters:
        - name: start_time
          in: query
          schema:
            type: string
            format: date-time
        - name: end_time
          in: query
          schema:
            type: string
            format: date-time
```

### Security Implementation

**Authentication and Authorization**:
```python
class SecurityManager:
    def __init__(self, config: SecurityConfig):
        self.jwt_secret = config.jwt_secret
        self.cert_manager = CertificateManager(config.certs)
        self.rbac = RoleBasedAccessControl(config.roles)
    
    async def authenticate_node(self, certificate: str) -> NodeIdentity:
        """Authenticate node using client certificate."""
        cert = await self.cert_manager.validate_certificate(certificate)
        return NodeIdentity(
            node_id=cert.subject.common_name,
            permissions=await self.rbac.get_node_permissions(cert)
        )
    
    async def authorize_job_submission(self, user: User, job: Job) -> bool:
        """Check if user can submit this type of job."""
        required_permissions = self._get_job_permissions(job)
        return all(
            perm in user.permissions 
            for perm in required_permissions
        )

# Network security
class NetworkSecurity:
    @staticmethod
    def setup_tls_context() -> ssl.SSLContext:
        """Configure TLS for secure communications."""
        context = ssl.create_default_context(ssl.Purpose.SERVER_AUTH)
        context.minimum_version = ssl.TLSVersion.TLSv1_2
        context.set_ciphers('ECDHE+AESGCM:ECDHE+CHACHA20:DHE+AESGCM:DHE+CHACHA20:!aNULL:!MD5:!DSS')
        return context
```

### Monitoring and Observability

**Metrics Collection**:
```python
from prometheus_client import Counter, Histogram, Gauge

# Application metrics
job_counter = Counter('drift_jobs_total', 'Total jobs processed', ['status', 'type'])
job_duration = Histogram('drift_job_duration_seconds', 'Job execution time')
active_nodes = Gauge('drift_active_nodes', 'Number of active nodes')
spectrum_data_points = Counter('drift_spectrum_data_points_total', 'Spectrum data points collected')

class MetricsCollector:
    def __init__(self):
        self.device_metrics = {}
        self.performance_metrics = {}
    
    async def collect_device_metrics(self, device: SDRDevice) -> None:
        """Collect device-specific metrics."""
        metrics = await device.get_performance_metrics()
        self.device_metrics[device.id] = {
            'temperature': metrics.temperature,
            'gain_stability': metrics.gain_stability,
            'frequency_accuracy': metrics.frequency_accuracy,
            'sample_drops': metrics.sample_drops
        }
    
    async def export_to_prometheus(self) -> str:
        """Export metrics in Prometheus format."""
        # Implementation for Prometheus exposition format
        pass
```

### Error Handling and Recovery

**Resilience Patterns**:
```python
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

class ResilientCoordinator:
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=4, max=10)
    )
    async def send_job_to_node(self, node: Node, job: Job) -> JobResult:
        """Send job to node with automatic retry."""
        try:
            return await node.execute_job(job)
        except NodeUnreachableError:
            # Try to reschedule on different node
            alternative_node = await self.find_alternative_node(job.requirements)
            if alternative_node:
                return await alternative_node.execute_job(job)
            raise
    
    async def handle_node_failure(self, failed_node: Node) -> None:
        """Handle node failure and job reassignment."""
        # Mark node as failed
        await self.resource_manager.mark_node_failed(failed_node.id)
        
        # Reassign running jobs to other nodes
        running_jobs = await self.get_jobs_on_node(failed_node.id)
        for job in running_jobs:
            try:
                alternative_node = await self.find_alternative_node(job.requirements)
                await self.reassign_job(job, alternative_node)
            except NoAlternativeNodeError:
                await self.mark_job_failed(job.id, "Node failure, no alternative available")
```

### Performance Optimization

**Data Processing Optimization**:
```python
import numpy as np
from numba import jit, cuda

class OptimizedSpectrumProcessor:
    def __init__(self, use_gpu: bool = False):
        self.use_gpu = use_gpu
        if use_gpu:
            self.fft_func = self._gpu_fft
        else:
            self.fft_func = self._cpu_fft
    
    @jit(nopython=True)
    def _cpu_fft(self, samples: np.ndarray) -> np.ndarray:
        """CPU-optimized FFT processing."""
        return np.fft.fft(samples)
    
    @cuda.jit
    def _gpu_fft(self, samples: np.ndarray) -> np.ndarray:
        """GPU-accelerated FFT processing."""
        # CUDA implementation for high-throughput processing
        pass
    
    async def process_high_throughput(self, sample_stream: AsyncIterator[np.ndarray]) -> AsyncIterator[SpectrumData]:
        """Process high-throughput sample streams."""
        async for samples in sample_stream:
            # Process in parallel using thread pool
            spectrum = await asyncio.get_event_loop().run_in_executor(
                None, self.fft_func, samples
            )
            yield SpectrumData(spectrum=spectrum, timestamp=time.time())
```

## Integration Strategies

### SoapySDR Integration

**Custom Device Modules**:
```cpp
// Example custom SoapySDR module for specialized hardware
#include <SoapySDR/Device.hpp>
#include <SoapySDR/Registry.hpp>

class CustomSDRDevice : public SoapySDR::Device {
public:
    CustomSDRDevice(const SoapySDR::Kwargs &args);
    
    // Implement required SoapySDR interface
    std::vector<std::string> listAntennas(const int direction, const size_t channel) const override;
    void setFrequency(const int direction, const size_t channel, const double frequency, const SoapySDR::Kwargs &args) override;
    // ... other required methods
    
    // Custom synchronization methods
    void setSyncSource(const std::string &source);
    double getSyncQuality() const;
};

// Registration
static SoapySDR::Registry registerCustomSDR("custom", &make_CustomSDRDevice);
```

### Database Schema Design

**PostgreSQL Schema**:
```sql
-- Nodes table
CREATE TABLE nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    location GEOGRAPHY(POINT),
    last_seen TIMESTAMP WITH TIME ZONE,
    status VARCHAR(50) DEFAULT 'offline',
    capabilities JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Jobs table
CREATE TABLE jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type VARCHAR(100) NOT NULL,
    parameters JSONB NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    priority INTEGER DEFAULT 0,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    allocated_nodes UUID[] DEFAULT '{}'
);

-- Devices table
CREATE TABLE devices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id UUID REFERENCES nodes(id),
    driver VARCHAR(100) NOT NULL,
    serial_number VARCHAR(255),
    capabilities JSONB,
    calibration_data JSONB,
    last_calibrated TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_jobs_status ON jobs(status);
CREATE INDEX idx_jobs_created_at ON jobs(created_at);
CREATE INDEX idx_nodes_location ON nodes USING GIST(location);
```

**InfluxDB Schema**:
```sql
-- Spectrum measurements
CREATE MEASUREMENT spectrum_data (
    time TIMESTAMP,
    node_id STRING,
    device_id STRING,
    center_frequency FLOAT,
    sample_rate FLOAT,
    frequency_bins FLOAT[],
    power_spectrum FLOAT[],
    job_id STRING
);

-- Device metrics
CREATE MEASUREMENT device_metrics (
    time TIMESTAMP,
    node_id STRING,
    device_id STRING,
    temperature FLOAT,
    sample_rate FLOAT,
    dropped_samples INTEGER
);
```

This technical overview provides the foundation for implementing DRIFT's core functionality while maintaining flexibility for future enhancements and community contributions.

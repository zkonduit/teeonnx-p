# teeonnx - Prebuilt Binaries and Containers

This repository provides binaries and Docker containers for teeonnx, a hybrid Trusted Execution Environment (TEE) approach for zero-knowledge proof witness generation. Run ML model inference inside Intel SGX enclaves with DCAP attestation supporting both traditional zkp workflows and TEE-based verification.

## Overview

teeonnx extends the standard workflow by moving witness generation into a secure Intel SGX enclave. The TEE approach provides cryptographic guarantees about the execution environment and computation integrity using tract for ONNX inference.

### Key Benefits

- **Verifiable Computation**: SGX enclave provides hardware-backed proof of correct execution
- **Hybrid zkp Support**: Generated witnesses work with standard proving systems, while DCAP quotes enable TEE-based verification
- **Cryptographic Binding**: keccak256 hashes in the DCAP quote cryptographically bind input, circuit, and witness
- **Compressed Verification**: RISC0 zkVM creates succinct proofs of quote validity for efficient on-chain verification

## Architecture

The system operates as a stateless "pure function" that verifiably runs inference:

```
┌─────────────┐    ┌─────────────────────┐    ┌──────────────┐
│  input.json │───▶│   SGX Enclave       │───▶│ output.json  │
│ circuit.bin │    │                     │    │  quote.bin   │
└─────────────┘    │ 1. Hash inputs      │    └──────────────┘
                   │ 2. Generate output  │
                   │ 3. Hash outputs     │
                   │ 4. Create DCAP quote│
                   └─────────────────────┘
                            │
                            ▼
                   ┌─────────────────────┐
                   │   zk                │
                   │ Quote Verification  │
                   │ → Proof             │
                   └─────────────────────┘
```

## Getting Started

### Prerequisites

**SGX-Enabled Hardware**: You need a machine with Intel SGX support. For cloud deployment, we recommend Azure DCsv3 instances.

**SGX Runtime Installation** (Ubuntu 22.04):
```bash
# Install SGX runtime libraries
echo 'deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu jammy main' | sudo tee /etc/apt/sources.list.d/intel-sgx.list
wget -qO - https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | sudo apt-key add -
sudo apt-get update
sudo apt-get install libsgx-urts libsgx-dcap-ql
```

### Using Docker Container (Recommended)

The easiest way to get started is with the prebuilt Docker container:

```bash
# Pull the latest SGX container
docker pull ghcr.io/zkonduit/teeonnx-sgx:latest

# Run inference in the SGX enclave
docker run --device /dev/sgx_enclave --device /dev/sgx_provision \
  -v $(pwd):/workspace \
  ghcr.io/zkonduit/teeonnx-sgx:latest \
  gen-output \
  --input /workspace/input.json \
  --model /workspace/network.onnx \
  --output /workspace/output.json \
  --quote /workspace/quote.bin
```

**Required Docker Arguments:**
- `--device /dev/sgx_enclave`: Access to SGX enclave device
- `--device /dev/sgx_provision`: Access to SGX provisioning service
- `-v $(pwd):/workspace`: Mount your working directory

### Using Prebuilt Binaries

Download the appropriate binary for your system from the [releases page](https://github.com/zkonduit/teeonnx-p/releases):

**CPU-only verification binary:**
- `teeonnx-zk-cpu-linux`: Works on any x86_64 Linux system

**CUDA-enabled verification binaries (faster proving):**
- `teeonnx-zk-cuda-linux-sm70`: Tesla V100, GTX 1080 Ti
- `teeonnx-zk-cuda-linux-sm75`: RTX 2080, RTX 2080 Ti, Tesla T4
- `teeonnx-zk-cuda-linux-sm80`: RTX 3080, RTX 3090, A100
- `teeonnx-zk-cuda-linux-sm86`: RTX 3050, RTX 3060, RTX 3070
- `teeonnx-zk-cuda-linux-sm89`: RTX 4090, RTX 4080
- `teeonnx-zk-cuda-linux-sm90`: H100
- `teeonnx-zk-cuda-linux-sm100`: Future architecture support
- `teeonnx-zk-cuda-linux-sm100a`: Future architecture support
- `teeonnx-zk-cuda-linux-sm120`: Future architecture support
- `teeonnx-zk-cuda-linux-sm120a`: Future architecture support

```bash
# Download CPU-only binary
wget https://github.com/zkonduit/teeonnx-p/releases/latest/download/teeonnx-zk-cpu-linux
chmod +x teeonnx-zk-cpu-linux

# Or download CUDA binary for your GPU architecture (example for RTX 3080/3090/A100)
wget https://github.com/zkonduit/teeonnx-p/releases/latest/download/teeonnx-zk-cuda-linux-sm80
chmod +x teeonnx-zk-cuda-linux-sm80
```

**Find your GPU's compute capability:**
```bash
# Check your GPU model
nvidia-smi

# Or use this command to get compute capability directly
nvidia-smi --query-gpu=compute_cap --format=csv,noheader,nounits
```

## Basic Usage

### 1. Generate Output in SGX Enclave

Using Docker (recommended):
```bash
docker run --device /dev/sgx_enclave --device /dev/sgx_provision \
  -v $(pwd):/workspace \
  ghcr.io/zkonduit/teeonnx-sgx:latest \
  gen-output \
  --input /workspace/input.json \
  --model /workspace/network.onnx \
  --output /workspace/output.json \
  --quote /workspace/quote.bin
```

This command:
- Loads your input and ONNX model into the SGX enclave
- Uses tract for ONNX inference to generate output
- Creates a DCAP quote with computation hashes
- Outputs both result and quote files

### 2. Generate Proof of Quote Validity

```bash
# Using CPU binary
./teeonnx-zk-cpu-linux prove \
  --quote quote.bin \
  --proof proof.json

# Using CUDA binary (faster)
./teeonnx-zk-cuda-linux-sm80 prove \
  --quote quote.bin \
  --proof proof.json
```

Creates a proof that the DCAP quote is valid.

### 3. Verify the Proof

```bash
./teeonnx-zk-cpu-linux verify --proof proof.json
```

You can also verify with additional hash checks:

```bash
./teeonnx-zk-cpu-linux verify --proof proof.json \
    --input-hash "INPUT_HASH_HEX" \
    --model-hash "MODEL_HASH_HEX" \
    --output-hash "OUTPUT_HASH_HEX"
```

### 4. Verify Hash Bindings

```bash
./teeonnx-zk-cpu-linux hash-check \
  --input input.json \
  --model network.onnx \
  --output output.json \
  --quote quote.bin
```

Verifies that the quote's user_data contains the correct hashes of your computation.

### 5. Check MRENCLAVE

```bash
./teeonnx-zk-cpu-linux mrenclave-check \
  --quote quote.bin \
  --mrenclave "EXPECTED_MRENCLAVE_HEX"
```

## Complete Verification Workflow

The full verification process involves three checks:
 
1. **Quote Validity**: Verify the DCAP quote is cryptographically valid (via RISC0 proof) and checks the integrity of the enclave execution
2. **(optional) MRENCLAVE Check**: Confirm the quote was generated by the expected enclave code
3. **(optional) Hash Verification**: Ensure the quote's user_data matches your input/model/output hashes

Example verification script:
```bash
#!/bin/bash
# Complete verification workflow

# Download verification binary
wget https://github.com/zkonduit/teeonnx-p/releases/latest/download/teeonnx-zk-cpu-linux
chmod +x teeonnx-zk-cpu-linux

# 1. Verify RISC0 proof of quote validity
./teeonnx-zk-cpu-linux verify --proof proof.json

# 2. Check MRENCLAVE matches expected value
./teeonnx-zk-cpu-linux mrenclave-check --quote quote.bin --mrenclave "$EXPECTED_MRENCLAVE"

# 3. Verify hash bindings
./teeonnx-zk-cpu-linux hash-check \
  --input input.json \
  --model network.onnx \
  --output output.json \
  --quote quote.bin

# Alternative: Verify proof with hash checks in one command
./teeonnx-zk-cpu-linux verify --proof proof.json \
  --input-hash "INPUT_HASH_HEX" \
  --model-hash "MODEL_HASH_HEX" \
  --output-hash "OUTPUT_HASH_HEX"

echo "✅ All verification checks passed!"
```

## Input Format

### Input JSON Format
```json
{
  "data": [
    [1.0, 2.0, 3.0, 4.0, 5.0, 6.0]
  ],
  "shapes": [
    [2, 3]
  ],
}
```

### Expected Output
```json
{
  "data": [
    [0.0, 0.0, 0.0]
  ],
  "shapes": [
    [1, 3]
  ],
}
```

- `output.json`: Inference results from the ONNX model
- `quote.bin`: DCAP quote containing cryptographic attestation
- `proof.json`: RISC0 Groth16 proof of quote validity

## Docker Containers for Proving

### CPU Proving Container

```bash
# Pull and run CPU proving container
docker pull ghcr.io/zkonduit/teeonnx-cpu:latest

# Generate proof using CPU
docker run -v $(pwd):/workspace ghcr.io/zkonduit/teeonnx-cpu:latest \
  prove --quote /workspace/quote.bin --proof /workspace/proof.json

# Verify proof
docker run -v $(pwd):/workspace ghcr.io/zkonduit/teeonnx-cpu:latest \
  verify --proof /workspace/proof.json
```

### GPU Proving Container (Requires NVIDIA GPU)

**Prerequisites:**
- NVIDIA GPU with CUDA support
- NVIDIA Container Toolkit installed

**Install NVIDIA Container Toolkit:**
```bash
# Install NVIDIA Container Toolkit
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit

# Configure Docker to use NVIDIA runtime
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

**Use GPU container for faster proving:**
```bash
# Pull GPU container for your architecture (example: sm80 for RTX 3080/3090/A100)
docker pull ghcr.io/zkonduit/teeonnx-gpu-sm80:latest

# Generate proof using GPU (much faster than CPU)
docker run --runtime=nvidia -v $(pwd):/workspace ghcr.io/zkonduit/teeonnx-gpu-sm80:latest \
  prove --quote /workspace/quote.bin --proof /workspace/proof.json

# Verify proof
docker run --runtime=nvidia -v $(pwd):/workspace ghcr.io/zkonduit/teeonnx-gpu-sm80:latest \
  verify --proof /workspace/proof.json
```

**Available GPU containers:**
- `ghcr.io/zkonduit/teeonnx-gpu-sm70:latest` - Tesla V100, GTX 1080 Ti
- `ghcr.io/zkonduit/teeonnx-gpu-sm75:latest` - RTX 2080, RTX 2080 Ti, Tesla T4
- `ghcr.io/zkonduit/teeonnx-gpu-sm80:latest` - RTX 3080, RTX 3090, A100
- `ghcr.io/zkonduit/teeonnx-gpu-sm86:latest` - RTX 3050, RTX 3060, RTX 3070
- `ghcr.io/zkonduit/teeonnx-gpu-sm89:latest` - RTX 4090, RTX 4080
- `ghcr.io/zkonduit/teeonnx-gpu-sm90:latest` - H100

### Docker Development Mode

For development and testing without SGX hardware:

```bash
# Pull and run in simulation mode
docker run -e SGX_MODE=SW \
  -v $(pwd):/workspace \
  ghcr.io/zkonduit/teeonnx-sgx:latest \
  gen-output \
  --input /workspace/input.json \
  --model /workspace/network.onnx \
  --output /workspace/output.json \
  --quote /workspace/quote.bin
```

## Available Verification Commands

The verification binary supports these commands:

- `prove`: Generate RISC0 proof of DCAP quote validity
- `verify`: Verify a RISC0 proof
- `hash-check`: Verify hash bindings in quote user_data
- `mrenclave-check`: Validate the enclave measurement
- `help`: Show detailed help for each command

## Use Cases

- **Privacy-Preserving ML**: Run sensitive models with hardware-backed privacy guarantees
- **Verifiable AI**: Prove model execution without revealing the model or inputs
- **Compliance**: Meet regulatory requirements for secure computation environments
- **Hybrid Verification**: Combine traditional zkps with TEE attestations for enhanced security

## Security Considerations

- **Quote Freshness**: Verify quotes against current Intel collateral to prevent replay attacks
- **MRENCLAVE Validation**: Always verify the enclave measurement matches the expected code
- **Hardware Requirements**: Ensure SGX hardware is properly provisioned and updated

## Cryptographic Bindings

The enclave creates keccak256 hashes embedded in the 64-byte DCAP quote user_data:

- **Bytes 0-31**: `P(witness)` - Hash of witness outputs
- **Bytes 32-63**: `keccak256(P(circuit) || P(input))` - Combined input hash

Where:
- `P(circuit)`: keccak256 hash of compiled circuit data
- `P(input)`: keccak256 hash of input field elements

This enables verifiers to confirm computations without accessing input or circuit data in the clear.

## Performance Considerations

- **CPU vs CUDA**: CUDA binaries provide significant speedup for proof generation
- **Memory Requirements**: Ensure adequate RAM for large models (8GB+ recommended)
- **SGX Memory**: Large models may require SGX memory configuration adjustments

## Troubleshooting

### Common Issues

1. **SGX Device Not Found**: Ensure SGX is enabled in BIOS and drivers are installed
2. **Permission Denied**: Check SGX device permissions (`/dev/sgx_enclave`, `/dev/sgx_provision`)
3. **Quote Generation Failed**: Verify DCAP service is running and configured
4. **Docker Issues**: Ensure SGX devices are properly mounted in container

### Debug Mode

Run verification with debug output:
```bash
RUST_LOG=debug ./teeonnx-zk-cpu-linux prove --quote quote.bin --proof proof.json
```

## Version Information

This binary distribution is built from the latest stable release of teeonnx. For version-specific information, check the [releases page](https://github.com/zkonduit/teeonnx-p/releases).

## License
Copyright 2025 Zkonduit Inc. Production use requires a license. For licensing inquiries, please contact `licensing@ezkl.xyz`. 


## Acknowledgments

- [tract](https://github.com/sonos/tract) - Neural network inference library for ONNX
- [zkdcap](https://github.com/datachainlab/zkdcap) - DCAP quote verification library by Datachain Lab
- [Automata Network](https://automata.network) - SGX SDK and infrastructure
- [RISC Zero](https://risc0.com) - zkVM technology for quote verification
- [Intel SGX](https://www.intel.com/content/www/us/en/architecture-and-technology/software-guard-extensions.html) - Trusted execution environment

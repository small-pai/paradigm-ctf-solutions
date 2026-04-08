# paradigm-ctf-2021-solutions

My personal solutions for paradigm ctf 2021. I am a web3 security beginner. These solutions represent my own thought process, including where I got stuck, how I unblock myself, and what I learned. I aim to solve each challenge without relying on existing writeups first. If I do refer to others’ solutions, I will rewrite them in my own words and credit the source.

## Environment Setup

This project is built and tested on **Windows 11 + WSL2 (Ubuntu)**.

### Prerequisites

- Windows 11 with WSL2 enabled
- Docker
- Python 3.10+
- An Ethereum mainnet RPC endpoint (Infura / Alchemy)

### Quick Start

```bash
# Clone the official repo
git clone https://github.com/paradigmxyz/paradigm-ctf-2021.git
cd paradigm-ctf-2021

# Setup Python virtual environment
python3 -m venv venv
source venv/bin/activate
pip install solc-select ecdsa pysha3 web3
pip install git+https://github.com/lunixbochs/mpwn.git

# Build all challenges
./build.sh

# Run a challenge (e.g., Bank)
./run.sh bank 31337 8545
```

### Configure RPC Endpoint (Required for some challenges)

Challenges like `bank`, `swap`, `vault` need to fork Ethereum mainnet state.

```bash
export ETH_RPC_URL=https://mainnet.infura.io/v3/YOUR_API_KEY
```
Get a free endpoint from [Infura](https://infura.io/) or [Alchemy](https://www.alchemy.com/).

### Lessons Learned
(To be filled as I progress)

### Acknowledgments
[Paradigm CTF 2021](https://github.com/paradigmxyz/paradigm-ctf-2021) - The official challenge repository
Writeups and resources I referenced will be credited here

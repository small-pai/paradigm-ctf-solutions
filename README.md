# paradigm-ctf-2021-solutions

My personal solutions for paradigm ctf 2021. I am a web3 security beginner. These solutions represent my own thought process, including where I got stuck, how I unblock myself, and what I learned. I aim to solve each challenge without relying on existing writeups first. If I do refer to others’ solutions, I will rewrite them in my own words and credit the source.

## Environment Setup

This project is built and tested on **Windows 11 + WSL2 (Ubuntu)**.

> Note: The official repository uses `public/` for participant-visible files and `private/` for internal files.

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

## Solutions

| Challenge | Status |
|-----------|--------|
| hello | ✅ Solved |
| babycrypto | ✅ Solved |
| babyrev | ✅ Solved |
| babysandbox | ✅ Solved |
| lockbox | 🚧 Partially Solved |
| bank | 📖 Analyzed |
| market | 📖 Analyzed |
| ... | 🚧 In Progress |

### Lessons Learned
### Environment Setup (Windows 11 + WSL2)

| Issue | Cause | Solution |
|-------|-------|----------|
| `wsl --install` fails with `HCS_E_HYPERV_NOT_INSTALLED` | BIOS virtualization is disabled | Reboot and enable Intel VT-x / AMD SVM in BIOS |
| `wsl --install -d Ubuntu` download fails | Unstable GitHub connection | Manually install Ubuntu from Microsoft Store |
| Docker fails to pull `gcr.io` images | Authentication required | `gcloud auth configure-docker` |
| `pip install` times out | Slow PyPI mirror | `pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple` |
| `externally-managed-environment` error | Ubuntu 24.04 protection mechanism | Use virtual environment `python3 -m venv venv` |
| `pysha3` compilation fails | Python 3.12 incompatibility | Use `hashlib.sha3_256()` instead |
| Port already allocated | Previous container still running | `docker stop $(docker ps -q)` |

### Toolchain

- **`socat`**: Expose a Python script as a network service
  ```bash
  socat TCP-LISTEN:31337,reuseaddr,fork EXEC:"python chal.py",pty,stderr
- **`cast`**: EVM interaction tool — disassemble bytecode, send transactions, query on-chain state
- **Virtual environment**: Remember to run `source venv/bin/activate` every time you open a new terminal

### Problem-Solving Mindset

1. **Environment setup takes 80% of the time** — this is normal, not a reflection of your ability
2. **Looking things up is not cheating** — CTF allows searching and learning
3. **Manual operations are often more reliable than automation** — complex scripts are prone to timing issues


### Acknowledgments
[Paradigm CTF 2021](https://github.com/paradigmxyz/paradigm-ctf-2021) - The official challenge repository  
Writeups and resources I referenced:
[Christoph Michel - Paradigm CTF 2021 Solutions](https://cmichel.io/paradigm-ctf-2021-solutions/)

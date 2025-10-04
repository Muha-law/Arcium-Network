# Arcium Network Node Setup Guide

A comprehensive step-by-step guide on How to Run an ARX Node on Arcium Network Testnet.

## What is Arcium?

**Arcium** is a confidential computing network built on Solana that enables Multi-Party Computation (MPC). It allows developers to build applications where sensitive data can be processed without revealing it to any single party.

### What is an ARX Node?

An **ARX (Arcium Execution) Node** is a critical component of the Arcium network that:
- **Performs MPC Computations**: Executes secure, distributed computations where multiple nodes work together without exposing private data
- **Maintains Network Security**: Participates in cryptographic protocols that ensure data remains confidential during processing
- **Enables Trustless Processing**: Allows applications to process sensitive information (like financial data, health records, or private keys) without requiring trust in a single entity
- **Supports Decentralization**: Distributes computation across many nodes, preventing any single point of failure or control

### How ARX Nodes Help the System

ARX nodes are the computational backbone of Arcium's confidential computing infrastructure:

1. **Distributed Trust**: Instead of trusting one server with sensitive data, the workload is split across multiple ARX nodes. No single node can see the complete data.

2. **Secure Computation**: Using MPC protocols, nodes collectively perform computations on encrypted data, producing correct results without ever decrypting the inputs.

3. **Cluster Collaboration**: Nodes form clusters to work together on specific computations, creating a flexible and scalable network that can handle various workloads.

4. **Real Computation Testing**: By running a testnet node, you help stress test the network's performance, identify bottlenecks, and ensure the system can handle real-world demands.

---

## Hardware Requirements

| Component | Requirement |
|-----------|-------------|
| **RAM** | 32 GB |
| **CPU** | 4+ cores |
| **Storage** | 50+ GB SSD (minimal storage needed) |
| **Network** | Stable internet connection |

> **Note**: Storage requirements are minimal as ARX nodes don't store blockchain history. The primary resource requirement is RAM for MPC computations.

---

## Prerequisites

### For Windows Users
Windows is not natively supported. You must install Ubuntu on Windows using WSL2:
- Follow this guide: [Install Linux on Windows](https://learn.microsoft.com/en-us/windows/wsl/install)
- Use Ubuntu 22.04 LTS recommended

### For VPS Users
You can use any VPS provider with the required specifications. Recommended providers:
- AWS, Google Cloud, DigitalOcean, Vultr, Contabo, etc.
- Ensure port 8080 is accessible from the internet

### Required Software
You'll need to install:
- **Rust**: Programming language runtime
- **Solana CLI**: For interacting with Solana blockchain
- **Docker & Docker Compose**: For running the node container
- **OpenSSL**: For generating cryptographic keys
- **Git**: For version control

---

## Installation Steps

### 1. Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Required Packages

```bash
sudo apt install curl iptables build-essential git wget jq make gcc nano htop pkg-config libssl-dev tar clang unzip ca-certificates gnupg -y
```

### 3. Install Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

Verify installation:
```bash
rustc --version
```

### 4. Install Solana CLI

```bash
sh -c "$(curl -sSfL https://release.solana.com/stable/install)"
```

Add to PATH:
```bash
echo 'export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Verify installation:
```bash
solana --version
```

Set Solana to Devnet:
```bash
solana config set --url https://api.devnet.solana.com
```

### 5. Install Docker

```bash
# Remove old Docker versions
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do 
  sudo apt-get remove $pkg 2>/dev/null
done

# Install Docker
sudo apt-get update
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

Test Docker:
```bash
sudo docker run hello-world
```

Enable Docker to start on boot:
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### 6. Install Arcium Tooling

```bash
curl --proto '=https' --tlsv1.2 -sSfL https://arcium-install.arcium.workers.dev/ | bash
```

Add to PATH and reload:
```bash
source ~/.bashrc
```

Verify installation:
```bash
arcium --version
arcup --version
```

---

## Setup Your Node

### 7. Create Workspace Directory

```bash
mkdir arcium-node-setup
cd arcium-node-setup
```

> **IMPORTANT**: Stay in this directory for all remaining steps!

### 8. Get Your Public IP Address

```bash
curl https://ipecho.net/plain ; echo
```

Save this IP address - you'll need it later.

### 9. Configure Firewall

```bash
# Allow SSH
sudo ufw allow 22
sudo ufw allow ssh

# Allow ARX Node Port
sudo ufw allow 8080

# Enable firewall
sudo ufw enable
```

### 10. Generate Required Keypairs

#### 10.1 Node Authority Keypair
This identifies your node on Solana blockchain:

```bash
solana-keygen new --outfile node-keypair.json --no-bip39-passphrase
```

#### 10.2 Callback Authority Keypair
This signs computation callbacks (must be different from node keypair):

```bash
solana-keygen new --outfile callback-kp.json --no-bip39-passphrase
```

#### 10.3 Identity Keypair
This handles node-to-node communication:

```bash
openssl genpkey -algorithm Ed25519 -out identity.pem
```

> **⚠️ SECURITY WARNING**: Keep these keypairs private and secure! Back them up safely.

### 11. Get Public Keys

```bash
# Get your node public key
solana address --keypair node-keypair.json

# Get your callback public key
solana address --keypair callback-kp.json
```

Save both addresses!

### 12. Fund Your Accounts with Devnet SOL

Each account needs Devnet SOL for transaction fees.

**Method 1: Using Solana CLI**
```bash
# Fund node account (replace <node-pubkey> with your actual address)
solana airdrop 2 <node-pubkey> -u devnet

# Fund callback account (replace <callback-pubkey> with your actual address)
solana airdrop 2 <callback-pubkey> -u devnet
```

**Method 2: Using Web Faucet** (More Reliable)
- Visit: https://faucet.solana.com/
- Paste your node public key and request SOL
- Repeat for callback public key

Verify balances:
```bash
solana balance <node-pubkey> -u devnet
solana balance <callback-pubkey> -u devnet
```

### 13. Choose Your Node Offset

Your **node offset** is a unique identifier for your node on the network. 

- Choose a random 8-10 digit number (example: 847392561)
- If you get an error during initialization that your offset is taken, try a different number
- Save this number - you'll use it multiple times

### 14. Initialize Node Accounts On-Chain

This registers your node with the Arcium network:

```bash
arcium init-arx-accs \
  --keypair-path node-keypair.json \
  --callback-keypair-path callback-kp.json \
  --peer-keypair-path identity.pem \
  --node-offset YOUR_NODE_OFFSET \
  --ip-address YOUR_PUBLIC_IP \
  --rpc-url https://api.devnet.solana.com
```

**Replace:**
- `YOUR_NODE_OFFSET` with your chosen offset number
- `YOUR_PUBLIC_IP` with your server's public IP from Step 8

**Example:**
```bash
arcium init-arx-accs \
  --keypair-path node-keypair.json \
  --callback-keypair-path callback-kp.json \
  --peer-keypair-path identity.pem \
  --node-offset 847392561 \
  --ip-address 203.0.113.45 \
  --rpc-url https://api.devnet.solana.com
```

If successful, you'll see confirmation that your node accounts are initialized on-chain.

---

## Optional: Use Better RPC Providers

For improved reliability, consider using dedicated RPC providers instead of the default public endpoint:

**Recommended Free Options:**
- **Helius**: https://helius.xyz (free tier works great)
- **QuickNode**: https://quicknode.com (free tier works great)

The free tiers are completely sufficient for testnet. When you sign up, you'll get:
- RPC URL (HTTP): `https://your-project.helius-rpc.com`
- WebSocket URL (WSS): `wss://your-project.helius-rpc.com`

---

## Configure Your Node

### 15. Create Node Configuration File

```bash
nano node-config.toml
```

Paste the following configuration:

```toml
[node]
offset = YOUR_NODE_OFFSET  # Replace with your node offset
hardware_claim = 0
starting_epoch = 0
ending_epoch = 9223372036854775807

[network]
address = "0.0.0.0"  # Binds to all interfaces

[solana]
endpoint_rpc = "https://api.devnet.solana.com"  # Or your RPC provider URL
endpoint_wss = "wss://api.devnet.solana.com"    # Or your RPC provider WebSocket URL
cluster = "Devnet"
commitment.commitment = "confirmed"
```

**Replace:**
- `YOUR_NODE_OFFSET` with your node offset
- Optionally update `endpoint_rpc` and `endpoint_wss` with your dedicated RPC provider URLs

Save and exit: `CTRL+X`, then `Y`, then `ENTER`

---

## Join or Create a Cluster

Clusters are groups of nodes that work together on MPC computations. You have two options:

### Option A: Create Your Own Cluster

```bash
arcium init-cluster \
  --keypair-path node-keypair.json \
  --offset CLUSTER_OFFSET \
  --max-nodes 10 \
  --rpc-url https://api.devnet.solana.com
```

**Replace:**
- `CLUSTER_OFFSET`: Choose a unique number different from your node offset
- `--max-nodes`: Maximum nodes allowed in your cluster (10 is good for testing)

**Example:**
```bash
arcium init-cluster \
  --keypair-path node-keypair.json \
  --offset 555123456 \
  --max-nodes 10 \
  --rpc-url https://api.devnet.solana.com
```

After creating, share your cluster offset with others to invite them!

### Option B: Join an Existing Cluster

To join a cluster, you need an invitation from the cluster authority. Once invited:

```bash
arcium join-cluster true \
  --keypair-path node-keypair.json \
  --node-offset YOUR_NODE_OFFSET \
  --cluster-offset CLUSTER_OFFSET \
  --rpc-url https://api.devnet.solana.com
```

**Replace:**
- `YOUR_NODE_OFFSET`: Your node offset
- `CLUSTER_OFFSET`: The cluster offset you're joining

> **Note**: If you want to reject an invitation, use `false` instead of `true`

---

## Run Your ARX Node

### 16. Prepare Log Directory

```bash
mkdir -p arx-node-logs && touch arx-node-logs/arx.log
```

### 17. Verify Files

Make sure you're in the correct directory and have all files:

```bash
pwd  # Should show: /path/to/arcium-node-setup
ls   # Should show: node-keypair.json, callback-kp.json, identity.pem, node-config.toml, arx-node-logs/
```

### 18. Start the Docker Container

```bash
docker run -d \
  --name arx-node \
  -e NODE_IDENTITY_FILE=/usr/arx-node/node-keys/node_identity.pem \
  -e NODE_KEYPAIR_FILE=/usr/arx-node/node-keys/node_keypair.json \
  -e OPERATOR_KEYPAIR_FILE=/usr/arx-node/node-keys/operator_keypair.json \
  -e CALLBACK_AUTHORITY_KEYPAIR_FILE=/usr/arx-node/node-keys/callback_authority_keypair.json \
  -e NODE_CONFIG_PATH=/usr/arx-node/arx/node_config.toml \
  -v "$(pwd)/node-config.toml:/usr/arx-node/arx/node_config.toml" \
  -v "$(pwd)/node-keypair.json:/usr/arx-node/node-keys/node_keypair.json:ro" \
  -v "$(pwd)/node-keypair.json:/usr/arx-node/node-keys/operator_keypair.json:ro" \
  -v "$(pwd)/callback-kp.json:/usr/arx-node/node-keys/callback_authority_keypair.json:ro" \
  -v "$(pwd)/identity.pem:/usr/arx-node/node-keys/node_identity.pem:ro" \
  -v "$(pwd)/arx-node-logs:/usr/arx-node/logs" \
  -p 8080:8080 \
  arcium/arx-node
```

> **Note**: The operator keypair intentionally uses the same file as your node keypair for simplified testnet setup.

### 19. Check Node Logs

```bash
docker logs -f arx-node
```

Press `CTRL+C` to exit logs (node keeps running).

---

## Verify Node Operation

### Check Node Status

```bash
arcium arx-info YOUR_NODE_OFFSET --rpc-url https://api.devnet.solana.com
```

### Check if Node is Active

```bash
arcium arx-active YOUR_NODE_OFFSET --rpc-url https://api.devnet.solana.com
```

### Monitor Real-Time Logs

```bash
docker logs -f arx-node
```

---

## Useful Docker Commands

### View Running Containers
```bash
docker ps
```

### Stop Your Node
```bash
docker stop arx-node
```

### Start Your Node Again
```bash
docker start arx-node
```

### Restart Your Node
```bash
docker restart arx-node
```

### Remove Container (keeps data)
```bash
docker stop arx-node
docker rm arx-node
```

After removing, you can start fresh using the Docker run command from Step 18.

---

## Troubleshooting

### Node Not Starting

**Check if container is running:**
```bash
docker ps
```

**If not listed, check all containers:**
```bash
docker ps -a
```

**View error logs:**
```bash
docker logs arx-node
```

**Common fixes:**
- Verify all keypair files exist: `ls -lah`
- Verify `node-config.toml` has correct syntax
- Ensure your IP is accessible from the internet
- Check if port 8080 is open: `sudo ufw status`

### Account Initialization Failed

- Verify you have at least 2 SOL in each account
- Try using a different RPC endpoint (Helius or QuickNode)
- Ensure your node offset is unique (try a different number)

### Cannot Join Cluster

- Verify you've been invited by the cluster authority
- Check that the cluster has available slots
- Ensure your node is properly initialized with `arx-info`

### Port Already in Use

```bash
# Find what's using port 8080
sudo lsof -i :8080

# Kill the process if needed
sudo kill -9 <PID>
```

### Docker Permission Issues

If you get permission errors:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Out of Memory Errors

- Verify you have 32GB RAM: `free -h`
- Close unnecessary applications
- Consider using a VPS with more RAM

---

## What to Expect in Testnet

Your ARX node is now part of Arcium's testnet! Here's what happens:

- ✅ **Real MPC Computations**: Your node performs actual multi-party computations
- ✅ **Stress Testing**: Helps test network performance and scalability
- ✅ **No Real Value**: Everything runs on Solana Devnet (testnet SOL has no value)
- ✅ **Variable Activity**: Sometimes busy, sometimes quiet depending on testing schedules
- ❌ **No Rewards Yet**: Staking and reward mechanisms aren't enabled in testnet

Think of it as helping build and test the infrastructure before mainnet launch!

---

## Security Best Practices

🔒 **Critical Security Reminders:**

1. **Never share your private keys** (`node-keypair.json`, `callback-kp.json`, `identity.pem`)
2. **Backup your keypairs** securely in multiple locations
3. **Use firewalls** to restrict unnecessary network access
4. **Keep systems updated**: `sudo apt update && sudo apt upgrade`
5. **Monitor your node logs** regularly for suspicious activity
6. **Use strong SSH passwords** or key-based authentication
7. **Don't reuse testnet keys** for mainnet when it launches

---

## Community & Support

### Join the Community

- **Discord**: https://discord.gg/arcium
- **Twitter**: Follow Arcium for updates
- **Documentation**: https://docs.arcium.com/

### Getting Help

If you encounter issues:

1. Check this troubleshooting section
2. Review your Docker logs: `docker logs arx-node`
3. Ask in the Arcium Discord #node-support channel
4. Ensure you're using the latest version: `arcium --version`

---

## Frequently Asked Questions

**Q: How much does it cost to run a testnet node?**
A: Only your server costs (VPS) or electricity. Devnet SOL is free.

**Q: Will testnet participants get rewards on mainnet?**
A: Not officially announced yet. Join Discord for updates.

**Q: Can I run multiple nodes on one server?**
A: Yes, but you'll need to adjust ports and offsets for each node.

**Q: How do I know if my node is working correctly?**
A: Use `arcium arx-active YOUR_NODE_OFFSET` and check your logs.

**Q: What happens if my node goes offline?**
A: Simply restart it when possible. No penalties in testnet.

---

## Next Steps

Once your node is running successfully:

1. ✅ Join the [Arcium Discord](https://discord.gg/arcium)
2. ✅ Introduce yourself in the community
3. ✅ Monitor your node's performance
4. ✅ Stay updated on testnet activities
5. ✅ Report any bugs or issues you encounter
6. ✅ Keep your Arcium tooling updated

---

## Additional Resources

- **Official Documentation**: https://docs.arcium.com/
- **GitHub Repository**: https://github.com/arcium-hq
- **Discord Community**: https://discord.gg/arcium

---

## Credits

Guide created by [muhalaw](https://x.com/muhalaw)

**Contributions welcome!** Feel free to submit PRs to improve this guide.

---

*Last Updated: October 2025*
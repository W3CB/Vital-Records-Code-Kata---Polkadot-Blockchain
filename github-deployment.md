# Deployment Instructions for Vital Records Management Pallet

> A comprehensive guide for packaging, deploying, and upgrading the Vital Records Management pallet on Polkadot or a Substrate-based chain.

- [Pre-Deployment Checklist](#1-pre-deployment-checklist)
- [Packaging for Production](#2-packaging-for-production)
- [Runtime Upgrade Planning](#3-runtime-upgrade-planning)
- [Testing the Upgrade](#4-testing-the-upgrade)
- [Upgrade Execution](#5-upgrade-execution)
- [Governance Processes](#6-governance-processes)
- [Post-Deployment Monitoring](#7-post-deployment-monitoring)
- [Rollback Procedures](#8-rollback-procedures)
- [Vital Records Special Considerations](#9-vital-records-special-considerations)

## 1. Pre-Deployment Checklist

Before deploying your Vital Records Management pallet to production, complete this checklist:

- [ ] All unit and integration tests pass
  ```bash
  cargo test -p pallet-vital-records
  ```

- [ ] Benchmarking is completed and weights are implemented
  ```bash
  cargo run --release -- benchmark pallet \
      --pallet pallet-vital-records \
      --execution=wasm --wasm-execution=compiled
  ```

- [ ] Security audit completed with no critical issues
  - [ ] Access control checks verified
  - [ ] Input validation confirmed
  - [ ] No storage poisoning vulnerabilities
  - [ ] No front-running vulnerabilities

- [ ] Documentation is complete and up-to-date
  - [ ] Function documentation
  - [ ] Storage documentation
  - [ ] Error documentation
  - [ ] Event documentation

- [ ] Storage migrations (if needed) are implemented and tested

- [ ] Regulatory compliance requirements met
  - [ ] GDPR compliance for EU deployments
  - [ ] Data minimization principles applied
  - [ ] Proper consent management implemented

## 2. Packaging for Production

### 2.1 Update Version and Dependencies

Update your pallet's version in `pallets/vital-records/Cargo.toml`:

```toml
[package]
name = "pallet-vital-records"
version = "1.0.0"  # Use semantic versioning
authors = ["Your Name <your.email@example.com>"]
edition = "2021"

[dependencies]
codec = { package = "parity-scale-codec", version = "3.0.0", features = ["derive"], default-features = false }
scale-info = { version = "2.9.0", default-features = false, features = ["derive"] }
frame-support = { default-features = false, version = "21.0.0" }
frame-system = { default-features = false, version = "21.0.0" }
# Other dependencies...
```

### 2.2 Optimize for Production

Configure the release profile in your workspace `Cargo.toml`:

```toml
[profile.release]
panic = "unwind"
lto = true
codegen-units = 1
opt-level = 3
strip = true
```

### 2.3 Build the Runtime

```bash
# Build the runtime with your pallet included
cargo build --release -p your-runtime
```

This generates:
- `target/release/wbuild/your-runtime/your_runtime.compact.wasm` (smaller, optimized)
- `target/release/wbuild/your-runtime/your_runtime.wasm` (full runtime)

### 2.4 Extract Runtime Metadata

```bash
# Using subkey (if you have a node running)
subkey inspect-node-raw --url http://localhost:9944 --output-metadata > metadata.scale

# Or using polkadot.js API (Node.js script)
const { ApiPromise, WsProvider } = require('@polkadot/api');

async function extractMetadata() {
  const api = await ApiPromise.create({ provider: new WsProvider('ws://localhost:9944') });
  const metadata = await api.rpc.state.getMetadata();
  require('fs').writeFileSync('metadata.scale', metadata.toU8a());
  process.exit(0);
}

extractMetadata();
```

### 2.5 Create the Release Package

```bash
# Create a release directory
mkdir -p vital-records-release-v1.0.0/docs

# Copy release files
cp target/release/wbuild/your-runtime/your_runtime.compact.wasm vital-records-release-v1.0.0/
cp target/release/wbuild/your-runtime/your_runtime.wasm vital-records-release-v1.0.0/
cp metadata.scale vital-records-release-v1.0.0/
cp -r target/doc/* vital-records-release-v1.0.0/docs/
cp CHANGELOG.md vital-records-release-v1.0.0/
cp README.md vital-records-release-v1.0.0/

# If you have migration docs
cp migrations.md vital-records-release-v1.0.0/
```

## 3. Runtime Upgrade Planning

### 3.1 Determine Upgrade Type

- **Soft Fork**: Backward-compatible changes, doesn't require all nodes to upgrade
- **Hard Fork**: Breaking changes, requires coordinated upgrade
- **Forkless Upgrade**: Using Substrate's native runtime upgrade mechanism

### 3.2 Storage Migration

If your upgrade changes storage layout:

1. Create a migration module:

```rust
// pallets/vital-records/src/migrations.rs
pub mod v1_to_v2 {
    use super::*;
    use frame_support::{
        storage::migration,
        traits::OnRuntimeUpgrade,
        weights::Weight,
    };
    
    pub struct Migration<T>(sp_std::marker::PhantomData<T>);
    
    impl<T: Config> OnRuntimeUpgrade for Migration<T> {
        fn on_runtime_upgrade() -> Weight {
            // Example: Convert old storage format to new format
            let mut count = 0;
            
            // Example migration: Update MarriageLicense format
            MarriageLicenses::<T>::translate::<OldLicenseType, _>(|key, old_license| {
                count += 1;
                // Convert old format to new format
                Some(NewLicenseType {
                    // mapping logic here
                })
            });
            
            // Log migration
            log::info!("💾 Migration complete: migrated {} marriage licenses", count);
            
            // Return consumed weight
            T::DbWeight::get().reads_writes(count as u64, count as u64)
        }
        
        #[cfg(feature = "try-runtime")]
        fn pre_upgrade() -> Result<(), &'static str> {
            // Verify state before migration
            Ok(())
        }
        
        #[cfg(feature = "try-runtime")]
        fn post_upgrade() -> Result<(), &'static str> {
            // Verify state after migration
            Ok(())
        }
    }
}
```

2. Register the migration in your runtime's `on_runtime_upgrade` hook:

```rust
// runtime/src/lib.rs
construct_runtime!(
    pub enum Runtime where
        Block = Block,
        NodeBlock = opaque::Block,
        UncheckedExtrinsic = UncheckedExtrinsic
    {
        // other pallets...
        VitalRecords: pallet_vital_records,
    }
);

// Add migration hook
pub type Executive = frame_executive::Executive<
    Runtime,
    Block,
    frame_system::ChainContext<Runtime>,
    Runtime,
    AllPalletsWithSystem,
    (
        // Migrations ordered by execution
        pallet_vital_records::migrations::v1_to_v2::Migration<Runtime>,
        // other migrations...
    )
>;
```

### 3.3 Version Your Storage

Always version your storage to support future migrations:

```rust
// In your pallet
#[pallet::storage]
#[pallet::getter(fn pallet_version)]
pub type PalletVersion<T> = StorageValue<_, StorageVersion, ValueQuery>;

// Define versions
#[derive(Encode, Decode, Clone, TypeInfo, Debug, PartialEq, Eq)]
pub enum StorageVersion {
    V1,
    V2,
}

impl Default for StorageVersion {
    fn default() -> Self {
        Self::V1
    }
}
```

## 4. Testing the Upgrade

### 4.1 Try-Runtime Testing

Test your migrations with try-runtime:

```bash
# Run try-runtime tests
cargo build --release --features try-runtime

# Test on existing chain state
cargo run --release -p your-node -- try-runtime \
    --runtime ./target/release/wbuild/your-runtime/your_runtime.wasm \
    on-runtime-upgrade live \
    --uri wss://your-node:9944
```

### 4.2 Simulate Upgrade on Testnet

1. Deploy to a testnet before production:

```bash
# Using sudo (if available on testnet)
polkadot-js-api --ws wss://testnet:9944 tx.sudo.sudoUncheckedWeight \
    system.setCode 0x$(cat your_runtime.wasm | xxd -p | tr -d '\n') 1000000000
```

2. Verify post-upgrade:
   - All existing records are accessible
   - New features work correctly
   - Performance meets expectations
   - Client applications remain functional

### 4.3 Execute a Canary Deployment

1. Deploy to a small subset of production first
2. Monitor for issues
3. Gradually roll out to more users

## 5. Upgrade Execution

### 5.1 For Solo Chains

Using sudo (if available):

```bash
polkadot-js-api --ws wss://your-chain:9944 tx.sudo.sudoUncheckedWeight \
    system.setCode 0x$(cat your_runtime.wasm | xxd -p | tr -d '\n') 1000000000
```

### 5.2 For Parachains on Polkadot/Kusama

Submit a parachain upgrade proposal:

```bash
polkadot-js-api --ws wss://your-parachain:9944 tx.parachain.setCode \
    0x$(cat your_runtime.wasm | xxd -p | tr -d '\n')
```

### 5.3 Monitoring the Upgrade

Monitor these during and after upgrade:

- Block production continues without issues
- Finality gadget is working
- Events are being emitted correctly
- RPC endpoints respond correctly
- Validators are on the new version

## 6. Governance Processes

### 6.1 On-Chain Governance Flow

#### Pre-Governance Discussion

1. Create a detailed proposal document:
   - Technical changes
   - User impact
   - Security considerations
   - Testing results

2. Share with the community:
   - Post on governance forums
   - Discuss in community calls
   - Gather feedback and address concerns

#### Technical Committee Review

Before formal proposal:

1. Submit code for technical committee review
2. Address all technical feedback
3. Get formal technical approval

#### Submit Democracy Proposal

For runtime upgrades:

```bash
# Submit a preimage of the upgrade
polkadot-js-api --ws wss://your-chain:9944 \
    tx.democracy.notePreimage 0x<encoded_proposal> --account <proposer>

# Submit the proposal using the preimage hash
polkadot-js-api --ws wss://your-chain:9944 \
    tx.democracy.propose <preimage_hash> <amount_to_lock> --account <proposer>
```

#### Voting Period

- Referendum appears in democracy queue
- Community votes (usually 7-28 days)
- If approved, moves to enactment queue

#### Enactment

- After enactment period, upgrade automatically applies
- Monitor chain during and after enactment

### 6.2 Parachain Council Governance

For chains using council governance:

1. Submit motion to council:

```bash
polkadot-js-api --ws wss://your-parachain:9944 \
    tx.council.propose <threshold> system.setCode 0x<wasm_blob> <length_bound> --account <council_member>
```

2. Council voting:

```bash
polkadot-js-api --ws wss://your-parachain:9944 \
    tx.council.vote <proposal_hash> <proposal_index> <approve> --account <council_member>
```

3. Execute approved motion:

```bash
polkadot-js-api --ws wss://your-parachain:9944 \
    tx.council.close <proposal_hash> <proposal_index> --account <any_account>
```

### 6.3 Emergency Procedures

For critical updates (e.g., security vulnerabilities):

1. Technical Committee Fast-Track:

```bash
# Technical committee proposes fast-track
polkadot-js-api --ws wss://your-chain:9944 \
    tx.technicalCommittee.propose <threshold> democracy.fastTrack <proposal_hash> <voting_period> <delay> --account <committee_member>

# After approval, execute
polkadot-js-api --ws wss://your-chain:9944 \
    tx.technicalCommittee.close <proposal_hash> <proposal_index> --account <any_account>
```

2. Emergency Root Origin (extreme cases only):

```bash
polkadot-js-api --ws wss://your-chain:9944 \
    tx.sudo.sudoUncheckedWeight system.setCode 0x<wasm_blob> 1000000000 --account <sudo_key>
```

### 6.4 Communication Strategy

For all governance processes:

1. Before Proposal:
   - Publish technical specifications
   - Host developer Q&A sessions
   - Release testing guidelines

2. During Voting:
   - Regular status updates
   - Address questions and concerns
   - Provide voting guides

3. After Implementation:
   - Publish post-deployment report
   - Document any issues and resolutions
   - Share metrics on new functionality

## 7. Post-Deployment Monitoring

### 7.1 Key Metrics to Monitor

- Block production rate and finality
- Transaction throughput and latency
- Storage growth rate
- CPU and memory usage on validator nodes
- Error rates in logs
- Number of records processed by type
- Voter registration volumes (especially near elections)

### 7.2 Setup Monitoring Infrastructure

1. Prometheus Metrics:

```toml
# Add to node config
[prometheus_endpoint]
port = 9615
```

2. Grafana Dashboards for:
   - Node performance
   - Transaction volumes
   - Record processing rates
   - Governance activity

3. Configure alerts for:
   - Chain halts or slow blocks
   - High error rates
   - Storage anomalies
   - Unexpected transaction patterns

### 7.3 Regular Maintenance Schedule

- Weekly review of metrics and logs
- Monthly security updates review
- Quarterly performance optimization
- Bi-annual full system audit

## 8. Rollback Procedures

### 8.1 Prepare Rollback Plan

1. Keep Previous Runtime:

```bash
# Archive previous runtime
cp target/release/wbuild/your-runtime/your_runtime.wasm rollback/your_runtime_v1.0.0.wasm
```

2. Document Rollback Procedure:

```markdown
# Rollback Procedure

1. Identify need for rollback based on monitoring alerts
2. Notify all validators and stakeholders
3. Submit rollback proposal via governance
4. Monitor chain during and after rollback
5. Verify system returns to stable state
```

### 8.2 Emergency Rollback

For critical issues:

```bash
# Using sudo (if available)
polkadot-js-api --ws wss://your-chain:9944 \
    tx.sudo.sudoUncheckedWeight system.setCode 0x$(cat rollback/your_runtime_v1.0.0.wasm | xxd -p | tr -d '\n') 1000000000

# Or fast-track democracy proposal
polkadot-js-api --ws wss://your-chain:9944 \
    tx.technicalCommittee.propose <threshold> democracy.fastTrack <rollback_proposal_hash> <voting_period> <delay> --account <committee_member>
```

## 9. Vital Records Special Considerations

### 9.1 Data Sensitivity and Compliance

1. Data Protection Impact Assessment

Before deploying in production:
   
- Complete a DPIA (Data Protection Impact Assessment)
- Verify compliance with relevant regulations (GDPR, etc.)
- Document privacy protection mechanisms

2. Government Partnership Procedure

When deploying with government entities:
   
- Prepare formal documentation for government IT review
- Schedule security audits with government security teams
- Establish SLAs for uptime and response times

### 9.2 Initial Registrar Setup

After deployment, initialize the first registrar:

```bash
polkadot-js-api --ws wss://your-chain:9944 \
    tx.sudo.sudo vitalRecords.initFirstRegistrar <registrar_address> --account <sudo_key>
```

### 9.3 Emergency Simulation Testing

Before full production use:

```bash
polkadot-js-api --ws wss://your-chain:9944 \
    tx.vitalRecords.startDisasterSimulation <scenario> <affected_districts> --account <registrar>
```

### 9.4 Data Integrity and Auditing

1. Setup regular integrity checks:

```bash
# Script to verify data consistency
#!/bin/bash
echo "Running vital records integrity check..."
curl -H "Content-Type: application/json" -d '{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "vitalRecords_verifyDataIntegrity",
  "params": []
}' http://localhost:9933
```

2. Implement audit trail for all records:
   - Who accessed each record
   - Changes made to records
   - Reason for changes
   - Timestamp of changes

## Additional Resources

- [Substrate Runtime Upgrade Documentation](https://docs.substrate.io/build/upgrade-the-runtime/)
- [Polkadot Governance Documentation](https://wiki.polkadot.network/docs/learn-governance)
- [Substrate Migration Guide](https://docs.substrate.io/reference/how-to-guides/storage-migrations/)
- [Prometheus & Grafana Setup Guide](https://docs.substrate.io/tutorials/get-started/monitor-node-metrics/)

---

These deployment instructions are maintained alongside the Vital Records Management pallet code. For questions or suggestions, please open an issue on the repository.

# Troubleshooting Guide for Vital Records Management Kata

> This guide helps you diagnose and resolve common issues encountered when implementing the Vital Records Management system on Polkadot/Substrate.

- [Compilation Errors](#compilation-errors)
- [Runtime Errors](#runtime-errors)
- [Event and Storage Issues](#event-and-storage-issues)
- [Substrate-Specific Debugging Techniques](#substrate-specific-debugging-techniques)
- [Weight and Performance Optimization](#weight-and-performance-optimization)
- [Common Implementation Errors](#common-implementation-errors)
- [Environment Issues](#substratepolkadot-environment-issues)
- [Advanced Troubleshooting](#advanced-troubleshooting)

## Compilation Errors

| Error | Possible Cause | Solution |
|-------|---------------|----------|
| `error[E0277]: the trait bound 'XXX: Parameter' is not satisfied` | Type isn't properly configured for storage | Ensure your type implements all required traits (`Encode`, `Decode`, `TypeInfo`, etc.) |
| `error[E0282]: type annotations needed` | Rust cannot infer type | Add explicit type annotations, especially for storage items |
| `error[E0080]: evaluation of constant value failed` | Issues with const generics or overflow | Check numeric literals and calculations in const expressions |
| `custom attribute panicked: invalid 'cfg_attr'` | Invalid Substrate macro attributes | Check syntax in macro attributes like `#[pallet::storage]` |
| `cannot find macro 'pallet' in this scope` | Missing imports or features | Ensure `frame_support::pallet` is imported properly |

## Runtime Errors

| Error | Possible Cause | Solution |
|-------|---------------|----------|
| `NotAuthorized` | Caller doesn't have required permissions | Check that origin is correct; verify registrar status before action |
| `BadOrigin` | Using wrong origin type for call | Ensure you're using `ensure_signed` or `ensure_root` correctly |
| `InvalidStatus` | Record in wrong state for operation | Add state transition checks; verify current status before modification |
| `Arithmetic operation overflowed` | Overflow in calculation | Use `saturating_add`/`saturating_sub` or checked operations |
| `StorageOverflow` | Vec or storage grows too large | Add bounds checking; consider using BoundedVec |

## Event and Storage Issues

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| Event not emitted | Forgot to call `deposit_event` | Add event deposit after successful operations |
| Event fields incorrect | Wrong event parameters | Check event field order and types match definition |
| Storage item not updated | Storage operation missing | Verify all storage maps are updated appropriately |
| Storage inconsistency | Partial update when error occurs | Use transactional storage pattern or ensure atomic updates |
| Query returns wrong data | Incorrect key used | Check storage key construction, especially for double maps |

## Substrate-Specific Debugging Techniques

### Using `debug` Module

Add this to your pallet to enable runtime debugging:

```rust
#[cfg(feature = "std")]
use frame_support::debug;

// Then use in your code:
debug::RuntimeLogger::init();
debug::info!("Processing record: {:?}", record_id);
```

### Effective Tracing with Events

Design detailed events for debugging:

```rust
#[pallet::event]
#[pallet::generate_deposit(pub(super) fn deposit_event)]
pub enum Event<T: Config> {
    // Instead of this:
    RecordCreated(T::Hash),
    
    // Use this:
    RecordCreated {
        record_id: T::Hash,
        creator: T::AccountId,
        timestamp: T::Moment,
        record_type: RecordType,
    },
}
```

### Storage Inspection

Inspect storage directly via RPC calls:

```bash
# Using curl to check storage
curl -H "Content-Type: application/json" -d '{"id":1, "jsonrpc":"2.0", "method": "state_getStorage", "params": ["0x..."]}' http://localhost:9933
```

Or use Polkadot.js Apps UI to browse storage (easier).

### Chain State Monitoring

```bash
# Monitor specific storage changes
subkey watch --substrate "vital_records MarriageLicenses<AccountId>"
```

### Unit Test Debugging

```rust
// Verbose test output
#[test]
fn test_with_logs() {
    new_test_ext().execute_with(|| {
        println!("Starting test");
        // Test code...
        println!("Current registrar status: {:?}", VitalRecords::registrars(1));
    });
}
```

Run with:
```bash
cargo test test_with_logs -- --nocapture
```

## Weight and Performance Optimization

### Weight Calculation Issues

| Issue | Cause | Solution |
|-------|------|----------|
| `Transaction weight is too high` | Function consumes too many resources | Review weight calculations; split into multiple operations |
| Slow transaction processing | Inefficient storage operations | Minimize storage reads/writes; use StorageValue over StorageMap where possible |
| Out of gas errors | Weight miscalculated | Benchmark functions and update weights accordingly |

### Performance Optimization Tips

1. **Minimize Storage Operations**

```rust
// Inefficient:
let mut data = Self::my_storage_map(key);
data.field1 = new_value1;
MyStorageMap::<T>::insert(key, data);
data = Self::my_storage_map(key);
data.field2 = new_value2;
MyStorageMap::<T>::insert(key, data);

// Optimized:
let mut data = Self::my_storage_map(key);
data.field1 = new_value1;
data.field2 = new_value2;
MyStorageMap::<T>::insert(key, data);
```

2. **Use Storage Efficiently**

```rust
// Consider using:
StorageValue<_, u32> // For single values
StorageMap<_, Blake2_128Concat, Key, Value> // For key-value pairs
StorageDoubleMap<_, Blake2_128Concat, Key1, Blake2_128Concat, Key2, Value> // For two-key maps
```

3. **Optimize Storage Keys**

Use appropriate key hashing algorithms:
- `Blake2_128Concat` - Most situations (secure + allows iteration)
- `Twox64Concat` - When keys are not user-controlled
- `Identity` - Only for sequential numeric keys

4. **Optimize Data Structures**

```rust
// Instead of this:
type BigRecord<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    T::Hash,
    (Vec<u8>, Vec<u8>, Vec<u8>, Vec<u8>, Vec<u8>, T::AccountId, T::BlockNumber),
>;

// Consider this:
#[derive(Clone, Encode, Decode, PartialEq, RuntimeDebug, TypeInfo)]
pub struct Record<AccountId, BlockNumber> {
    pub name: BoundedVec<u8, ConstU32<64>>,
    pub details: BoundedVec<u8, ConstU32<256>>,
    pub owner: AccountId,
    pub created_at: BlockNumber,
}

type OptimizedRecord<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    T::Hash,
    Record<T::AccountId, T::BlockNumber>,
>;
```

5. **Lazy Loading**

Only load data when needed:

```rust
// Instead of this in your function
let all_records = SomeMap::<T>::iter().collect::<Vec<_>>();

// Do this
if let Some(record) = SomeMap::<T>::get(specific_key) {
    // Process only what you need
}
```

## Common Implementation Errors in Vital Records System

### Marriage License Issues

| Issue | Likely Cause | Solution |
|-------|-------------|----------|
| Cannot issue license | Registrar permissions missing | Check registrar initialization and permissions |
| Duplicate licenses for same partners | Missing uniqueness check | Add check for existing licenses before creating new ones |
| License status transitions incorrect | Missing state validation | Add explicit state transition validation |

### Birth Certificate Issues

| Issue | Likely Cause | Solution |
|-------|-------------|----------|
| Cannot associate with person account | Person already has certificate | Check for existing certificate before association |
| Certificate amendment fails | Wrong status or unauthorized | Verify certificate status and caller permissions |
| Parent validation fails | Missing parent verification | Ensure parent accounts and identities are verified |

### Death Certificate Issues

| Issue | Likely Cause | Solution |
|-------|-------------|----------|
| Multiple death certificates | Missing uniqueness check | Check if person already has death certificate |
| Death effects not processing | Missing cross-record updates | Ensure process_death_certificate_effects is called |
| Death date validation | Birth/death date inconsistency | Add timestamp validation between birth and death |

### Driver's License Issues

| Issue | Likely Cause | Solution |
|-------|-------------|----------|
| Underage issuance works | Age calculation error | Fix timestamp comparison for age verification |
| License not expiring | Expiry check missing | Add explicit expiry date validation |
| Cannot revoke license | Status transition missing | Ensure all status transitions are allowed and handled |

### Voter Registration Issues

| Issue | Likely Cause | Solution |
|-------|-------------|----------|
| Dead voters not removed | Missing cross-check with death certs | Add automatic voter removal on death certificate issuance |
| Duplicate district entries | Missing unique constraint | Ensure voters can only be in one district at a time |
| Eligibility verification bypassed | Missing validation checks | Add comprehensive eligibility validation |

## Substrate/Polkadot Environment Issues

### Common Setup Issues

| Issue | Cause | Solution |
|-------|------|----------|
| `cargo build` fails with missing features | Features not enabled | Enable required features in Cargo.toml |
| Node won't start after runtime upgrade | Runtime API version mismatch | Ensure runtime API versions match with node |
| Chain spec errors | Invalid chain specification | Check genesis configuration and parameters |
| Runtime panics on startup | Storage migration errors | Add proper migration handling for storage changes |

### Node Connection Issues

| Issue | Cause | Solution |
|-------|------|----------|
| Cannot connect to node | RPC not enabled | Enable RPC in node config with correct interfaces |
| Unauthorized RPC access | Missing RPC permissions | Configure RPC methods allowed in node settings |
| WebSocket connection drops | Large responses or timeouts | Adjust WebSocket settings; break large queries |

## Advanced Troubleshooting

### Runtime Upgrade Issues

If you're upgrading your runtime with new functionality:

1. **Storage migrations**: Ensure storage formats are compatible or migrated
2. **Weight changes**: Recalculate weights after significant logic changes
3. **API compatibility**: Maintain backwards compatibility in runtime APIs

### Benchmarking Problems

If benchmarking fails:

1. Check benchmarking code doesn't panic under any conditions
2. Ensure range of input values is reasonable
3. Verify benchmark instances cover worst-case scenarios

### Security Vulnerability Checks

Common security issues to watch for:

1. **Origin validation bypass**: Always check origin permissions
2. **Underflow/overflow**: Use saturating/checked math operations
3. **Storage poisoning**: Validate all user inputs before storage
4. **Front-running**: Consider transaction ordering attacks for sensitive operations
5. **Logic errors in validation**: Thoroughly test all validation paths

## Performance Profiling

To identify performance bottlenecks:

1. Use benchmarking to measure function costs
2. Add metrics collection to your node
3. Analyze storage access patterns
4. Check for unnecessary Vec allocations or cloning
5. Review weight calculations for accuracy

```rust
// Add benchmarking support to measure function costs
#[pallet::weight(T::WeightInfo::issue_marriage_license())]
pub fn issue_marriage_license(
    origin: OriginFor<T>,
    license_id: T::Hash,
) -> DispatchResult {
    // Function implementation
}
```

## Community Resources

If you're still stuck after trying these solutions:

- [Substrate StackExchange](https://substrate.stackexchange.com/)
- [Polkadot Discord](https://discord.gg/wGUDt2p)
- [Substrate GitHub Issues](https://github.com/paritytech/substrate/issues)
- [Substrate Documentation](https://docs.substrate.io/)

## Contribution

If you find other common issues and solutions while working on the Vital Records Kata, please submit a PR to update this troubleshooting guide!

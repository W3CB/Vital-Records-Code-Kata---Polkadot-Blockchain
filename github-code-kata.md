# Polkadot Blockchain Code Kata: Vital Records Management System

> This progressive kata guides you through building a comprehensive vital records management system on the Polkadot blockchain.

- [Prerequisites and Environment Setup](#prerequisites-and-environment-setup)
- [Step 1: Marriage License Issuance](#step-1-marriage-license-issuance)
- [Step 2: Birth Certificate Registration](#step-2-birth-certificate-registration)
- [Step 3: Death Certificate Registration](#step-3-death-certificate-registration)
- [Step 4: Driver's License Management](#step-4-drivers-license-management)
- [Step 5: Voter Registration](#step-5-voter-registration)
- [Advanced Features](#advanced-features)
- [Testing Instructions](#testing-instructions)

## Prerequisites and Environment Setup

### Required Tools and Libraries

1. **Rust and Cargo**
   - Install Rust using rustup: https://rustup.rs/
   - Required version: 1.68.0 or newer
   - Make sure you have the nightly toolchain: `rustup install nightly`
   - Add WebAssembly target: `rustup target add wasm32-unknown-unknown --toolchain nightly`

2. **Substrate Development Environment**
   ```bash
   # For Ubuntu/Debian
   sudo apt update
   sudo apt install -y build-essential git clang curl libssl-dev llvm libudev-dev pkg-config make
   
   # For macOS
   brew install openssl cmake llvm
   ```

3. **Node.js and Yarn**
   - Node.js v18+ required for Polkadot.js apps
   - Yarn for package management

4. **Docker** (optional but recommended)
   - For running local networks without full environment setup
   - Install from https://docs.docker.com/get-docker/

### Project Setup

1. **Clone the Substrate Node Template**
   ```bash
   git clone https://github.com/substrate-developer-hub/substrate-node-template
   cd substrate-node-template
   # Checkout the latest polkadot-v1.0.0 branch
   git checkout polkadot-v1.0.0
   ```

2. **Create Your Vital Records Pallet**
   ```bash
   cd pallets
   mkdir vital-records
   cd vital-records
   # Initialize basic pallet structure
   ```

3. **Add dependencies to your pallet's Cargo.toml**
   ```toml
   [dependencies]
   codec = { package = "parity-scale-codec", version = "3.0.0", features = ["derive"], default-features = false }
   scale-info = { version = "2.9.0", default-features = false, features = ["derive"] }
   frame-support = { default-features = false, version = "21.0.0" }
   frame-system = { default-features = false, version = "21.0.0" }
   frame-benchmarking = { default-features = false, version = "21.0.0", optional = true }
   sp-std = { default-features = false, version = "8.0.0" }
   sp-runtime = { default-features = false, version = "24.0.0" }
   sp-core = { default-features = false, version = "21.0.0" }
   sp-io = { default-features = false, version = "23.0.0" }
   ```

### Connecting to a Test Network

1. **Local Development Network**
   ```bash
   # Start a development chain
   cd substrate-node-template
   cargo run --release -- --dev
   ```

2. **Connecting to Polkadot Testnet (Westend)**
   ```bash
   # Using Polkadot binary
   polkadot --chain=westend --name="vital-records-tester"
   ```

3. **Using Polkadot.js Apps UI**
   - Visit https://polkadot.js.org/apps/
   - Connect to your local node (default: ws://127.0.0.1:9944)
   - Or connect to Westend testnet from the network dropdown

## Step 1: Marriage License Issuance

### Learning Goals
- Basic pallet structure
- Storage items and mappings
- Basic dispatchable functions
- Events and errors
- Simple access control

### Implementation

```rust
#![cfg_attr(not(feature = "std"), no_std)]

pub use pallet::*;

#[frame_support::pallet]
pub mod pallet {
    use frame_support::{
        dispatch::DispatchResult,
        pallet_prelude::*,
        traits::Time,
    };
    use frame_system::pallet_prelude::*;
    use sp_std::vec::Vec;

    #[pallet::config]
    pub trait Config: frame_system::Config {
        type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
        type TimeProvider: Time;
    }

    #[pallet::pallet]
    #[pallet::generate_store(pub(super) trait Store)]
    pub struct Pallet<T>(_);

    // Define the marriage license structure
    #[pallet::storage]
    #[pallet::getter(fn marriage_licenses)]
    pub type MarriageLicenses<T: Config> = StorageMap<
        _,
        Blake2_128Concat,
        T::Hash,  // License ID
        MarriageLicense<T>,
        OptionQuery,
    >;

    // Track licenses by account
    #[pallet::storage]
    #[pallet::getter(fn account_licenses)]
    pub type AccountLicenses<T: Config> = StorageMap<
        _,
        Blake2_128Concat,
        T::AccountId,
        Vec<T::Hash>,  // List of license IDs
        ValueQuery,
    >;

    // Track authorized registrars
    #[pallet::storage]
    #[pallet::getter(fn registrars)]
    pub type Registrars<T: Config> = StorageMap<
        _,
        Blake2_128Concat,
        T::AccountId,
        bool,
        ValueQuery,
    >;

    // Marriage license data structure
    #[pallet::storage]
    #[pallet::unbounded]
    pub type MarriageLicense<T: Config> = StorageValue<
        _,
        (
            (T::AccountId, Vec<u8>),  // Partner 1 (AccountId, Name)
            (T::AccountId, Vec<u8>),  // Partner 2 (AccountId, Name)
            Vec<u8>,                  // License details (county, state)
            T::AccountId,             // Registrar who issued the license
            MomentOf<T>,              // Issuance date
            LicenseStatus,            // Current status
        ),
    >;

    pub type MomentOf<T> = <<T as Config>::TimeProvider as Time>::Moment;

    #[derive(Clone, Encode, Decode, Eq, PartialEq, RuntimeDebug, TypeInfo, MaxEncodedLen)]
    pub enum LicenseStatus {
        Pending,
        Issued,
        Revoked,
    }

    #[pallet::event]
    #[pallet::generate_deposit(pub(super) fn deposit_event)]
    pub enum Event<T: Config> {
        RegistrarAdded(T::AccountId),
        RegistrarRemoved(T::AccountId),
        MarriageLicenseRequested(T::Hash, T::AccountId, T::AccountId),
        MarriageLicenseIssued(T::Hash),
        MarriageLicenseRevoked(T::Hash),
    }

    #[pallet::error]
    pub enum Error<T> {
        NotAuthorized,
        LicenseNotFound,
        InvalidStatus,
        AlreadyMarried,
    }

    #[pallet::call]
    impl<T: Config> Pallet<T> {
        /// Add a new registrar (restricted to governance or existing registrars)
        #[pallet::weight(10_000)]
        pub fn add_registrar(
            origin: OriginFor<T>,
            registrar: T::AccountId,
        ) -> DispatchResult {
            let who = ensure_signed(origin)?;
            
            // For simplicity, we'll allow any existing registrar to add another
            // In a real system, this would be restricted further
            ensure!(Self::registrars(who), Error::<T>::NotAuthorized);
            
            Registrars::<T>::insert(&registrar, true);
            Self::deposit_event(Event::RegistrarAdded(registrar));
            Ok(())
        }

        /// Initialize first registrar (should be called only once by sudo)
        #[pallet::weight(10_000)]
        pub fn init_first_registrar(
            origin: OriginFor<T>,
            registrar: T::AccountId,
        ) -> DispatchResult {
            ensure_root(origin)?;
            
            Registrars::<T>::insert(&registrar, true);
            Self::deposit_event(Event::RegistrarAdded(registrar));
            Ok(())
        }

        /// Request a marriage license (both partners must sign)
        #[pallet::weight(10_000)]
        pub fn request_marriage_license(
            origin: OriginFor<T>,
            partner1: (T::AccountId, Vec<u8>),
            partner2: (T::AccountId, Vec<u8>),
            license_details: Vec<u8>,
        ) -> DispatchResult {
            let who = ensure_signed(origin)?;
            
            // Simple check that requester is one of the partners
            ensure!(who == partner1.0 || who == partner2.0, Error::<T>::NotAuthorized);
            
            // Generate a unique license ID
            let license_id = T::Hashing::hash_of(&(partner1.0.clone(), partner2.0.clone(), frame_system::Pallet::<T>::block_number()));
            
            // Create pending license
            let now = T::TimeProvider::now();
            let license = (
                partner1.clone(),
                partner2.clone(),
                license_details,
                T::AccountId::default(), // will be set when issued
                now,
                LicenseStatus::Pending,
            );
            
            MarriageLicenses::<T>::insert(license_id, license);
            
            // Add license ID to both partners' accounts
            let mut partner1_licenses = Self::account_licenses(partner1.0.clone());
            partner1_licenses.push(license_id);
            AccountLicenses::<T>::insert(partner1.0, partner1_licenses);
            
            let mut partner2_licenses = Self::account_licenses(partner2.0.clone());
            partner2_licenses.push(license_id);
            AccountLicenses::<T>::insert(partner2.0, partner2_licenses);
            
            Self::deposit_event(Event::MarriageLicenseRequested(license_id, partner1.0, partner2.0));
            Ok(())
        }

        /// Issue a pending marriage license (only registrars)
        #[pallet::weight(10_000)]
        pub fn issue_marriage_license(
            origin: OriginFor<T>,
            license_id: T::Hash,
        ) -> DispatchResult {
            let who = ensure_signed(origin)?;
            
            // Ensure caller is an authorized registrar
            ensure!(Self::registrars(who.clone()), Error::<T>::NotAuthorized);
            
            // Get the license
            let mut license = Self::marriage_licenses(license_id).ok_or(Error::<T>::LicenseNotFound)?;
            
            // Ensure license is in pending status
            ensure!(license.5 == LicenseStatus::Pending, Error::<T>::InvalidStatus);
            
            // Update registrar and status
            license.3 = who;
            license.5 = LicenseStatus::Issued;
            
            // Save updated license
            MarriageLicenses::<T>::insert(license_id, license);
            
            Self::deposit_event(Event::MarriageLicenseIssued(license_id));
            Ok(())
        }

        /// Revoke a marriage license (only registrars)
        #[pallet::weight(10_000)]
        pub fn revoke_marriage_license(
            origin: OriginFor<T>,
            license_id: T::Hash,
        ) -> DispatchResult {
            let who = ensure_signed(origin)?;
            
            // Ensure caller is an authorized registrar
            ensure!(Self::registrars(who.clone()), Error::<T>::NotAuthorized);
            
            // Get the license
            let mut license = Self::marriage_licenses(license_id).ok_or(Error::<T>::LicenseNotFound)?;
            
            // Update status
            license.5 = LicenseStatus::Revoked;
            
            // Save updated license
            MarriageLicenses::<T>::insert(license_id, license);
            
            Self::deposit_event(Event::MarriageLicenseRevoked(license_id));
            Ok(())
        }
    }
}
```

## Step 2: Birth Certificate Registration

For the next steps, please refer to the [full code kata document](./docs/code-kata-full.md) which includes all five implementations in detail.

## Advanced Features

The kata also includes advanced features such as:

### Real-world Events Simulation
- Natural disasters requiring emergency record validation
- Identity theft situations requiring record verification
- Mass record updates following redistricting events
- Election day simulation with high-volume voter verification

### Privacy-Preserving Features
- Selective disclosure mechanisms
- Data minimization techniques
- ZK-proofs for age verification without revealing birthdate
- Privacy-preserving voting registration

### Real Government Compliance
- GDPR-compliant data management
- NIST digital identity guidelines
- Real ID Act compliance for driver's licenses
- Voting accessibility requirements (ADA, HAVA)

## Testing Instructions

Complete testing instructions are available in the [Testing Guide](./docs/guides/testing.md).

## Additional Resources

- [Troubleshooting Guide](./docs/guides/troubleshooting.md)
- [Deployment Instructions](./docs/guides/deployment.md)
- [Code Review Checklist](./docs/guides/code-review.md)
- [Glossary of Terms](./docs/glossary.md)

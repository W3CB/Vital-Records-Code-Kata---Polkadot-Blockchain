# Step-by-Step Testing Instructions for Vital Records Management Kata

> This guide provides comprehensive testing instructions for each component of the Vital Records Management system built on Polkadot.

- [Setting Up the Testing Environment](#setting-up-the-testing-environment)
- [Testing Marriage License Functions](#1-testing-marriage-license-functions)
- [Testing Birth Certificate Functions](#2-testing-birth-certificate-functions)
- [Testing Death Certificate Functions](#3-testing-death-certificate-functions)
- [Testing Driver's License Functions](#4-testing-drivers-license-functions)
- [Testing Voter Registration Functions](#5-testing-voter-registration-functions)
- [Testing Advanced Features](#6-testing-advanced-features)
- [Integration Testing](#integration-testing)
- [Testing with Large Data Sets](#testing-with-large-data-sets)
- [Testing Best Practices](#testing-best-practices)

## Setting Up the Testing Environment

Before starting with specific test cases, set up your testing environment:

```bash
# Ensure you're in the main directory of your pallet
cd substrate-node-template/pallets/vital-records

# Create a proper test structure in your lib.rs file or in a separate tests.rs file
# Make sure your pallet's Cargo.toml has the necessary test dependencies
```

### Basic Test Framework

Here's a template for your test module structure:

```rust
#[cfg(test)]
mod tests {
    use crate::{mock::*, Error, Event};
    use frame_support::{assert_noop, assert_ok, assert_err};
    use sp_runtime::traits::BlakeTwo256;
    use sp_core::H256;
    
    // Helper function to generate a random H256 hash
    fn random_hash() -> H256 {
        let mut hash = [0u8; 32];
        for i in 0..32 {
            hash[i] = (i as u8).wrapping_add(1);
        }
        H256::from(hash)
    }
    
    // Your test cases will go here
}
```

## 1. Testing Marriage License Functions

### Unit Tests for Marriage License

```rust
#[test]
fn test_init_first_registrar() {
    new_test_ext().execute_with(|| {
        let registrar_account = 1;
        
        // Test initialization of first registrar with root origin
        assert_ok!(VitalRecords::init_first_registrar(
            RuntimeOrigin::root(),
            registrar_account
        ));
        
        // Verify registrar was added
        assert!(VitalRecords::registrars(registrar_account));
        
        // Verify event was emitted
        System::assert_has_event(Event::RegistrarAdded(registrar_account).into());
    });
}

#[test]
fn test_add_registrar() {
    new_test_ext().execute_with(|| {
        let first_registrar = 1;
        let second_registrar = 2;
        
        // Initialize first registrar
        assert_ok!(VitalRecords::init_first_registrar(
            RuntimeOrigin::root(),
            first_registrar
        ));
        
        // Test adding a second registrar by the first one
        assert_ok!(VitalRecords::add_registrar(
            RuntimeOrigin::signed(first_registrar),
            second_registrar
        ));
        
        // Verify second registrar was added
        assert!(VitalRecords::registrars(second_registrar));
        
        // Try adding a registrar with unauthorized account (should fail)
        let unauthorized_account = 3;
        assert_noop!(
            VitalRecords::add_registrar(
                RuntimeOrigin::signed(unauthorized_account),
                4
            ),
            Error::<Test>::NotAuthorized
        );
    });
}

#[test]
fn test_marriage_license_lifecycle() {
    new_test_ext().execute_with(|| {
        let registrar = 1;
        let partner1_account = 2;
        let partner2_account = 3;
        
        // Setup registrar
        assert_ok!(VitalRecords::init_first_registrar(
            RuntimeOrigin::root(),
            registrar
        ));
        
        // Test requesting a marriage license
        let partner1_name = b"Alice".to_vec();
        let partner2_name = b"Bob".to_vec();
        let license_details = b"County: Exampleshire, State: Substrate".to_vec();
        
        assert_ok!(VitalRecords::request_marriage_license(
            RuntimeOrigin::signed(partner1_account),
            (partner1_account, partner1_name.clone()),
            (partner2_account, partner2_name.clone()),
            license_details.clone()
        ));
        
        // Find the license ID from events
        let license_id = match System::events().iter().find_map(|r| {
            if let Event::MarriageLicenseRequested(id, p1, p2) = &r.event {
                Some(id.clone())
            } else {
                None
            }
        }) {
            Some(id) => id,
            None => panic!("Marriage license request event not found"),
        };
        
        // Verify license was created with pending status
        let license = VitalRecords::marriage_licenses(license_id).unwrap();
        assert_eq!(license.5, LicenseStatus::Pending);
        
        // Test issuing the license
        assert_ok!(VitalRecords::issue_marriage_license(
            RuntimeOrigin::signed(registrar),
            license_id
        ));
        
        // Verify license status changed to issued
        let updated_license = VitalRecords::marriage_licenses(license_id).unwrap();
        assert_eq!(updated_license.5, LicenseStatus::Issued);
        
        // Verify event was emitted
        System::assert_has_event(Event::MarriageLicenseIssued(license_id).into());
        
        // Test revoking the license
        assert_ok!(VitalRecords::revoke_marriage_license(
            RuntimeOrigin::signed(registrar),
            license_id
        ));
        
        // Verify license status changed to revoked
        let revoked_license = VitalRecords::marriage_licenses(license_id).unwrap();
        assert_eq!(revoked_license.5, LicenseStatus::Revoked);
    });
}
```

### Mock Data for Marriage License Testing

```rust
// In your mock.rs file or test module:

// Sample partner data
pub const PARTNER1_ACCOUNT: u64 = 2;
pub const PARTNER2_ACCOUNT: u64 = 3;
pub const PARTNER1_NAME: &[u8] = b"Alice Smith";
pub const PARTNER2_NAME: &[u8] = b"Bob Johnson";
pub const LICENSE_DETAILS: &[u8] = b"County: Exampleshire, State: Substrate, Date: 2025-01-15";

// Sample registrar data
pub const REGISTRAR1_ACCOUNT: u64 = 1;
pub const REGISTRAR2_ACCOUNT: u64 = 4;
```

## 2. Testing Birth Certificate Functions

See the [complete testing guide](./testing-guide-full.md) for detailed testing of birth certificates, death certificates, driver's licenses, and voter registration.

## Integration Testing

After testing individual components, you should perform integration tests to ensure they work together correctly:

```rust
#[test]
fn test_full_lifecycle_integration() {
    new_test_ext().execute_with(|| {
        let registrar = 1;
        let person_account = 2;
        
        // 1. Setup registrar
        assert_ok!(VitalRecords::init_first_registrar(
            RuntimeOrigin::root(),
            registrar
        ));
        
        // 2. Issue birth certificate
        let person_name = b"Test Person".to_vec();
        let current_time = System::block_timestamp();
        let birth_time = current_time - 25 * 365 * 24 * 60 * 60 * 1000; // 25 years ago
        
        assert_ok!(VitalRecords::request_birth_certificate(
            RuntimeOrigin::signed(registrar),
            person_name.clone(),
            birth_time,
            b"Birth Hospital".to_vec(),
            vec![]
        ));
        
        let birth_cert_id = match System::events().iter().find_map(|r| {
            if let Event::BirthCertificateRequested(id, _) = &r.event {
                Some(id.clone())
            } else {
                None
            }
        }) {
            Some(id) => id,
            None => panic!("Birth certificate request event not found"),
        };
        
        assert_ok!(VitalRecords::issue_birth_certificate(
            RuntimeOrigin::signed(registrar),
            birth_cert_id,
            Some(person_account)
        ));
        
        // 3. Issue driver's license
        assert_ok!(VitalRecords::issue_driver_license(
            RuntimeOrigin::signed(registrar),
            person_account,
            person_name.clone(),
            birth_cert_id,
            LicenseClass::C,
            vec![],
            vec![],
            1825,
            b"DMV Test Authority".to_vec()
        ));
        
        let license_id = match System::events().iter().find_map(|r| {
            if let Event::DriverLicenseIssued(id, account) = &r.event {
                if *account == person_account {
                    Some(id.clone())
                } else {
                    None
                }
            } else {
                None
            }
        }) {
            Some(id) => id,
            None => panic!("Driver license issuance event not found"),
        };
        
        // 4. Register to vote
        // First add a district
        let district_id = b"district-01".to_vec();
        assert_ok!(VitalRecords::add_election_district(
            RuntimeOrigin::signed(registrar),
            district_id.clone(),
            b"First District".to_vec(),
            b"Region".to_vec(),
            DistrictType::County
        ));
        
        assert_ok!(VitalRecords::register_voter(
            RuntimeOrigin::signed(person_account),
            birth_cert_id,
            b"123 Test St, City".to_vec(),
            district_id.clone()
        ));
        
        assert_ok!(VitalRecords::approve_voter_registration(
            RuntimeOrigin::signed(registrar),
            person_account
        ));
        
        // 5. Simulate death and verify effects
        assert_ok!(VitalRecords::request_death_certificate(
            RuntimeOrigin::signed(registrar),
            Some(person_account),
            person_name.clone(),
            Some(birth_cert_id),
            None,
            System::block_timestamp(),
            b"Hospital".to_vec(),
            b"Natural causes".to_vec(),
            b"Dr. Examiner".to_vec()
        ));
        
        let death_cert_id = match System::events().iter().find_map(|r| {
            if let Event::DeathCertificateRequested(id, _) = &r.event {
                Some(id.clone())
            } else {
                None
            }
        }) {
            Some(id) => id,
            None => panic!("Death certificate request event not found"),
        };
        
        assert_ok!(VitalRecords::issue_death_certificate(
            RuntimeOrigin::signed(registrar),
            death_cert_id
        ));
        
        // Process death certificate effects
        assert_ok!(VitalRecords::process_death_certificate_effects(
            RuntimeOrigin::signed(registrar),
            death_cert_id
        ));
        
        // Verify driver's license is revoked
        let revoked_license = VitalRecords::driver_licenses(license_id).unwrap();
        assert_eq!(revoked_license.status, LicenseStatus::Revoked);
        
        // Verify voter registration is removed
        let removed_voter = VitalRecords::voter_registrations(person_account).unwrap();
        assert_eq!(removed_voter.status, VoterStatus::Removed);
        
        // Verify person is removed from district voters
        let district_voters = VitalRecords::district_voters(district_id);
        assert!(!district_voters.contains(&person_account));
    });
}
```

## Mock Data for Integration Testing

For convenience, we've provided helper functions for creating test data:

```rust
// Create test marriage
pub fn create_test_marriage(
    partner1: u64,
    partner2: u64,
    registrar: u64
) -> Result<H256, &'static str> {
    let partner1_name = format!("Partner {}", partner1).into_bytes();
    let partner2_name = format!("Partner {}", partner2).into_bytes();
    
    // Request the license
    assert_ok!(VitalRecords::request_marriage_license(
        RuntimeOrigin::signed(partner1),
        (partner1, partner1_name),
        (partner2, partner2_name),
        b"Test County, Test State".to_vec()
    ));
    
    // Find the license ID from events
    let license_id = match System::events().iter().find_map(|r| {
        if let Event::MarriageLicenseRequested(id, p1, p2) = &r.event {
            if *p1 == partner1 && *p2 == partner2 {
                Some(id.clone())
            } else {
                None
            }
        } else {
            None
        }
    }) {
        Some(id) => id,
        None => return Err("Marriage license request event not found"),
    };
    
    // Issue the license
    assert_ok!(VitalRecords::issue_marriage_license(
        RuntimeOrigin::signed(registrar),
        license_id
    ));
    
    Ok(license_id)
}

// Create test birth certificate
pub fn create_test_birth_certificate(
    person_account: u64,
    age_in_years: u64,
    registrar: u64
) -> Result<H256, &'static str> {
    let person_name = format!("Person {}", person_account).into_bytes();
    let birth_time = birth_time_for_age(age_in_years);
    
    // Request certificate
    assert_ok!(VitalRecords::request_birth_certificate(
        RuntimeOrigin::signed(registrar),
        person_name,
        birth_time,
        b"Test Hospital".to_vec(),
        vec![]
    ));
    
    // Find certificate ID
    let cert_id = match System::events().iter().find_map(|r| {
        if let Event::BirthCertificateRequested(id, _) = &r.event {
            Some(id.clone())
        } else {
            None
        }
    }) {
        Some(id) => id,
        None => return Err("Birth certificate request event not found"),
    };
    
    // Issue certificate
    assert_ok!(VitalRecords::issue_birth_certificate(
        RuntimeOrigin::signed(registrar),
        cert_id,
        Some(person_account)
    ));
    
    Ok(cert_id)
}
```

## Testing with Large Data Sets

For stress testing the system with large volumes of data:

```rust
#[test]
fn test_large_scale_data() {
    new_test_ext().execute_with(|| {
        let registrar = REGISTRAR_ACCOUNT;
        
        // Setup registrar
        assert_ok!(VitalRecords::init_first_registrar(
            RuntimeOrigin::root(),
            registrar
        ));
        
        // Create 5 districts
        let mut districts = Vec::new();
        for i in 1..=5 {
            districts.push(create_test_district(i, registrar));
        }
        
        // Create 100 birth certificates
        let mut birth_certs = Vec::new();
        let mut people = Vec::new();
        
        for i in 10..110 {
            let person = i;
            people.push(person);
            
            let age = 20 + (i % 50); // Ages from 20 to 69
            let birth_cert_id = create_test_birth_certificate(person, age, registrar)
                .expect("Failed to create birth certificate");
                
            birth_certs.push((person, birth_cert_id));
        }
        
        // Create 80 driver licenses (80% of people)
        let mut licenses = Vec::new();
        for i in 0..80 {
            let (person, birth_cert_id) = birth_certs[i];
            
            if let Ok(license_id) = create_test_driver_license(person, birth_cert_id, registrar) {
                licenses.push((person, license_id));
            }
        }
        
        // Register 60 voters (60% of people)
        for i in 0..60 {
            let (person, birth_cert_id) = birth_certs[i];
            let district_idx = i % districts.len();
            
            let _ = create_test_voter_registration(
                person,
                birth_cert_id,
                districts[district_idx].clone(),
                registrar
            );
        }
        
        // Create 10 death certificates (10% of people)
        for i in 0..10 {
            let (person, birth_cert_id) = birth_certs[i];
            let person_name = format!("Person {}", person).into_bytes();
            
            assert_ok!(VitalRecords::request_death_certificate(
                RuntimeOrigin::signed(registrar),
                Some(person),
                person_name,
                Some(birth_cert_id),
                None,
                System::block_timestamp(),
                b"Test Hospital".to_vec(),
                b"Natural causes".to_vec(),
                b"Dr. Examiner".to_vec()
            ));
            
            // Find the certificate ID
            let death_cert_id = match System::events().iter().find_map(|r| {
                if let Event::DeathCertificateRequested(id, name) = &r.event {
                    Some(id.clone())
                } else {
                    None
                }
            }) {
                Some(id) => id,
                None => continue,
            };
            
            // Issue the certificate
            assert_ok!(VitalRecords::issue_death_certificate(
                RuntimeOrigin::signed(registrar),
                death_cert_id
            ));
            
            // Process effects
            assert_ok!(VitalRecords::process_death_certificate_effects(
                RuntimeOrigin::signed(registrar),
                death_cert_id
            ));
        }
        
        // Run election day simulation
        assert_ok!(VitalRecords::start_disaster_simulation(
            RuntimeOrigin::root(),
            b"election_day".to_vec(),
            vec![]
        ));
        
        assert_ok!(VitalRecords::simulate_election_day(
            RuntimeOrigin::root(),
            75, // 75% turnout
            districts.clone()
        ));
        
        assert_ok!(VitalRecords::end_simulation(
            RuntimeOrigin::root(),
            true
        ));
    });
}
```

## Testing Best Practices

As you work through these tests, keep these best practices in mind:

### 1. Test Isolation

Each test should be independent and not rely on state created by other tests. Use the `new_test_ext()` function to start each test with a clean state.

```rust
#[test]
fn my_isolated_test() {
    new_test_ext().execute_with(|| {
        // Test code here starts with fresh state
    });
}
```

### 2. Complete Coverage

Aim to test:
- ✅ Success paths
- ✅ All error conditions
- ✅ Edge cases
- ✅ Boundary values
- ✅ Cross-component interactions

### 3. Descriptive Names

Use clear test names that describe what is being tested:

```rust
#[test]
fn test_registrar_cannot_issue_license_to_underage_person() {
    // Test code here
}
```

### 4. Arrange-Act-Assert Pattern

Structure tests with clear phases:

```rust
#[test]
fn test_something() {
    new_test_ext().execute_with(|| {
        // ARRANGE - set up test conditions
        let registrar = 1;
        assert_ok!(VitalRecords::init_first_registrar(
            RuntimeOrigin::root(),
            registrar
        ));
        
        // ACT - perform the action being tested
        let result = VitalRecords::some_function(
            RuntimeOrigin::signed(registrar),
            some_parameters
        );
        
        // ASSERT - verify the expected outcome
        assert_ok!(result);
        // Additional assertions...
    });
}
```

### 5. Test Assertions

Use the proper assertion macros:

- `assert_ok!(result)` - Verify operation succeeded
- `assert_noop!(call, error)` - Verify operation fails with specific error
- `assert_err!(call, error)` - Same as `assert_noop!` but actually executes the call
- `assert!(condition)` - Verify a condition is true
- `assert_eq!(a, b)` - Verify two values are equal

### 6. Weight Testing

Test that weight calculations are accurate:

```rust
#[test]
fn check_weights_are_correct() {
    // Set up test scenario with worst-case inputs
    // Measure execution time/resources
    // Compare with calculated weights
}
```

## Running the Tests

To run all tests for your pallet:

```bash
# Run all tests
cargo test -p pallet-vital-records

# Run a specific test
cargo test -p pallet-vital-records test_marriage_license_lifecycle

# Run tests with verbose output
cargo test -p pallet-vital-records -- --nocapture
```

## Debugging Failed Tests

When tests fail, use these strategies to identify the issue:

1. Run with `--nocapture` to see all output
2. Add debug prints with `println!` or `frame_support::debug::debug!`
3. Break complex tests into smaller ones
4. Check events to verify they were emitted
5. Verify storage values at each step
6. Check error conditions with `assert_noop!`

## Next Steps

After completing the unit and integration tests, consider:

1. [Benchmarking](https://docs.substrate.io/reference/how-to-guides/weights/calculate-weights/) your pallet for accurate weight calculations
2. Setting up [try-runtime](https://docs.substrate.io/test/try-runtime/) tests for migrations
3. Adding property-based testing with [quickcheck](https://github.com/BurntSushi/quickcheck)
4. Creating end-to-end tests with a running node

For more information, see the [Substrate Testing Guide](https://docs.substrate.io/test/).

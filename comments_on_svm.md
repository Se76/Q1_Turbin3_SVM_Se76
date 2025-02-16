# 1. Description of my focusing area

### Disclaimer: I will be firstly very high level for a little moment to describe infrustructure in general, then I will come to something more specific. 

I chose Runtime and SVM Api. In the part of runtime we are having bank, so bassically bank is state for the current slot on which voters/validators should vote later on, runtime(bank) will call SVM to execute transactions, smart contracts(programs) and their instructions. In the SVM in general will be made two steps of the transaction pipeline load accounts and execution.

## Specific part:

The main method of the whole SVM Api is ```load_and_execute_sanitized_transactions()``` as the name says it loads needed program accounts, built-in programs also and all kinds of different accounts that are needed for execution and then depends on the result of loads execute txs which accounts are succesfully loaded and then writes changes to the current state.

## Details of `load_and_execute_sanitized_transactions()`

#### Parameters
This method takes the following parameters:

- **`callbacks`**: A reference to an implementation of `TransactionProcessingCallback`. This helps load accounts by ensuring they have the required attributes (e.g., owner, data, lamports, executable flag).
- **`sanitized_txs`**: An array of transactions implementing the `SVMTransaction` trait. These transactions are processed in this batch.
- **`check_results`**: A vector of `TransactionCheckResult`, containing pre-check results for the transactions.
- **`environment`**: The runtime environment used to process the batch of transactions.
- **`config`**: The configuration settings for transaction processing.

The method returns a `LoadAndExecuteSanitizedTransactionsOutput` struct, which includes error metrics, execution timings, and the results of the processing.

#### Execution Steps

1. **Initialization**:
   - Performs assertions to ensure that `sanitized_txs` and `check_results` are equal.
   - Initializes default metrics (`TransactionErrorMetrics`), timings (`ExecuteTimings`), and processing results.

2. **Loading Programs**:
   - Filters executable program accounts using `filter_executable_program_accounts` and adds them to a hashmap.
   - Loads missing program accounts into the program cache with `replenish_program_cache`. If the cache exceeds its limit, processing halts with an error.

3. **Account Loader Creation**:
   - Calculates the required capacity for accounts based on transaction account keys.
   - Creates an `AccountLoader` to efficiently manage the loading of accounts.

4. **Feature Flags**:
   - Checks active features like `enable_transaction_loading_failure_fees` to determine specific processing rules.

5. **Processing Transactions**:
   - Iterates through each transaction in `sanitized_txs`:
     - Validates conditions such as unique nonces and fees from the fee-payer.
     - Loads necessary accounts and verifies them.
     - Executes the transaction if all conditions are met, updating balances and account states.
   - Tracks metrics for validation, loading, and execution times.

6. **Final Results**:
   - Logs detailed timings for all steps and updates the `ExecuteTimings` object.
   - Returns a `LoadAndExecuteSanitizedTransactionsOutput` struct containing:
     - **Error Metrics**: Data on errors encountered during processing.
     - **Execution Timings**: Detailed timings for each phase.
     - **Processing Results**: The outcomes for each transaction, including successes and specific errors.

---

# 2.Anotated code from Agave Validator

> For this purpose I created a fork of Agave Validator and commented this part out, you can watch it [here](https://github.com/Se76/agave-fork-comments/blob/master/svm/src/transaction_processor.rs#L332)

> And I took more than 30 lines because the function was longer and I thought that it would make more sense to work on the whole function

```rust
/// Main entrypoint to the SVM.
    pub fn load_and_execute_sanitized_transactions<CB: TransactionProcessingCallback>( // literally main method, 
                                                                                       //<CB: TransactionProcessingCallback> 
                                                                                       // -> makes generic type CB over TransactionProcessingCallback
        &self, // reference to self (TransactionBatchProcessor)
        callbacks: &CB, // callbacks, type reference to CB, so bassicly this parameter should be an implementation of TransactionProcessingCallback !!!is very important to load accounts, bassicly each account should have owner, its data (amount of lamports, address, executable or not, etc) to be loaded -> for trnsactions to be proccessed 
        sanitized_txs: &[impl SVMTransaction], // reference to an array of variables that implement the SVMTransaction trait, transactions that will be processed
        check_results: Vec<TransactionCheckResult>, // vector of TransactionCheckResult, results of transaction checks
        environment: &TransactionProcessingEnvironment, // runtime environment for transaction batch processing
        config: &TransactionProcessingConfig, // config 
    ) -> LoadAndExecuteSanitizedTransactionsOutput {  // returns LoadAndExecuteSanitizedTransactionsOutput struct (error_metrics, execute_timings, processing_results)
        // If `check_results` does not have the same length as `sanitized_txs`,
        // transactions could be truncated as a result of `.iter().zip()` in
        // many of the below methods.
        // See <https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.zip>.
        debug_assert_eq!(    
            sanitized_txs.len(),      // macro than ensures that check_results.len() == sanitized_txs.len(), if they are not equal it will panic and print a message
            check_results.len(),      // already well described above
            "Length of check_results does not match length of sanitized_txs" // message that will be printed in case if check_results.len() == sanitized_txs.len() is not true
        );

        // Initialize metrics.
        let mut error_metrics = TransactionErrorMetrics::default(); // initializes default of this struct
        let mut execute_timings = ExecuteTimings::default();    // initializes default of this struct
        let mut processing_results = Vec::with_capacity(sanitized_txs.len()); // creates a vector with length (ccapacity) of sanitized_txs.len()

        let native_loader = native_loader::id(); // just an pubkey for native loader
        let (program_accounts_map, filter_executable_us) = measure_us!({ // custom macro measure_us measures execution time
                                                                                                             // and returns tuple of that what will be returned inside of macro and ex. time
                                                                                                             // and measured time in ms, in our case filter_executable_us - time in ms
                                                                                                             // and program_accounts_map - hash map, for executable accounts

            let mut program_accounts_map = Self::filter_executable_program_accounts(  // filters all executable program accounts and returns hashmap with these accounts
                callbacks,
                sanitized_txs,
                &check_results,
                PROGRAM_OWNERS, // program that owns all executable programs
            );
            for builtin_program in self.builtin_program_ids.read().unwrap().iter() { // adds/pushes/inserts already built in programs to the program_accounts_map
                program_accounts_map.insert(*builtin_program, (&native_loader, 0));
            }
            program_accounts_map // returns program_accounts_map with executable accounts and the measured time will be stopped as this will be returned inside of macro
        });

        let (program_cache_for_tx_batch, program_cache_us) = measure_us!({ // custom macro measure_us measures execution time
                                                                                                        // and returns tuple of that what will be returned inside of macro and ex. time
            let program_cache_for_tx_batch = self.replenish_program_cache( // checks if some program accounts are missing in the cache
                                                                           // if yes, it loads them and returnes ProgramCacheForTxBatch, 
                                                                           // where inside are all programs needed for execution
                callbacks,
                &program_accounts_map,
                &mut execute_timings, // mutable reference
                config.check_program_modification_slot,
                config.limit_to_load_programs,
            );

            if program_cache_for_tx_batch.hit_max_limit {  // if cache is reached max limit of storage then error will be returned
                return LoadAndExecuteSanitizedTransactionsOutput {
                    error_metrics,
                    execute_timings,
                    processing_results: (0..sanitized_txs.len()) // makes a range of length of sanitized_txs, iterates each with each index and for each retruns an error in processing_results
                        .map(|_| Err(TransactionError::ProgramCacheHitMaxLimit)) // just small closure that takes whatever _ (index in this case) and returns TransactionError::ProgramCacheHitMaxLimit
                        .collect(), // all iterators are "lazy" that is why we should always collect each iterator 
                };
            }

            program_cache_for_tx_batch // if cache isn't reached max limit of storage then it will be returned
        });

        // Determine a capacity for the internal account cache. This
        // over-allocates but avoids ever reallocating, and spares us from
        // deduplicating the account keys lists.
        let account_keys_in_batch = sanitized_txs.iter().map(|tx| tx.account_keys().len()).sum(); // pretty well described above :)
                                                                                                                               // rust concept: iterator over sanitized txs will be created
                                                                                                                               // then each transaction will be mapped and with the help of closure
                                                                                                                               //will be returned amount of all account keys in particular transaction
                                                                                                                               // at the end it will be summed and will be returned

        // Create the account loader, which wraps all external account fetching.
        let mut account_loader = AccountLoader::new_with_account_cache_capacity( // creates account loader with the capacity that we already calculated before
                                                                                                        // it "loads all accounts that are needed for execution of transactions"
            config.account_overrides,
            program_cache_for_tx_batch,
            program_accounts_map,
            callbacks,
            environment.feature_set.clone(),
            account_keys_in_batch,
        );

        let enable_transaction_loading_failure_fees = environment
            .feature_set                                // feature set, so whether there are some features turned on or not
            .is_active(&enable_transaction_loading_failure_fees::id()); // if these features are active than enable_transaction_loading_failure_fees will be true

        let (mut validate_fees_us, mut load_us, mut execution_us): (u64, u64, u64) = (0, 0, 0); // initializes default timings for validation, loading and execution

        // Validate, execute, and collect results from each transaction in order.
        // With SIMD83, transactions must be executed in order, because transactions
        // in the same batch may modify the same accounts. Transaction order is
        // preserved within entries written to the ledger.
        for (tx, check_result) in sanitized_txs.iter().zip(check_results) { // just a for loop that iterates over sanitized_txs (every single tx) 
                                                                                                                        // and check_results (corresponding to the tx result of checks)
            let (validate_result, single_validate_fees_us) = // measure_us -> validate_result and single_validate_fees_us
                measure_us!(check_result.and_then(|tx_details| {     // if err -> return err, if ok -> continue with closure
                    Self::validate_transaction_nonce_and_fee_payer( // ensures that transaction is nor repeating and that fee payer is provided and has enough funds
                        &mut account_loader, // mutable reference
                        tx,
                        tx_details,
                        &environment.blockhash,
                        environment.fee_lamports_per_signature,
                        environment
                            .rent_collector
                            .unwrap_or(&RentCollector::default()), // if rent_collector is not provided then use default
                        &mut error_metrics, // mutable reference
                    )
                }));
            validate_fees_us = validate_fees_us.saturating_add(single_validate_fees_us); // it will add time that it took for one transaction to validate fees to the sum of time 
                                                                                         // that it took for all transactions (0 by default)


            // load_transaction actually uses another function (bassicly is just a wrapper) called load_transaction_accounts which 
            // loads all accounts that are needed for execution of transactions, makes some additional checks and returns Result<LoadedTransactionAccounts> 
            // and the wrapper function load_transaction handles errors and returns an enum TransactionLoadResult
            let (load_result, single_load_us) = measure_us!(load_transaction(  // measure_us -> load_result and single_load_us
                &mut account_loader,  // mutable reference
                tx,
                validate_result,
                &mut error_metrics,  // mutable reference
                environment
                    .rent_collector
                    .unwrap_or(&RentCollector::default()), // if rent_collector is not provided then use default
            ));
            load_us = load_us.saturating_add(single_load_us);  // calculates time that was used for loading, simillar to validate_fees_us

            // exactly the execution of the transaction / processing 
            let (processing_result, single_execution_us) = measure_us!(match load_result { // measure_us -> processing_result and single_execution_us
                // it matches on the enum TransactionLoadResult
                TransactionLoadResult::NotLoaded(err) => Err(err), // if there was an error than return same error
                // FeesOnly is kind of tricky, as fas as I understood it is the case if the transaction fails during loading
                // and it is already too far and the fees should be charged though it failed
                TransactionLoadResult::FeesOnly(fees_only_tx) => {  
                    if enable_transaction_loading_failure_fees {  // if the feature is enabled than it will be true
                        // Update loaded accounts cache with nonce and fee-payer
                        account_loader
                            .update_accounts_for_failed_tx(tx, &fees_only_tx.rollback_accounts);

                        Ok(ProcessedTransaction::FeesOnly(Box::new(fees_only_tx)))
                    } else {
                        Err(fees_only_tx.load_error) // if the feature is not enabled than return an error
                    }
                }
                TransactionLoadResult::Loaded(loaded_transaction) => {  // if programs, accounts and transaction were loaded successfully
                    // the transaction will be executed, all the instrcutions will be executed and all the balances will be changed
                    let executed_tx = self.execute_loaded_transaction(
                        callbacks,
                        tx,
                        loaded_transaction,
                        &mut execute_timings, // mutable reference
                        &mut error_metrics, // mutable reference
                        &mut account_loader.program_cache, // mutable reference
                        environment,
                        config,
                    );

                    // Update loaded accounts cache with account states which might have changed.
                    // Also update local program cache with modifications made by the transaction,
                    // if it executed successfully.
                    account_loader.update_accounts_for_executed_tx(tx, &executed_tx);

                    Ok(ProcessedTransaction::Executed(Box::new(executed_tx)))
                }
            });
            execution_us = execution_us.saturating_add(single_execution_us); // measure time that was used for execution

            processing_results.push(processing_result); // push the result to the vector of results
        }

        // Skip eviction when there's no chance this particular tx batch has increased the size of
        // ProgramCache entries. Note that loaded_missing is deliberately defined, so that there's
        // still at least one other batch, which will evict the program cache, even after the
        // occurrences of cooperative loading.
        if account_loader.program_cache.loaded_missing
            || account_loader.program_cache.merged_modified
        {
            const SHRINK_LOADED_PROGRAMS_TO_PERCENTAGE: u8 = 90;
            self.program_cache
                .write()
                .unwrap()
                .evict_using_2s_random_selection(
                    Percentage::from(SHRINK_LOADED_PROGRAMS_TO_PERCENTAGE),
                    self.slot,
                );
        }

        // logs
        debug!(
            "load: {}us execute: {}us txs_len={}",
            load_us,
            execution_us,
            sanitized_txs.len(),
        );


        // writing all timings
        execute_timings
            .saturating_add_in_place(ExecuteTimingType::ValidateFeesUs, validate_fees_us);
        execute_timings
            .saturating_add_in_place(ExecuteTimingType::FilterExecutableUs, filter_executable_us);
        execute_timings
            .saturating_add_in_place(ExecuteTimingType::ProgramCacheUs, program_cache_us);
        execute_timings.saturating_add_in_place(ExecuteTimingType::LoadUs, load_us);
        execute_timings.saturating_add_in_place(ExecuteTimingType::ExecuteUs, execution_us);

        // returning the results
        LoadAndExecuteSanitizedTransactionsOutput {
            error_metrics,
            execute_timings,
            processing_results,
        }
    }
```

## What is it doing?

I have already desciribed everything in the code and [above](https://github.com/Se76/Q1_Turbin3_SVM_Se76/blob/main/comments_on_svm.md#details-of-load_and_execute_sanitized_transactions) 

## How can it be made better?

### I would underline two points

- Documentation
  - more comments for the explination
  - more exact information on internet
  - complete educational video on SVM

 - To add additional function that will make readability of code higher

> Code before

```rust
...
if program_cache_for_tx_batch.hit_max_limit {
    return LoadAndExecuteSanitizedTransactionsOutput {
        error_metrics,
        execute_timings,
        processing_results: (0..sanitized_txs.len())
            .map(|_| Err(TransactionError::ProgramCacheHitMaxLimit))
            .collect(),
    };
}
...
```

> Code after

```rust
...
if program_cache_for_tx_batch.hit_max_limit {
    return Self::handle_program_cache_limit(sanitized_txs.len(), error_metrics, execute_timings);
}
...

impl<FG: ForkGraph> TransactionBatchProcessor<FG> {
   ...
   fn handle_program_cache_limit(
        tx_count: usize, 
        error_metrics: TransactionErrorMetrics, 
        execute_timings: ExecuteTimings
    ) -> LoadAndExecuteSanitizedTransactionsOutput {
        LoadAndExecuteSanitizedTransactionsOutput {
            error_metrics,
            execute_timings,
            processing_results: (0..tx_count)
                .map(|_| Err(TransactionError::ProgramCacheHitMaxLimit))
                .collect(),
        }
    }
   ...
}
```







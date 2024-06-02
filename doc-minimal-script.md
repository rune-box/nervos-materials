### A Minimal Script
As we have learned before, a Script that returns 0 is considered successful. The simplest CKB Script code in Rust looks like this:

```
pub fn program_entry() -> i8 {
    0
}
```

The simplest CKB Script code, often called `always-success`, always returns 0 as its return code. You might think this simplest Script code is also the dumbest one because if you use this as your Lock Script, your tokens could be taken by anyone.

However, this simplest Script proves useful in a development environment. When testing Scripts on a local blockchain for your dApp, you might allow the testing Cells to have an `always-success` Lock Script to simplify your testing workflow.

Despite its utility, the `always-success` Script is not very interesting due to its simplicity.

Here we will start with a more interesting idea:
> Personally I dislike carrot. I do know that carrot is great from a nutritional point of view, but I still want to avoid it due to the taste. Now what if I want to set a rule, that none of my Cells on CKB has data that begin with the word "carrot"?

Let’s write a Script code to ensure this.

### Use CKB Syscall with CKB-STD Library
To ensure the security of CKB Script, each Script has to run in an isolated environment that is totally separated from the main computer you are running CKB. This way it won’t be able to access data it doesn’t need, such as your private keys or passwords.

However, for a Script to be useful, there must be certain data it want to access, such as the Cell a Script guards, or a transaction a Script validates. CKB provides syscalls to ensure this, syscalls are defined in RISC-V standard, they provide a way to access certain resources provided by the environment. In a normal situation, the environment here means the operating system, but in the case of CKB VM, the environment refers to the actual CKB process. With syscalls, a CKB Script can access the whole transaction containing itself, including inputs, outputs, witnesses, and deps.

The good news, is that we have encapsulated syscalls in an easy to use library called CKB-STD in Rust, you are very welcome to poke around the source code of this library to see how syscalls are implemented. The bottomline is you can just grab this library and use the wrapped functions to make syscalls as you want.

For our carrot-forbidden Script to ensure none of the Cells can have carrot in Cell data, we will use CKB syscall to read Cell data in the Script.

Open the main code file `main.rs` and change it to the following:
```
#![no_std]
#![cfg_attr(not(test), no_main)]

#[cfg(test)]
extern crate alloc;

mod error;

use ckb_std::{ckb_constants::Source, debug, error::SysError, high_level::load_cell_data};

#[cfg(not(test))]
use ckb_std::default_alloc;
use error::Error;
#[cfg(not(test))]
ckb_std::entry!(program_entry);
#[cfg(not(test))]
default_alloc!();

pub fn program_entry() -> i8 {
    match carrot_forbidden() {
        Ok(_) => 0,
        Err(err) => err as i8,
    }
}

fn carrot_forbidden() -> Result<(), Error> {
    let mut index = 0;
    loop {
        match load_cell_data(index, Source::GroupOutput) {
            Ok(data) => {
                if data.starts_with("carrot".as_bytes()) {
                    return Err(Error::CarrotAttack);
                }else{
                    debug!("output #{:} has no carrot! Hooray!", index);
                }
            },
            Err(err) => {
                match err {
                    // we loop out all the output cell
                    SysError::IndexOutOfBound => break,
                    _ => return Err(Error::from(err)),
                }
            }
        }
        // Increment index to process next cell
        index += 1;
    }
    Ok(())
}
```

We also need to a Error module to define the error code when carrot Script fails. Create a new file named `error.rs` in the same directory:
```
use ckb_std::error::SysError;

#[cfg(test)]
extern crate alloc;

#[repr(i8)]
pub enum Error {
    IndexOutOfBound = 1,
    ItemMissing,
    LengthNotEnough,
    Encoding,
    // Add customized errors here...
    CarrotAttack,
}

impl From<SysError> for Error {
    fn from(err: SysError) -> Self {
        match err {
            SysError::IndexOutOfBound => Self::IndexOutOfBound,
            SysError::ItemMissing => Self::ItemMissing,
            SysError::LengthNotEnough(_) => Self::LengthNotEnough,
            SysError::Encoding => Self::Encoding,
            SysError::Unknown(err_code) => panic!("unexpected sys error {}", err_code),
        }
    }
}
```

Several points worth explaining here:

To load Cell data, we use `load_cell_data` syscall from CKB-STD library. the function takes two arguments, one is the index number, and one is source type flag that denoting the source of Cells to locate. Different source type and index number will point to different Cell. For example, `load_cell_data(0, Source::GroupOutput)` means load the first Cell from the output Cells group with the same running Script as current Script. You can check rfc for more details.

The workflow of our Script goes like this:

First, it loops through all output Cells in the transaction, load each Cell data and test if those bytes match the word carrot. If we found a match, the Script would return -1, denoting an error status, if no match is found, the Script exits with 0, meaning execution success.

To perform the loop, the Script would keep an index variable, in each loop iteration, it would tries to make the syscall to fetch the Cell denoted by current index value, if the syscall returns CKB_INDEX_OUT_OF_BOUND, it means the Script has iterated through all the Cells, hence it just exits the loop, otherwise, the loop would continue, the Cell data is tested, then index variable is incremented for the next iteration.

Besides the main logic in `main.rs` file, we also write a `error.rs` to define our custom error code for the carrot Script.

CKB defines some basic error code like:
* 1(CKB_INDEX_OUT_OF_BOUND) means you have finished fetching all indices in a kind
* 2(CKB_ITEM_MISSING) means an entity is not present, such as fetching a Type Script from a Cell that doesn’t have one.
* 3(CKB_LENGTH_NOT_ENOUGH) means some data length is wrong such as invalid Script args or signature length.
* 4(CKB_INVALID_DATA) means there is something wrong with the molecule serialization.

We define error code 5 to be the custom CarrotAttack error. So everytime the Script throws out error code 5, we know that means the Script have found a Cell data starts with the carrot word so it fails.

So that's all! This concludes your first useful CKB Script code!

In the next section, we will see how we can test and deploy it to CKB and run it.

### Testing Your Script
ckb-script-templates packs the unit test section that we can use to quickly test our Script without deploying the Script to blockchains.

All test case goes into the `tests.rs` file. The way tests works is to leverage a library called ckb_testtool to simulate the execution of the Script. The way tests works is to leverage a library called ckb_testtool to simulate the execution of the Script.
```
// Include your tests here
// See https://github.com/xxuejie/ckb-native-build-sample/blob/main/tests/src/tests.rs for examples

use super::*;
use ckb_testtool::{
    builtin::ALWAYS_SUCCESS,
    ckb_types::{bytes::Bytes, core::TransactionBuilder, packed::*, prelude::*},
    context::Context,
};

const MAX_CYCLES: u64 = 10_000_000;

#[test]
fn test_no_carrot() {
    // deploy contract
    let mut context = Context::default();
    let loader = Loader::default();
    let carrot_bin = loader.load_binary("carrot");
    let carrot_out_point = context.deploy_cell(carrot_bin);
    let carrot_cell_dep = CellDep::new_builder()
        .out_point(carrot_out_point.clone())
        .build();

    // prepare scripts
    let always_success_out_point = context.deploy_cell(ALWAYS_SUCCESS.clone());
    let lock_script = context
        .build_script(&always_success_out_point.clone(), Default::default())
        .expect("script");
    let lock_script_dep = CellDep::new_builder()
        .out_point(always_success_out_point)
        .build();

    // prepare cell deps
    let cell_deps: Vec<CellDep> = vec![lock_script_dep, carrot_cell_dep];

    // prepare cells
    let input_out_point = context.create_cell(
        CellOutput::new_builder()
            .capacity(1000u64.pack())
            .lock(lock_script.clone())
            .build(),
        Bytes::new(),
    );
    let input = CellInput::new_builder()
        .previous_output(input_out_point.clone())
        .build();

    let type_script = context
        .build_script(&carrot_out_point, Bytes::new())
        .expect("script");

    let outputs = vec![
        CellOutput::new_builder()
            .capacity(500u64.pack())
            .lock(lock_script.clone())
            .type_(Some(type_script.clone()).pack())
            .build(),
        CellOutput::new_builder()
            .capacity(500u64.pack())
            .lock(lock_script)
            .build(),
    ];

    // prepare output cell data
    let outputs_data = vec![Bytes::from("apple"), Bytes::from("tomato")];

    // build transaction
    let tx = TransactionBuilder::default()
        .cell_deps(cell_deps)
        .input(input)
        .outputs(outputs)
        .outputs_data(outputs_data.pack())
        .build();

    let tx = tx.as_advanced_builder().build();

    // run
    let cycles = context
        .verify_tx(&tx, MAX_CYCLES)
        .expect("pass verification");
    println!("consume cycles: {}", cycles);
}
```

As you can see, we wrote the first test case named test_no_carrot to simulate a transaction that use our carrot-forbidden Script in one of the output Cell, and we build all the output Cell without carrot in their Cell data. To make the test pass, the transaction should be valid and verified by CKB successfully.

We can also write another test that simulates if the transaction does contains output Cell that its data starts with the word carrot:
```
#[test]
fn test_carrot_attack() {
    // deploy contract
    let mut context = Context::default();
    let loader = Loader::default();
    let carrot_bin = loader.load_binary("carrot");
    let carrot_out_point = context.deploy_cell(carrot_bin);
    let carrot_cell_dep = CellDep::new_builder()
        .out_point(carrot_out_point.clone())
        .build();

    // prepare scripts
    let always_success_out_point = context.deploy_cell(ALWAYS_SUCCESS.clone());
    let lock_script = context
        .build_script(&always_success_out_point.clone(), Default::default())
        .expect("script");
    let lock_script_dep = CellDep::new_builder()
        .out_point(always_success_out_point)
        .build();

    // prepare cell deps
    let cell_deps: Vec<CellDep> = vec![lock_script_dep, carrot_cell_dep];

    // prepare cells
    let input_out_point = context.create_cell(
        CellOutput::new_builder()
            .capacity(1000u64.pack())
            .lock(lock_script.clone())
            .build(),
        Bytes::new(),
    );
    let input = CellInput::new_builder()
        .previous_output(input_out_point.clone())
        .build();

    let type_script = context
        .build_script(&carrot_out_point, Bytes::new())
        .expect("script");

    let outputs = vec![
        CellOutput::new_builder()
            .capacity(500u64.pack())
            .lock(lock_script.clone())
            .type_(Some(type_script.clone()).pack())
            .build(),
        CellOutput::new_builder()
            .capacity(500u64.pack())
            .lock(lock_script)
            .build(),
    ];

    // prepare output cell data
    let outputs_data = vec![Bytes::from("carrot"), Bytes::from("tomato")];

    // build transaction
    let tx = TransactionBuilder::default()
        .cell_deps(cell_deps)
        .input(input)
        .outputs(outputs)
        .outputs_data(outputs_data.pack())
        .build();

    let tx = tx.as_advanced_builder().build();

    // run
    let err = context.verify_tx(&tx, MAX_CYCLES).unwrap_err();
    assert_script_error(err, 5);
}

fn assert_script_error(err: Error, err_code: i8) {
    let error_string = err.to_string();
    assert!(
        error_string.contains(format!("error code {} ", err_code).as_str()),
        "error_string: {}, expected_error_code: {}",
        error_string,
        err_code
    );
}
```

In this test, we check if the error code throws by our Script is matched with the error code 5 in `assert_script_error(err, 5);`.
The directory of the smart contract always contain three source files of Rust: 
* `entry.rs` - This file contains the logic of the smart contract.
* `error.rs` - This file contains error codes used in the project.
* `main.rs` - This file contains boilerplate code for the contract.

Every Rust program begins with `main.rs`. Feel free to open it and take a look at what it contains, but we're not going to go through it here. This file is all boilerplate code that is needed to build a script. We won't need to touch anything in this file. 

Next, let's look at the contents of `entry.rs`:
```
// Import from `core` instead of from `std` since we are in no-std mode
use core::result::Result;

// Import heap related library from `alloc`
// https://doc.rust-lang.org/alloc/index.html
use alloc::{vec, vec::Vec};

// Import CKB syscalls and structures
// https://docs.rs/ckb-std/
use ckb_std::{
    debug,
    high_level::{load_script, load_tx_hash},
    ckb_types::{bytes::Bytes, prelude::*},
};

use crate::error::Error;

pub fn main() -> Result<(), Error> {
    // remove below examples and write your code here

    let script = load_script()?;
    let args: Bytes = script.args().unpack();
    debug!("script args is {:?}", args);

    // return an error if args is invalid
    if args.is_empty() {
        return Err(Error::MyError);
    }

    let tx_hash = load_tx_hash()?;
    debug!("tx hash is {:?}", tx_hash);

    let _buf: Vec<_> = vec![0u8; 32];

    Ok(())
}
```

This file contains the main logic of the script. This example script contains boilerplate code and example code. Functionally, all it does is check if `args` were supplied to the script, and it gives an error if there were none.

Next, let's look at `error.rs`:
```
use ckb_std::error::SysError;

/// Error
#[repr(i8)]
pub enum Error {
    IndexOutOfBound = 1,
    ItemMissing,
    LengthNotEnough,
    Encoding,
    // Add customized errors here...
    MyError,
}

impl From<SysError> for Error {
    fn from(err: SysError) -> Self {
        use SysError::*;
        match err {
            IndexOutOfBound => Self::IndexOutOfBound,
            ItemMissing => Self::ItemMissing,
            LengthNotEnough(_) => Self::LengthNotEnough,
            Encoding => Self::Encoding,
            Unknown(err_code) => panic!("unexpected sys error {}", err_code),
        }
    }
}
```

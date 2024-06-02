
CKB defines some basic error codes:

* 1: CKB_INDEX_OUT_OF_BOUND, means you have finished fetching all indices in a kind
* 2: CKB_ITEM_MISSING, means an entity is not present, such as fetching a type script from a cell that doesn’t have one.
* 3: CKB_LENGTH_NOT_ENOUGH, means some data length is wrong such as invalid script args or signature length.
* 4: CKB_INVALID_DATA, means there is something wrong with the molecule serialization.

Molecule Error Codes from CKB standard C library
```
#define MOL2_ERR_TOTAL_SIZE 0x01
#define MOL2_ERR_HEADER 0x02
#define MOL2_ERR_OFFSET 0x03
#define MOL2_ERR_UNKNOWN_ITEM 0x04
#define MOL2_ERR_INDEX_OUT_OF_BOUNDS 0x05
#define MOL2_ERR_FIELD_COUNT 0x06
#define MOL2_ERR_DATA 0x07
#define MOL2_ERR_OVERFLOW 0x08
```

# ipfs-embed
A small embeddable ipfs implementation compatible with libipld and with a concurrent garbage
collector.

## Getting started
```rust
use ipfs_embed::{Config, Store};
use ipld_block_builder::{BlockBuilder, DagCbor, Key};

#[derive(Clone, DagCbor, Debug, Eq, PartialEq)]
struct Identity {
    id: u64,
    name: String,
    age: u8,
}

#[async_std::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = Config::from_path("/tmp/db")?;
    let store = Store::new(config)?;
    let key = Key::from(b"private encryption key".to_vec());
    let builder = BlockBuilder::new_private(store, key);

    let identity = Identity {
        id: 0,
        name: "David Craven".into(),
        age: 26,
    };
    let cid = builder.insert(&identity).await?;
    let identity2 = builder.get(&cid).await?;
    assert_eq!(identity, identity2);
    println!("encrypted identity cid is {}", cid);

    Ok(())
}
```

## Concurrent garbage collection
```rust
    // #roots = {} #live = {}
    let a = builder.insert(&ipld!({ "a": [] })).await?;
    // #roots = {a} #live = {a}
    let b = builder.insert(&ipld!({ "b": [&a] })).await?;
    // #roots = {a, b} #live = {a, b}

    // block `b` contains a reference to block `a`, so the
    // garbage collector won't remove it until block `b` gets
    // unpinned
    builder.unpin(&a).await?;
    // #roots = {b} #live = {a, b}

    let c  = builder.insert(&ipld!({ "c": [&a] })).await?;
    // #roots = {b, c} #live = {a, b, c}

    // the backing storage is content addressed, but inserting
    // a block multiple times increases the ref count.
    let c2 = builder.insert(&ipld!({ "c": [&a] })).await?;
    // #roots = {b, c, c} #live = {a, b, c}

    builder.unpin(&b).await?;
    // #roots = {b, c} #live = {a, c}
    builder.unpin(&c2).await?;
    // #roots = {c} #live = {a, c}
    builder.unpin(&c).await?;
    // #roots = {} #live = {}
```

## License
MIT OR Apache-2.0

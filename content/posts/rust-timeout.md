---
layout:     post
title:      "RUST 的定时与计时器"
date:       2019-02-02
author:     "Gitai"
tags:
    - Rust
---

```rust
#![feature(duration_as_u128)]
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::{Duration, SystemTime};
use std::sync::mpsc;
use std::thread::JoinHandle;

fn main() {
    let (tx, rx) = mpsc::channel();

    let now = SystemTime::now();
    let handle = thread::spawn(move || {
        {
            // 定时器
            let tx = tx.clone();
            thread::spawn(move || {// do something
                thread::sleep(Duration::from_millis(500));
                tx.send(Err(())).unwrap();
            });
        }
        // 干一些不为人知的事情
        thread::sleep(Duration::from_millis(300));
        tx.send(Ok("result")).unwrap();
    });

    match rx.recv().unwrap() {
        // 计时器
        Ok(data) => println!("{:?}, {}", data, now.elapsed().unwrap().as_millis()),
        Err(err) => {
            handle.join();
        }
    }
}
```

<!-- more -->
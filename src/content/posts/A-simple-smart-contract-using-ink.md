---
title: ink!实现一个简单的智能合约
published: 2023-03-06
description: "用ink!实现一个简单的智能合约，部署到波卡生态网络上"
img: "https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Java/3.jpg"
category: 编程实践
tags: ["Rust"]
draft: false
---

ink!是rust下的一种EDSL，用来在波卡生态上实现智能合约。这一篇博客来用ink!实现一个简单的token，部署到Aleph Zero的测试网上。

## 前期准备

首先需要安装最新版的rust，然后安装工具：

```shell
rustup toolchain install nightly
rustup component add rust-src --toolchain nightly
rustup target add wasm32-unknown-unknown --toolchain nightly
```

还要安装binaryen，这个是wasm相关的工具，

```shell
# For Ubuntu or Debian users:
sudo apt install binaryen
# For MacOS users:
brew install binaryen
# For Arch or Manjaro users:
pacman -S binaryen
```

安装好必要的工具之后，接下来就可以来实现我们的简单的智能合约了。

## 创建项目

首先使用命令来创建合约。

```shell
cargo contract new toytoken
```

进入到目录里面，我们会发现ink!已经为我们生成好了非常简单的代码，就在`lib.rs`里面，只能保存一个bool值。这个自动生成的合约代码也非常值得读一读。

不过我们的目标是实现我们自己的合约，因此需要把其中`mod toytoken`里面的内容删除，换上我们自己的。

## Toytoken

首先要实现的是Toytoken的结构体，在实现之前，注意一下ink!虽然是rust的EDSL，但是它不能引入rust自身的数据结构，只能引入ink!的数据结构。为了存储账户和余额，引入ink!里的Mapping。

```rust
    use ink::storage::Mapping;

    #[ink(storage)]
    #[derive(Default)]
    pub struct Toytoken {
        total_supply: Balance,
        balances: Mapping<AccountId, Balance>,
    }
```

结构体注意使用`#[ink(storage)]`来指明这个结构体需要存储在链上。同时使用`#[derive(Default)]`生成默认构造函数。

然后来实现一些功能。

首先是合约启动时需要调用的函数new，它会直接分配给调用者全部的token数量。

```rust
impl Toytoken {
    #[ink(constructor)]
    pub fn new(total_supply: Balance) -> Self {
        let mut balances = Mapping::default();
        let caller = Self::env().caller();
        balances.insert(caller, &total_supply);
        Self {
            total_supply,
            balances,
        }
    }
    //...
}
```

接着是转账函数，转账函数的sender默认就是函数的调用者，然后检查余额是否足够发起转账，如果不够，需要抛出一个异常，所以函数需要返回一个`Result`。如果转账成功，那么receiver的余额要增加，最后要返回一个Ok。

```rust
pub fn transfer(&mut self, to: AccountId, value: Balance) -> Result<(), Error> {
    let from = self.env().caller();
    let from_balance = self.balance_of(from);
    if from_balance < value {
         return Err(Error::InsufficientBalance);
    }
    let current_from_balance = from_balance.checked_sub(value).unwrap();
    let to_balance = self.balance_of(to);
    let to_balance = to_balance.checked_add(value).unwrap();
    self.balances.insert(from, &current_from_balance);
    self.balances.insert(to, &to_balance);
    Ok(())
}
```

注意这里面使用了一个`Error::InsufficientBalance`。这个需要额外实现，可以直接使用下面的代码。

```rust
    #[ink::scale_derive(Encode, Decode, TypeInfo)]
    #[derive(Debug, PartialEq, Eq)]
    pub enum Error {
        InsufficientBalance,
    }
```

最后就是测试了，不过在测试之前，实现两个查询用的函数：

```rust
#[ink(message)]
pub fn total_supply(&self) -> Balance {
    self.total_supply
}

#[ink(message)]
pub fn balance_of(&self, owner: AccountId) -> Balance {
    self.balances.get(&owner).unwrap_or_default()
}
```

## 测试

最后进行测试，就是正常的rust测试：

```rust
#[cfg(test)]
mod tests {
    /// Imports all the definitions from the outer scope so we can use them here.
    use super::*;

    #[ink::test]
    fn total_supply_works() {
        let mytoken = Toytoken::new(100);
        assert_eq!(mytoken.total_supply(), 100);
    }

    #[ink::test]
    fn balance_of_works() {
        let mytoken = Toytoken::new(100);
        let accounts = ink::env::test::default_accounts::<ink::env::DefaultEnvironment>();
        assert_eq!(mytoken.balance_of(accounts.alice), 100);
        assert_eq!(mytoken.balance_of(accounts.bob), 0);
    }

    #[ink::test]
    fn transfer_works() {
        let mut mytoken = Toytoken::new(100);
        let accounts = ink::env::test::default_accounts::<ink::env::DefaultEnvironment>();

        assert_eq!(mytoken.balance_of(accounts.bob), 0);
        assert_eq!(mytoken.transfer(accounts.bob, 10), Ok(()));
        assert_eq!(mytoken.balance_of(accounts.bob), 10);
    }
}
```

使用`cargo test`进行测试即可。

## 合约部署

ink!构建的合约可以在大多数波卡生态链上运行，这里使用的是Aleph Zero的测试网络。

使用`cargo +nightly contract build --release`来编译合约。

编译完成之后，可以在`token/target/ink`目录下，找到三个文件，分别是`toytoken.contract`，`toytoken.json`以及一个`toytoken.wasm`。

接着就可以部署合约，到这个网站：[https://ui.use.ink/](https://ui.use.ink/)。

需要一个波卡生态项目的钱包，这里推荐使用Aleph Zero网络，因为没啥人用，所以速度比较快。

生成好一个钱包之后，在测试网的Faucet上领取一些免费的测试币（Aleph Zero的Faucet地址是：[https://faucet.test.azero.dev/](https://faucet.test.azero.dev/)）。接着就可以将合约上传上去，剩余的步骤，在网站上都有详细的解释，难度不高，这里就不再赘述。

## 完整代码

```rust
#![cfg_attr(not(feature = "std"), no_std, no_main)]

#[ink::contract]
mod toytoken {

    use ink::storage::Mapping;

    /// Defines the storage of your contract.
    /// Add new fields to the below struct in order
    /// to add new static storage fields to your contract.
    #[ink(storage)]
    #[derive(Default)]
    pub struct Toytoken {
        total_supply: Balance,
        balances: Mapping<AccountId, Balance>,
    }

    // #[derive(Debug, PartialEq, Eq, scale::Encode, scale::Decode)]
    // #[cfg_attr(feature = "std", derive(scale_info::TypeInfo))]
    #[ink::scale_derive(Encode, Decode, TypeInfo)]
    #[derive(Debug, PartialEq, Eq)]
    pub enum Error {
        InsufficientBalance,
    }

    impl Toytoken {
        #[ink(constructor)]
        pub fn new(total_supply: Balance) -> Self {
            let mut balances = Mapping::default();
            let caller = Self::env().caller();
            balances.insert(caller, &total_supply);
            Self {
                total_supply,
                balances,
            }
        }

        #[ink(message)]
        pub fn transfer(&mut self, to: AccountId, value: Balance) -> Result<(), Error> {
            let from = self.env().caller();
            let from_balance = self.balance_of(from);
            if from_balance < value {
                return Err(Error::InsufficientBalance);
            }
            let current_from_balance = from_balance.checked_sub(value).unwrap();
            let to_balance = self.balance_of(to);
            let to_balance = to_balance.checked_add(value).unwrap();
            self.balances.insert(from, &current_from_balance);
            self.balances.insert(to, &to_balance);
            Ok(())
        }

        #[ink(message)]
        pub fn total_supply(&self) -> Balance {
            self.total_supply
        }

        #[ink(message)]
        pub fn balance_of(&self, owner: AccountId) -> Balance {
            self.balances.get(&owner).unwrap_or_default()
        }
    }

    #[cfg(test)]
    mod tests {
        /// Imports all the definitions from the outer scope so we can use them here.
        use super::*;

        #[ink::test]
        fn total_supply_works() {
            let mytoken = Toytoken::new(100);
            assert_eq!(mytoken.total_supply(), 100);
        }

        #[ink::test]
        fn balance_of_works() {
            let mytoken = Toytoken::new(100);
            let accounts = ink::env::test::default_accounts::<ink::env::DefaultEnvironment>();
            assert_eq!(mytoken.balance_of(accounts.alice), 100);
            assert_eq!(mytoken.balance_of(accounts.bob), 0);
        }

        #[ink::test]
        fn transfer_works() {
            let mut mytoken = Toytoken::new(100);
            let accounts = ink::env::test::default_accounts::<ink::env::DefaultEnvironment>();

            assert_eq!(mytoken.balance_of(accounts.bob), 0);
            assert_eq!(mytoken.transfer(accounts.bob, 10), Ok(()));
            assert_eq!(mytoken.balance_of(accounts.bob), 10);
        }
    }
}
```

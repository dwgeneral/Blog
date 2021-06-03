> 最近在负责海龙项目的整个容器化部署, 发现在编译Rust镜像的时候, 学到了一些新知识

**动态编译**

- 简单说就是动态编译的可执行文件, 在运行时需要相应链接库的支持, 如果运行环境中没有安装相应的支持, 则无法执行
- 动态编译的可执行文件需要附带一个动态链接库, 在执行时, 需要调用其对应动态链接库中的命令.
- 优点是: 缩小了执行文件本身的体积; 加快了编译速度
- 缺点是: 哪怕是很简单的程序, 也需要附带一个相对庞大的链接库; 如果服务器没有安装对应的运行库, 则用动态编译的可执行文件就不能运行
- Rust 默认是以动态链接的方式链到外部库的

**静态编译**

- 编译器在编译可执行文件时, 把需要用到的对应动态链接库中的部分提取出来, 链接到可执行文件中去, 是可执行文件在运行时不需要依赖动态链接库
- 缺点是可执行文件体积更大
- 目前业内最佳实践是将 musl 库以及第三方的 C 库静态链接到 Rust 程序可执行文件中(因为 musl 是轻量级的 libc 库, 与 glibc 库相比, 它的代码比较简洁, 体积也更小)

**bin**

- 二进制可执行crate, 编译出的文件为二进制可执行文件

**lib**

- 库crate, 它其实只是一个指代, 指代下面各种库crate中的一种, 默认指代 rlib, 会生成 .rlib 文件

**rlib**

- Rust Library 特定静态中间库格式, 如果只是纯 Rust 代码项目之间的依赖和调用, 用 rlib 就能完全满足需求

**dylib**

- 动态库, 会在编译的时候, 生成动态链接库( Linux 上为 .so, MacOS 上为 .dylib, Windows 上为 .dll)
- 该动态库只能被 Rust 写的程序调用

**cdylib**

- C 规范动态库, 与 dylib 类似, 但这种动态库可以被其他语言调用(因为几乎所有语言都有遵循C规范的 FFI 实现) [FFI: Foreign Function Interface 跨语言调用]

**staticlib**

- 静态库, 编译会生成 .a 文件(Linux, MacOS), 或 .lib 文件(Windows)
- 编译器会把所有实现的 rust 库代码以及依赖的库代码全部编译到一个静态库文件中, 也就是对外界不产生任何依赖了
- 适合将 Rust 实现的功能封装好给第三方应用使用

参考资料:

[https://users.rust-lang.org/t/what-is-the-difference-between-dylib-and-cdylib/28847/3](https://users.rust-lang.org/t/what-is-the-difference-between-dylib-and-cdylib/28847/3)

[https://rustcc.cn/article?id=98b96e69-7a5f-4bba-a38e-35bdd7a0a7dd](https://rustcc.cn/article?id=98b96e69-7a5f-4bba-a38e-35bdd7a0a7dd)

[https://www.bilibili.com/read/cv2076710/](https://www.bilibili.com/read/cv2076710/)

[https://blog.biofan.org/2019/08/rust-static-build-with-musl/](https://blog.biofan.org/2019/08/rust-static-build-with-musl/)

附录:

- 使用 docker 静态编译 rust 程序

  ```go
  # ------------------------------------------------------------------------------
  # Build Stage
  # ------------------------------------------------------------------------------
  FROM ekidd/rust-musl-builder:stable as builder

  COPY --chown=rust:rust id_rsa .

  RUN eval $(ssh-agent) && \
      mkdir ~/.ssh && mv id_rsa ~/.ssh/ && \
      touch ~/.ssh/config && touch ~/.ssh/known_hosts && \
      echo "StrictHostKeyChecking no" >> ~/.ssh/config && \
      chmod 700 ~/.ssh && \
      chmod 600 ~/.ssh/id_rsa && \
      ssh-add ~/.ssh/id_rsa && \
      ssh-keyscan -H git.private-repo >> ~/.ssh/known_hosts

  RUN mkdir build && mkdir src && \
      echo "fn main() {println!(\"if you see this, the build broke\")}" > src/main.rs
  COPY .cargo/config .cargo/config
  COPY Cargo.toml Cargo.lock ./

  RUN cargo build --target=x86_64-unknown-linux-musl --release
  RUN rm -f target/x86_64-unknown-linux-musl/release/deps/app* && \
      rm -f target/release/app*

  COPY src src
  COPY config config
  COPY migrations migrations
  RUN cargo build --target=x86_64-unknown-linux-musl --release

  # ------------------------------------------------------------------------------
  # Run Stage
  # ------------------------------------------------------------------------------
  FROM alpine:latest
  COPY --from=builder /home/rust/src/target/x86_64-unknown-linux-musl/release/app ./
  COPY --from=builder /home/rust/src/config/config-local.toml ./config.toml
  CMD ["./app", "--config=./config.toml"]
  ```

- 使用 docker 动态编译 rust 程序

  ```go
  # ------------------------------------------------------------------------------
  # Build Stage
  # ------------------------------------------------------------------------------
  FROM my-private-repo/rust-musl-builder:2.2.5 as builder

  RUN mkdir build src && \
      echo "fn main() {println!(\"if you see this, the build broke\")}" > src/main.rs
  COPY .cargo/config .cargo/config
  COPY Cargo.toml Cargo.lock ./

  RUN cargo build --release
  RUN rm -f target/release/deps/hyper_asset*

  COPY src src
  COPY config config
  COPY migrations migrations
  RUN cargo build --release

  # ------------------------------------------------------------------------------
  # Run Stage
  # ------------------------------------------------------------------------------
  FROM debian:buster-slim
  RUN apt-get update && apt-get -y install libpq5 libpq-dev
  COPY --from=builder /volume/target/release/app ./
  COPY --from=builder /volume/config/config-local.toml ./config.toml
  CMD ["./app", "--config=./config.toml"]
  ```

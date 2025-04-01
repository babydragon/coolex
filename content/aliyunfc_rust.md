+++
title = "阿里云函数计算Rust"
date = "2025-03-31"
[taxonomies]
tags = ["function", "阿里云"]
categories = ["Rust"]
+++

阿里云函数计算的SDK是没有Rust版本的，但是可以通过自定义运行时来直接运行Rust代码。
第一次尝试使用阿里云函数计算实现从OSS下载前端工程，通过函数进行构建后，将构建产物重新上传到OSS，并且通过CDN进行分发。
<!-- more -->

## 准备工作
阿里云函数计算实际还是通过http请求来调用的，因此，我们需要一个http server来接收请求。
这里使用`actix-web`来实现一个简单的http server。

```bash
cargo add actix-web
```

## 实现
### actix-web实现
根据阿里云函数的文档，http服务需要响应以下路径：

* `/initialize`：POST请求，函数实例初始化时候调用，可以用来准备函数执行上下文。因为自定义运行时，是可以指定一个函数实例相应多个请求的，因此可以为整个函数运行时进行准备。
* `/invoke`：POST请求，函数调用。
* `/pre-stop`：GET请求，函数实例销毁前调用。这里可以用来保存缓存数据等。

```rust

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // init logger
    env_logger::init_from_env(Env::default().default_filter_or("info"));
    let args: Vec<String> = env::args().collect();
    let port = if args.len() > 1 {
        args[1].parse().unwrap_or(9000)
    } else {
        9000
    };

    info!("Starting server on port: {}", port);

    HttpServer::new(|| {
        App::new()
            .route("/invoke", web::post().to(invoke))
            .route("/initialize", web::post().to(initialize))
            .route("/pre-stop", web::get().to(pre_stop))
    })
    .bind(("0.0.0.0", port))?
    .run()
    .await
}
```
入口代码非常简单，只是创建了一个http server，监听指定端口，然后分别处理三个请求。阿里云函数默认端口是9000，也可以在创建的时候指定。

函数计算在调用函数的时候（包括initialize和pre-stop）都会通过http请求头和环境变量传递上下文参数，这里统一初始化成一个结构体。

```rust
fn init_fc_context(req: &HttpRequest) -> FcContext {
    let headers = req.headers();
    // parse headers to create FcContext
    FcContext {
        request_id: headers
            .get("x-fc-request-id")
            .and_then(|h| h.to_str().ok())
            .unwrap_or("")
            .to_string(),
        access_key_id: headers
            .get("x-fc-access-key-id")
            .and_then(|h| h.to_str().ok())
            .unwrap_or(env::var("ALIBABA_CLOUD_ACCESS_KEY_ID").unwrap_or("".to_string()).as_str())
            .to_string(),
        access_key_secret: headers
            .get("x-fc-access-key-secret")
            .and_then(|h| h.to_str().ok())
            .unwrap_or(env::var("ALIBABA_CLOUD_ACCESS_KEY_SECRET").unwrap_or("".to_string()).as_str())
            .to_string(),
        security_token: headers
            .get("x-fc-security-token")
            .and_then(|h| h.to_str().ok())
            .unwrap_or(env::var("ALIBABA_CLOUD_SECURITY_TOKEN").unwrap_or("".to_string()).as_str())
            .to_string(),
        function_name: headers
            .get("x-fc-function-name")
            .and_then(|h| h.to_str().ok())
            .unwrap_or("")
            .to_string(),
        region: headers
            .get("x-fc-region")
            .and_then(|h| h.to_str().ok())
            .unwrap_or(env::var("FC_REGION").unwrap_or("cn-shanghai".to_string()).as_str())
            .to_string(),
        log_project: headers
            .get("x-fc-log-project")
            .and_then(|h| h.to_str().ok())
            .unwrap_or("")
            .to_string(),
        log_store: headers
            .get("x-fc-log-store")
            .and_then(|h| h.to_str().ok())
            .unwrap_or("")
            .to_string(),
    }
}
```

**注意**：函数的文档中说新的Debian 11运行时会取消通过http请求头来传递STS令牌信息，实际测试老版本也不会传递，必须通过环境变量传递。

初始化完成之后，就可以在各自回调函数中使用这个上下文信息了。我选择了`ali-oss-rs`来实现OSS的上传下载功能。
请使用新版本，新版本增加了完全去除openssl的依赖，使用rustls来实现https，具体原因后面会说。
```toml
ali-oss-rs = { version = "0.2.1", default-features = false, features = ["async","rust-tls"] }
```

```rust
let oss_client = ali_oss_rs::ClientBuilder::new(
        fc_context.access_key_id.as_str(),
        fc_context.access_key_secret.as_str(),
        format!("oss-{}-internal.aliyuncs.com", fc_context.region),
    ).region(fc_context.region.clone())
        .sts_token(fc_context.security_token.clone()).build().unwrap();
```

这个库可以很方便异步的方式上传和下载文件，同时默认已经使用OSS v4版本进行签名，避免后续兼容问题。
需要注意的是，v4版本签名，必须指定region，否则会出现签名错误。

另外需要特别注意的是，通过阿里云SDK中的`invokeFunction`接口来调用函数的时候，传入的参数需要从请求体中获取，
但是不论传递的时候给的是字符串还是json，请求的`Content-Type`都是`application/octet-stream`，因此不能使用`actix-web`的`Json`提取参数。

```rust
async fn invoke(body: Bytes, req: HttpRequest) -> Result<impl Responder, BuildError> {
    let fc_context = init_fc_context(&req);
    let build_context: BuildContext = serde_json::from_slice(&body).unwrap();

...
```
这里使用`Bytes`来接收请求体，然后通过`serde_json`来解析json数据。

### 阿里云函数配置
由于我们需要使用到Node.js的运行时，因此没有选择最新基于Debian 11的运行时，而是选择了基于Debian 10的运行时。
这个函数需要访问OSS，所以除了函数本身之外，还需要创建一个关联的RAM角色，然后给这个角色授权访问OSS的权限。
这里注意的是，创建函数的时候选择web函数，然后选择自定义运行时，同时创建完成之后，需要将Node.js配置到环境变量中（PATH）。

然后将编译好的二进制文件直接上传到函数平台即可。

## 运行函数遇到的问题

### 子账号初始化问题
文档中没有明确说明老版本的运行时，初始化的时候子账号的信息，包括AK、SK、SecurityToken已经不能从http请求头中获取。
实际验证后才发现，现在这些敏感数据，只能通过环境变量传递。

### openssl依赖问题
第一版二进制文件上传到函数平台之后，运行的时候报错，提示无法找到openssl的动态库。通过`cargo tree`查看依赖，
发现`reqwest`依赖了`openssl`。这个依赖是从`ali-oss-rs`中传递过来的，当时这个库可以指定使用`rustls`，但是并没有去除对`openssl`的依赖。
后续给`ali-oss-rs`提交了PR，可以通过feature开关完全去除对native openssl的依赖。

### glibc版本问题
本地构建出来的二进制文件依赖的glibc版本比较高，导致在函数平台上运行的时候报错。而函数的运行时采用了Debian，对glibc版本比较保守。
由于降低本地glibc版本比较麻烦，本地编译一个musl版本也比较麻烦（Gentoo系统glibc和musl是完全独立的两个版本，很难共存），
所以选择通过`cross`工具来编译一个musl版本的二进制文件。

首先安装`cross`工具：
```bash
cargo install cross
```

然后构建：
```bash
cross build --release --target x86_64-unknown-linux-musl
```

这样就可以得到一个不依赖glibc的二进制文件，完全静态链接，避免执行时候的依赖问题。

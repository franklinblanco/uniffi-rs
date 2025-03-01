# Using `uniffi` with iOS

Getting a Rust library working with `uniffi` and iOS has a few working parts.

For this example, we'll build a small app linked to one of the `uniffi` examples, the `todolist`.

This assumes you have your files setup this way: /directory/rust_dir & directory/xcode_dir

This will not be a complete tutorial on how to use `uniffi`, just the bits to get it running with Xcode and iOS.

## Install Rust compiler targets with `rustup`

```sh
% rustup target add x86_64-apple-ios aarch64-apple-ios
```

## Associate the iOS project with the Rust project

1. In Xcode, click on the project in the Project Navigator.
2. In the main window select the app's main target, and then select "Build Phases".
3. Click on Compile Sources and expand the dropdown menu
4. Add the .udl file by clicking "Add Other", then find it in your rust src

## Tell Xcode what to do with `udl` files.

1. In Xcode, click on the project in the Project Navigator.
2. In the main window, select the app's main target, and then select "Build Rules".
3. Add a custom rule, to process sources files with names matching `*.udl`.
4. Use `uniffi-bindgen` on each file to generate the headers and swift scaffolding.

Copy this just as it is

```sh
$HOME/.cargo/bin/uniffi-bindgen generate "$INPUT_FILE_PATH" --language swift --out-dir "$DERIVED_FILE_DIR"
```

These will output two files Xcode is interested in.

Paste these two individually in the output files.

```sh
$(DERIVED_FILE_DIR)/Generated/$(INPUT_FILE_BASE).swift
$(DERIVED_FILE_DIR)/Generated/$(INPUT_FILE_BASE)FFI.h
```

This is how it should look:
<img width="1135" alt="Screenshot 2022-11-30 at 10 00 35 AM" src="https://user-images.githubusercontent.com/72827992/204815820-f03c3326-230b-4704-98b3-b914a1627fcc.png">

### Just in case step

In some cases you might have to go into your rust dir and run 

```sh
uniffi-bindgen generate src/network.udl --language swift
```

then grab the FFI.h & .swift files, then copy it to the xcode project root/Generated manually at least for the first time.

## Header File

The header file is a descriptor of the C API to a library that hasn't been built yet. The Swift file is a Swift facade that calls that C API.

## Tell `swiftc` about the generated header files

1. In Xcode, click on the project in the Project Navigator.
2. In the main window, select the app's main target, and then select "Build Settings".
3. Search for `Objective C Bridging Header`, and add one if needed. For our example app, we've called it `IOSApp-Bridging-Header.h`. Only write the name in the field.
4. It should be generated for you, but in case it isn't create it manually in your xcode project root, it should look like:

```h
#ifndef IOSApp_Bridging_Header_h
#define IOSApp_Bridging_Header_h

#import "proj_nameFFI.h"

#endif /* IOSApp_Bridging_Header_h */
```

The `#import` directive points to the header file generated by `uniffi-bindgen generate` above— `$(DERIVED_FILE_DIR)//Generated/$(INPUT_FILE_BASE)FFI.h`

```h
#import "proj_nameFFI.h"
```

This `IOSApp-Bridging-Header.h` tells Xcode to look for the header file; because it's in `/Generated`, Xcode should know where to look.

<img width="401" alt="Screenshot 2022-11-30 at 10 02 55 AM" src="https://user-images.githubusercontent.com/72827992/204816116-0bffbf6e-38ef-4b74-9908-97c553158940.png">

## Configure `cargo` to build the crate as a static library

1. In the `Cargo.toml` file of the Rust project, add `staticlib` to the `crate-type` list.

```toml
[lib]
crate-type = ["staticlib", "cdylib"]
name = "uniffi_todolist"
```

The `package` `name` and the `lib` `name` will be used below. In this case, the package name is `uniffi-example-todolist` and the lib name is `uniffi_todolist`.

## Tell Xcode how to build the Rust project.

#### TODO use a better explanation of what is going on here

1. In Xcode, click on the project in the Project Navigator.
2. In the main window, select the app's main target, and then select "Build Phases".
3. Add a new `Run Script` build phase and move it to the top.
4. Add the script that will build a universal binary for the Rust project.

For this project, we've used a script adapted from the #mozilla/application-services project to build a universal binary with `lipo`.

```sh
bash $SRCROOT/xc-universal-binary.sh libuniffi_todolist.a uniffi-example-todolist $SRCROOT/../../../ $CONFIGURATION
```

In this case we constructed the command:

```sh
xc-universal-binary.sh <STATIC_LIB_NAME> <FFI_TARGET> <WORKSPACE_PATH> <BUILD_CONFIGURATION>"
```

by making:

 * `STATIC_LIB_NAME` from the `lib` `name` above: `uniffi_todolist` --> `libuniffi_todolist.a`
 * `FFI_TARGET` from the `package` `name` above: `uniffi-example-todolist`.

The workspace path is where the `Cargo.toml` will resolve the Rust project, and also determine the target directory that `cargo build` and `lipo` will put its artifacts.

This script performs a few steps:

1. Runs `cargo build` to compile the Rust project for the `x86_64-apple-ios` and `aarch64-apple-ios` targets.
    * This includes using `uniffi-bindgen` to generate the Rust scaffolding so we can go from C to Rust.
2. Runs `lipo` to combine these libs in to a universal binary.
3. Puts the universal binary in to the `$WORKSPACE_PATH/target/universal` directory.

<img width="1136" alt="Screenshot 2022-11-30 at 10 05 03 AM" src="https://user-images.githubusercontent.com/72827992/204816678-536b4259-6964-4756-959c-df927bedef1c.png">

## Tell Xcode where the universal library is

Finally, we need to tell Xcode to look for the universal binary `libuniffi_todolist.a` is, so it can tie it together with the header file `todolist-Bridging-Header.h`.

1. In Xcode, click on the project in the Project Navigator.
2. In the main window, select the app's main target, and then select "Build Settings".
3. Search for `Library Search Paths`.
4. Add paths to where lipo constructed the universal binaries for each of `Debug` and `Release`.

```sh
$(SRCROOT)/../../../target/universal/debug
$(SRCROOT)/../../../target/universal/release
```

## Back to code.

By now, we should've 

1. Used `uniffi-bindgen` to generate a Swift file to call a C API header and put them in a place Xcode can find them.
2. Configured Xcode to find the header.
3. Configured cargo to build a static library version of the crate.
4. Used `uniffi-bindgen` to generate a C API for the Rust crate.
5. Used cargo to cross-compile the crate for different targets.
6. Used lipo to combine those crates for different target into a universal binary.
7. Configured Xcode to find the universal binary.

By now, we should be able to hit Run in Xcode and build something.

Once this builds and runs okay, we should be able to start using the library in Swift.

We don't need to import anything. We should be able to start using the Swift bindings which will start talking to the Rust code.

```swift
let todolist = TodoList()
todolist.addEntry(entry: TodoListEntry(text: "Make uniffi work on iOS"))
```

When you need to make a change to the Rust API, add the Rust parts, then the UDL, and Xcode should pick up the changes on the next build.

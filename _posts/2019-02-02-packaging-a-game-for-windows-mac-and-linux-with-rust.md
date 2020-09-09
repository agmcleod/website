---
title: Packaging a game for Windows, Mac, and Linux with Rust.
date: 2019-02-02 00:00:00
layout: post
---

Building a cross platform game for desktop operating systems in Rust is fairly doable without needing much platform specific code. [Glutin][0] is a Rust alternative to SDL for handling window creation & input. [GFX][1] handles most of the graphics API abstraction for you. You still write the shaders, but I was able to just use OpenGL and get it working on Windows 10, MacOS, and Ubuntu.

## The File System

When you build a game, you load a lot of files. Images, sound clips, music, data files, etc. How you do this, and pull them relative to the game's location on the file system depends on the engine & programming language you use. Typically with JavaScript and HTML5, you run a web server, and pass absolute URLs to some remote images on S3.

With ruby you can use `require_relative` and that will include ruby scripts relative to that script file. If you use `IO::load` to load some text file, a string path will be relative to where you execute the script from. So if you call `ruby somedir/myscript.rb` it will check for files relative to your current directory, not `somedir/`.

Rust file loading works in a similar way, it loads files relative to where you call the binary from. A typical rust project looks like so:

    resources/
      images/
    src/
    Cargo.toml

From this directory, you use `cargo run` to compile & run the game. This is pretty normal for any rust project. Then lets say I load up a texture:

    let image = File::load("resources/images/texture.png");

That'll work fine. However, when you package it into a .app for MacOS, it falls apart. This is because of the way .app is structured. For EnergyGrid, it looks like this:

![MacOS folder layout in Mac binary](/assets/packing-rust-mac.png)

Everything is inside `Contents/MacOS`, so that means it doesn't check right folder for `resources/`. The fix I found is to use `std::env::current_exe()`, which I then created a little utility function for:

```rust
pub fn get_exe_path() -> PathBuf {
    match env::current_exe() {
        Ok(mut p) => {
            p.pop();
            p
        }
        Err(_) => PathBuf::new(),
    }
}
```

The pop() removes the filename of the executable, we just want the path to the current directory. From there, I was able to create code for loading images. So if I wanted to load something at `resources/images/texture.png`, I could call:

```rust
let path = get_exe_path().join("resources/images/texture.png");
```

Then pass that to the image loader. However this meant that `cargo run` on its own would not work. Whenever I updated stuff inside the resources directory, I would then copy it into `target/debug/` so the executable could find the files.

## What about packaging the app?

So Windows compiles a rust binary into a .exe, so that's pretty much done for us, not much packaging needed. MacOS & Linux require a bit of extra work.

### First on Mac

Most apps on mac are installed using the dmg format, which mounts on your system like a drive, and gives you a UI for dragging the app into the /Applications folder. For creating the .app folder and creating the dmg file, I borrowed most of the following bash script from [another person in the Rust community](https://github.com/stevebob/howl/blob/master/scripts/build_macos.sh). You can find the original here:

```bash
#!/bin/bash

set -e

APP_NAME=ld39
MACOS_BIN_NAME=energygrid-bin
MACOS_APP_NAME=EnergyGrid
MACOS_APP_DIR=$MACOS_APP_NAME.app

mkdir -p macbuild
cd macbuild
echo "Creating app directory structure"
rm -rf $MACOS_APP_NAME
rm -rf $MACOS_APP_DIR
mkdir -p $MACOS_APP_DIR/Contents/MacOS

cargo rustc \
    --verbose \
    --release

echo "Copying binary"
MACOS_APP_BIN=$MACOS_APP_DIR/Contents/MacOS/$MACOS_BIN_NAME
cp ../target/release/$APP_NAME $MACOS_APP_BIN

echo "Copying resources directory"
cp -r ../resources $MACOS_APP_DIR/Contents/MacOS

echo "Copying launcher"
cp ../scripts/macos_launch.sh $MACOS_APP_DIR/Contents/MacOS/$MACOS_APP_NAME

echo "Copying Icon"
mkdir -p $MACOS_APP_DIR/Contents/Resources
mv ../resources/Info.plist $MACOS_APP_DIR/Contents/
mv ../resources/logo.icns $MACOS_APP_DIR/Contents/Resources/

echo "Creating dmg"
mkdir -p $MACOS_APP_NAME
cp -r $MACOS_APP_DIR $MACOS_APP_NAME/
rm -rf $MACOS_APP_NAME/.Trashes

FULL_NAME=$MACOS_APP_NAME

hdiutil create $FULL_NAME.dmg -srcfolder $MACOS_APP_NAME -ov
rm -rf $MACOS_APP_NAME
```

It first creates the directory structure, then builds a binary. It copies over the binary, the resources, the icon file & Info.plist (which is used to specify the icon file), and then creates the dmg. The `macos_launch.sh` launch script looks like so:

```
#!/bin/bash

DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )"
BIN=energygrid-bin

$DIR/$BIN
```

It ensures that we are running things from the right directory, and that all the hard work on the file organization wasn't for naught.

### Windows

Windows is actually a bit easier (surprise, right?), provided you have a windows installation lying around. The only tricky thing was the Icon, which I used: [RCEdit](http://github.com/electron/rcedit/) to help with. The script is as follows:

```bash
#!/bin/bash

rm -rf winbuild/
./scripts/copy_resources.sh
cargo rustc --release -- -Clink-args="/SUBSYSTEM:WINDOWS /ENTRY:mainCRTStartup"
mkdir -p winbuild
cp -r resources winbuild/
cp target/release/ld39.exe winbuild/EnerygGrid.exe
mv winbuild/resources/logo.ico winbuild/logo.ico

cd winbuild
# Expects https://github.com/electron/rcedit to be on Path
rcedit "EnerygGrid.exe" --set-icon "logo.ico"
rm logo.ico
```

The `-Clink-args` option for the compile is so that it doesn't launch a CMD window when running the game.


### Linux

I'm not a Linux guru, I mostly just use it for work on the server. I had some recommendations on the Rust subreddit to use [AppImage](https://appimage.org/) which is a sorta similar format to that of .app on MacOS. Once you install it on the system, you just need a manifest file and some bash commands. Here's the script:

```bash
#!/bin/bash

rm *.AppImage
rm -rf EnergyGrid.AppDir
cargo build --release
mkdir -p EnergyGrid.AppDir
cp -r resources EnergyGrid.AppDir

cp target/release/ld39 EnergyGrid.AppDir/AppRun

cd EnergyGrid.AppDir
mv resources/logo.png energygrid.png

echo '[Desktop Entry]' > energygrid.desktop
echo 'Name=EnergyGrid' >> energygrid.desktop
echo 'Exec=EnergyGrid' >> energygrid.desktop
echo 'Icon=energygrid' >> energygrid.desktop
echo 'Type=Application' >> energygrid.desktop
echo 'Categories=Game;' >> energygrid.desktop

cd ..
appimagetool-x86_64.AppImage EnergyGrid.AppDir
```

The first group is just my usual file cleanup code, building the game, and creating the directory structure. From there I create the manifest file, which is based on their documentation. After that you just use the app image tool command to package it and it's ready.

## Uh Oh, A Bug!

A kind user last week posted on the [Energy Grid](https://agmcleod.itch.io/energygrid) store page that the game crashed when they would change any settings on the Linux build, and very kindly posted the stack trace. The issue was that my settings code would write a JSON file in the exe folder. AppImage folders are read only! Do'h! So today I worked on a fix, and the answer was to use different paths for the operating systems. I basically left Windows & MacOS untouched, and used attributes to have a specific method for linux.

```rust
#[cfg(target_os="linux")]
pub fn get_settings_path() -> PathBuf {
    if let Some(home_dir) = dirs::home_dir() {
        if !home_dir.join("EnergyGrid").exists() {
            create_dir(home_dir.join("EnergyGrid")).unwrap();
        }
        home_dir.join("EnergyGrid").join("settings.json")
    } else {
        panic!("Could not find $HOME");
    }
}

#[cfg(target_os="windows")]
pub fn get_settings_path() -> PathBuf {
    get_settings_path_win_mac()
}

#[cfg(target_os="macos")]
pub fn get_settings_path() -> PathBuf {
    get_settings_path_win_mac()
}

fn get_settings_path_win_mac() -> PathBuf {
    get_exe_path().join("settings.json")
}
```

Mac & Windows share the same function there, but linux tries to create a directory in the home folder, and put the settings in there. The save() function in the settings lib I have then calls that function to determine where to write the JSON file to. After that I re-built the AppImage and confirmed the fix.
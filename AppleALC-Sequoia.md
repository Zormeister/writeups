# AppleALC and the great compiler-update of macOS Sequoia

Do note that I wasn't the *first* person to do this. I'm the first to have published it to GitHub without having *cough* deleted their code *cough*

With that out of the way, let's set the scene.

You eagerly update to macOS Sequoia on your AMD hackintosh, somehow despite all of the abominable hacks (AirportItlwm and OCLP to name one)

Oh no! Your audio isnt working? How could this be???

## So... what exactly changed?

Apple updated their compilers.

The byte pattern in the 2nd patch for Zen controllers would no longer match to macOS Sequoia.

- ```Find: 3D 02 10 00 00 74```
- ```Replace: 3D 22 10 00 00 7E```

The above patch broke due to a change in the compiler, because if we assemble this...

We get:

- ```cmp eax, 0x1002```
- ```je <missing>```

The patch itself replaces the 0x1002 with 0x1022, and changes the `je` to a `jle`

After having located the original pattern, which leads us to `AppleHDAController::setupHostInterface`, I was on my way for making a patch for Sequoia

## The new patch?

It was pretty simple to work out actually, everything except the last byte in the origitnal pattern is the same!

Why?

The compiler opted to use the near-jump variants of jump opcodes, which explains the byte pattern not matching.

The end result gathered was

- ```Find: 3D 02 10 00 00 0F 84```
- ```Replace: 3D 02 10 00 00 0F 8E```

Thus marked my great journey of testing!

Luckily, I managed to knock two devices out early since I had devices with said controllers, however I still had one left to go.
As of the same day of writing, the patch was tested on a `1022:1487` device, proving it worked on *every* device!

You can find the PR for this [here!](https://github.com/acidanthera/AppleALC/pull/923)

## Handy Tools

A massive helping tool was `native-sigscan` for Binary Ninja, allowing me to quickly lookup byte patterns

## Conclusion

RE is the goat

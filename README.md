# gradle-libpath
Example of PR change impact

### How to use
Steps to produce:
* Run "./gradlew build" on the checkout and generate a shared library (library) & cpp application (main)
* Navigate to the install directory: "gradle-libpath/main/build/install/debug/lib"
```
/gradle-libpath/main/build/install/main/debug/lib> ls
liblibrary.so  main
```
* Run the linux "ldd" command on the application (ldd main) to see the shared libraries the application needs:
```
/gradle-libpath/main/build/install/main/debug/lib> ldd main
          linux-vdso.so.1 =>  (0x00007ffc88dac000)
          /gradle-libpath/library/build/lib/main/debug/liblibrary.so (0x00007f774b0ad000)
          libstdc++.so.6 => /usr/lib64/libstdc++.so.6 (0x0000003912600000)
          libm.so.6 => /lib64/libm.so.6 (0x0000003910a00000)
```
* Notice the "hard path link" to "liblibrary.so" as opposed to the ones leveraging the library path to find the library "=>".
* Set the library path [LD_LIBRARY_PATH](http://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html) to a different file location such as "/gradle-libpath/library-on-library-path" to pick up the liblibrary.so. (export LD_LIBRARY_PATH=/gradle-libpath/library-on-library-path:$LD_LIBRARY_PATH)
* Rerun the "ldd" command on the application (ldd main) to see it still preferring the hard link to gradle-libpath/library/build/lib/main/debug/liblibrary.so instead of leveraging the library path.

Note the install posix script "gradle-libpath/main/build/install/main/debug/main" also attempts to set the LD_LIBRARY_PATH but given it's a hard link to that directory, it's not doing anything.

### Problem this causes
If we deploy what gradle produces to another machine, it will continue to look for the library on that absolute path. (Aka, on a production machine, it "ldd main" would look for "/gradle-libpath/main/build/lib/main/debug/liblibrary.so" which makes it less portable and troublesome to install.

### With the PR version...
If you use with the PR version https://github.com/gradle/gradle/pull/6176, you'll notice that it switches to leveraging the library path.
```
/gradle-libpath/main/build/install/main/debug/lib> ldd main
        linux-vdso.so.1 =>  (0x00007ffed99c0000)
        liblibrary.so => /gradle-libpath/main/build/install/main/debug/lib/liblibrary.so (0x00007f8704d69000)
        libstdc++.so.6 => /usr/lib64/libstdc++.so.6 (0x00007f8704a5a000)
        libm.so.6 => /lib64/libm.so.6 (0x00007f87047d6000)
```
Likewise, if we change the LD_LIBRARY_PATH (i.e. on a production server) then it checks the new location for the binary.
```
/>export LD_LIBRARY_PATH=/gradle-libpath/library-on-library-path:$LD_LIBRARY_PATH
/gradle-libpath/main/build/install/main/debug/lib> ldd main
        linux-vdso.so.1 =>  (0x00007ffdae9cc000)
        liblibrary.so => /gradle-libpath/library-on-library-path/liblibrary.so (0x00007f65e012e000)
        libstdc++.so.6 => /usr/lib64/libstdc++.so.6 (0x00007f65dfe1f000)
        libm.so.6 => /lib64/libm.so.6 (0x00007f65dfb9b000)
```
The good news is the build (including tests) are already setting the LD_LIBRARY_PATH to the correct directory so this doesn't impact it. (It does impact manually running the produced binary without setting your LD_LIBRARY_PATH first / makes you want to use the provided posix shell script).

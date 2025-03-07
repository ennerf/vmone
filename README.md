Build on linux:

```
make clean all
```

Build on Android:

```
make TARGET=android clean all
```

Build on iOS:

```
make TARGET=ios clean all
```

Build on Windows:

```
md build
cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -G "NMake Makefiles"
nmake
```
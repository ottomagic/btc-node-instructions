# Fulcrum installation

Fulcrum is a performant electrum server alternative.

Code repository: https://github.com/cculianu/Fulcrum

Performance: https://www.sparrowwallet.com/docs/server-performance.html


## 1. Clone the git repository

```
git clone https://github.com/cculianu/Fulcrum.git
cd Fulcrum
```

Checkout the correct version tag:
```
git checkout tags/v1.9.1 -b tags/v1.9.1
```


## 2. Install build dependencies

```
sudo apt update
sudo apt install -y libqt5core5a libqt5network5 qtbase5-dev qt5-qmake libbz2-dev zlib1g-dev
```

Install optional/recommended dependencies (zmq libraries):
```
sudo apt install libzmq3-dev
```


## 3. Build

To generate the Makefile:
```
qmake
```

Replace 6 here with the number of cores on your machine:
```
make -j6
```

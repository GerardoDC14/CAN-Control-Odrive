# Linux Driver Files

This workspace includes the Linux native Ginkgo driver binaries in:

- `64bit/`
- `32bit/`

Expected files:

- `libGinkgo_Driver.so`
- `libusb-1.0.so`
- `libusb.so`

The standalone tools and the ROS 2 bridge look for the Linux shared libraries in these directories automatically.

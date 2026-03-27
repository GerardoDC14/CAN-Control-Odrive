# Linux Driver Files

The current copied vendor bundle does not include the Linux native Ginkgo driver binaries.

If you want to use the Ginkgo USB-CAN adapter from Linux in this workspace, add the vendor-provided files here:

- `64bit/libGinkgo_Driver.so`
- `64bit/libusb-1.0.so` if your vendor bundle provides it
- `64bit/libusb.so` if your vendor bundle provides it

If you need 32-bit support, place the matching binaries under:

- `32bit/`

The standalone tools and ROS bridge will look for the Linux shared libraries in these directories.

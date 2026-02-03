1. Định nghĩa & Khái niệm cơ bản
Trước khi đi vào thực hiện, cần nắm rõ các thành phần cốt lõi:

Toolchain (Cross-Compiler): Bộ công cụ chạy trên máy tính (Host - x86_64) dùng để biên dịch mã nguồn thành mã máy chạy trên thiết bị nhúng (Target - ARM). Trong bài này sử dụng arm-unknown-eabi-gcc.

U-Boot (Universal Bootloader): Trình nạp khởi động phổ biến nhất cho Linux nhúng.

MLO (Secondary Program Loader - SPL): Chương trình khởi động nhỏ đầu tiên chạy từ ROM của chip, có nhiệm vụ khởi tạo RAM và load U-Boot chính.

u-boot.img: U-Boot chính, cung cấp giao diện dòng lệnh (CLI) để cấu hình và nạp Kernel.

Linux Kernel (zImage): "Trái tim" của hệ điều hành, quản lý phần cứng (CPU, RAM, I/O) và cung cấp dịch vụ cho phần mềm. zImage là định dạng nén của Kernel giúp tiết kiệm bộ nhớ khi boot.

Device Tree Blob (.dtb): File nhị phân mô tả cấu trúc phần cứng (địa chỉ RAM, GPIO, I2C, SPI...) để Kernel hiểu và điều khiển được bo mạch cụ thể (ở đây là AM335x BoneBlack).

bootz: Lệnh của U-Boot dùng để khởi động zImage.

🛠 2. Yêu cầu phần cứng & Công cụ
Bo mạch: BeagleBone Black (AM335x ARM Cortex-A8).

Máy tính: Ubuntu Linux (Host).

Kết nối: Cáp USB-UART (Debug) để giao tiếp Serial Console.

Lưu trữ: Thẻ nhớ MicroSD (được chia ít nhất 2 phân vùng: Boot & Rootfs).

🚀 3. Phần 1: Biên dịch và Cài đặt U-Boot
Bước 1: Biên dịch U-Boot
Sử dụng Toolchain đã cấu hình để build source code U-Boot.

Lệnh thực hiện:

Bash
# Clean build cũ
make distclean

# Cấu hình cho BeagleBone Black
make am335x_evm_defconfig

# Biên dịch (tạo ra MLO và u-boot.img)
make -j$(nproc)
✅ Dấu hiệu thành công:

Trong thư mục source xuất hiện file MLO và u-boot.img.

Lệnh file u-boot.img báo u-boot legacy image... ARM.

Bước 2: Cài đặt U-Boot lên thẻ nhớ
Copy file vào phân vùng Boot (FAT32) của thẻ nhớ.

Lệnh thực hiện:

Bash
# Mount thẻ nhớ (Ví dụ thẻ là /dev/sdX1)
sudo cp MLO /media/user/BOOT/
sudo cp u-boot.img /media/user/BOOT/
Bước 3: Kiểm thử trên Hardware (Debug UART)
Kết nối cáp USB-TTL vào Header J1 của BBB (GND-Pin1, RX-Pin4, TX-Pin5).

Mở Terminal (Picocom/Minicom): sudo picocom -b 115200 /dev/ttyUSB0.

Giữ nút S2 và cấp nguồn.

✅ Dấu hiệu thành công:

Terminal hiện log khởi động của U-Boot.

Nhấn phím Space để vào chế độ dòng lệnh =>.

Gõ lệnh version hiện thông tin ngày giờ build mới nhất.

Gõ lệnh bdinfo hiện thông tin phần cứng (DRAM bank, arch_number).

(Chèn ảnh màn hình U-Boot version và bdinfo tại đây)

🐧 4. Phần 2: Biên dịch và Cài đặt Kernel
Bước 1: Biên dịch Kernel & Device Tree
Biên dịch Kernel phiên bản 6.6.14.

Lệnh thực hiện:

Bash
# Cấu hình mặc định cho chip TI OMAP2+
make ARCH=arm CROSS_COMPILE=arm-unknown-linux-gnueabi- multi_v7_defconfig

# Biên dịch Kernel (zImage) và Device Tree (dtbs)
make ARCH=arm CROSS_COMPILE=arm-unknown-linux-gnueabi- -j$(nproc) zImage dtbs modules
✅ Dấu hiệu thành công:

Tạo thành công file: arch/arm/boot/zImage.

Tạo thành công file: arch/arm/boot/dts/ti/omap/am335x-boneblack.dtb.

Kiểm tra bằng lệnh ls -lh thấy kích thước zImage khoảng 5-10MB.

Bước 2: Nạp Kernel vào thẻ nhớ
Copy Kernel và DTB vào thư mục /boot trên phân vùng RootFS (ext4) của thẻ nhớ.

Lệnh thực hiện (trên BBB hoặc Host):

Bash
# Tạo thư mục boot
sudo mkdir -p /media/rootfs/boot/

# Copy file
sudo cp arch/arm/boot/zImage /media/rootfs/boot/
sudo cp arch/arm/boot/dts/ti/omap/am335x-boneblack.dtb /media/rootfs/boot/
Bước 3: Boot Kernel từ U-Boot
Cấu hình boot arguments và load file thủ công vào RAM để khởi động.

Các lệnh thực hiện tại dấu nhắc => của U-Boot:

Thiết lập bootargs (Quan trọng):

Bash
setenv bootargs console=ttyO0,115200n8 root=/dev/mmcblk0p2 rw rootfstype=ext4 rootwait
Load Kernel vào RAM (0x82000000):

Bash
load mmc 0:2 0x82000000 /boot/zImage
Load Device Tree vào RAM (0x88000000):

Bash
load mmc 0:2 0x88000000 /boot/am335x-boneblack.dtb
Khởi động (Boot):

Bash
bootz 0x82000000 - 0x88000000
Dấu hiệu thành công:

Màn hình hiện dòng chữ Starting kernel ....

Log khởi động của Linux chạy liên tục.

Hiển thị đúng phiên bản: Linux version 6.6.14 ....

Cuối cùng dừng lại ở thông báo chờ RootFS (hoặc Kernel Panic nếu chưa có RootFS đầy đủ) -> Điều này chứng tỏ Kernel đã chạy thành công.

(Chèn ảnh màn hình log "Starting kernel..." và "Linux version" tại đây)


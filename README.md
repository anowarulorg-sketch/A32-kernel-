name: Build SM-A325N APatch Kernel

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Install Build Dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y build-essential bc bison flex libssl-dev libelf-dev \
            python3 python-is-python3 curl wget zip tar gzip cpio kmod \
            gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu
          sudo apt-get install -y python2 || true

      - name: Download Kernel.tar.gz from Release
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          mkdir -p kernel_src
          gh release download v1.0 --pattern "Kernel.tar.gz" --repo ${{ github.repository }}
          tar -xzf Kernel.tar.gz -C kernel_src

      - name: Download Samsung Toolchains (clang-r383902 & gcc-4.9)
        run: |
          mkdir -p toolchains/clang
          wget -qO- https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/master-kernel-build-2021/clang-r383902.tar.gz | tar -xz -C toolchains/clang
          git clone --depth=1 https://github.com/LineageOS/android_prebuilts_gcc_linux-x86_aarch64_aarch64-linux-android-4.9 toolchains/gcc64

      - name: Configure and Compile Kernel
        run: |
          export WORKDIR=$(pwd)
          export PATH="$WORKDIR/toolchains/clang/bin:$WORKDIR/toolchains/gcc64/bin:$PATH"
          
          # সরাসরি a32ks_defconfig ফাইলটির লোকেশন খুঁজে বের করা
          DEFCONFIG_PATH=$(find kernel_src -name "a32ks_defconfig" -print -quit)
          if [ -z "$DEFCONFIG_PATH" ]; then
            echo "a32ks_defconfig পাওয়া যায়নি! ভেতরের ফাইলগুলোর তালিকা:"
            find kernel_src -maxdepth 3
            find kernel_src -name "*a32*defconfig*"
            exit 1
          fi
          
          echo "Found defconfig at: $DEFCONFIG_PATH"
          KERNEL_DIR=$(echo "$DEFCONFIG_PATH" | sed 's|/arch/arm64/configs/.*||')
          echo "Entering Kernel Directory: $KERNEL_DIR"
          cd "$KERNEL_DIR"
          
          # APatch এর জন্য KALLSYMS চালু এবং Samsung সিকিউরিটি লক বন্ধ করা
          REL_DEFCONFIG=$(find arch/arm64/configs -name "a32ks_defconfig" -print -quit)
          echo "CONFIG_KALLSYMS=y" >> "$REL_DEFCONFIG"
          echo "CONFIG_KALLSYMS_ALL=y" >> "$REL_DEFCONFIG"
          echo "CONFIG_UH=n" >> "$REL_DEFCONFIG"
          echo "CONFIG_RUSTUH_RKP=n" >> "$REL_DEFCONFIG"
          echo "CONFIG_RUSTUH_KDP=n" >> "$REL_DEFCONFIG"
          echo "CONFIG_RUSTUH_KDP_CRED=n" >> "$REL_DEFCONFIG"
          echo "CONFIG_RUSTUH_KDP_NS=n" >> "$REL_DEFCONFIG"
          echo "CONFIG_SECURITY_DEFEX=n" >> "$REL_DEFCONFIG"
          echo "CONFIG_PROCA=n" >> "$REL_DEFCONFIG"
          echo "CONFIG_FIVE=n" >> "$REL_DEFCONFIG"
          
          export ANDROID_MAJOR_VERSION=r
          export ARCH=arm64
          export SUBARCH=arm64
          
          # Step 1: Make defconfig
          make -C $(pwd) O=$(pwd)/out \
            ARCH=arm64 \
            ANDROID_MAJOR_VERSION=r \
            KCFLAGS=-w \
            CONFIG_SECTION_MISMATCH_WARN_ONLY=y \
            CC=clang \
            CLANG_TRIPLE=aarch64-linux-gnu- \
            CROSS_COMPILE=aarch64-linux-androidkernel- \
            a32ks_defconfig
            
          # Step 2: Compile Kernel
          make -j$(nproc --all) -C $(pwd) O=$(pwd)/out \
            ARCH=arm64 \
            ANDROID_MAJOR_VERSION=r \
            KCFLAGS=-w \
            CONFIG_SECTION_MISMATCH_WARN_ONLY=y \
            CC=clang \
            CLANG_TRIPLE=aarch64-linux-gnu- \
            CROSS_COMPILE=aarch64-linux-androidkernel-

      - name: Upload Compiled Image
        uses: actions/upload-artifact@v4
        with:
          name: SM-A325N-Kernel-Image
          path: |
            kernel_src/**/out/arch/arm64/boot/Image.gz
            kernel_src/**/out/arch/arm64/boot/Image
            kernel_src/out/arch/arm64/boot/Image.gz
            kernel_src/out/arch/arm64/boot/Image

**CMPE 283 – Assignment 2 (KVM Statistics)**

**Repo**: https://github.com/joshini-mn/linux

**Key file edited**: arch/x86/kvm/vmx/vmx.c

**Commits** containing the code changes:

abed91fb9c5ae60e239497354c37ffd8deeeb8e1,
dcf082fad3f6f36c3299103f2abd48e885c6c13d,
dd3326d5b861506a126f70bde4f60883e2761203,
5643d4833e6c3a65e08031cda7cae746baa64ce3

**1) I've worked individually for this assignment.**

**2) Reproducible Steps (outer-VM host on GCP, inner-VM)**

A. Prepare build environment on the outer-vm
sudo apt-get update

sudo apt-get install -y build-essential flex bison libncurses-dev libssl-dev bc \
                        dwarves libelf-dev cpu-checker libvirt-daemon-system \
                        libvirt-daemon-config-network qemu-kvm virtinst

Verify nested KVM is usable:

kvm-ok   # expect: "KVM acceleration can be used"

B. Build a custom kernel from this repo

From repo root (~/linux):

cp -v /boot/config-$(uname -r) .config || true

yes "" | make oldconfig

make -j"$(nproc)"

sudo make modules_install

sudo make install

sudo update-grub

sudo reboot

After reboot, confirm the intended kernel:

uname -r

<img width="600" height="90" alt="image" src="https://github.com/user-attachments/assets/188ec7e8-7d64-4870-9316-d66b9d3bf74d" />


If the system still boots an older kernel, set the default entry and update GRUB:

sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT=saved/' /etc/default/grub

sudo grub-set-default 'Ubuntu, with Linux <your-version>'

sudo update-grub

sudo reboot


C. Implement human-readable VM-exit names in vmx.c

Add this helper near other static helpers at the top of vmx.c.
Looks at the code changes in Commits provided.

D. Rebuild just KVM and reload the module (faster inner-loop)

cd ~/linux

sudo make -j"$(nproc)" M=arch/x86/kvm modules

sudo make M=arch/x86/kvm modules_install

Ensure the inner VM is not using KVM modules before unloading:
sudo systemctl stop libvirtd || true
sudo rmmod kvm_intel kvm 2>/dev/null || true

Load freshly installed modules from the current kernel

sudo modprobe kvm

sudo modprobe kvm_intel


E. Start the inner-vm and capture logs

sudo systemctl enable --now libvirtd

sudo virsh start inner-vm

sudo virsh list --all   # inner-vm should be "running"

<img width="860" height="248" alt="image" src="https://github.com/user-attachments/assets/31faa27e-5f9a-47c1-a193-a9664c1ecc25" />

F. Capture the stream:

sudo dmesg -w | grep -E "CMPE283: (VMEXIT totals|exit)"

The logs should print exit reasons like this:
<img width="940" height="450" alt="image" src="https://github.com/user-attachments/assets/f1cb9009-dce6-44b3-9fba-12604a4a75c8" />

**3) Exit Frequency Commentary**

Boot vs. steady-state: Exits spike during boot due to frequent CPUID, MSR access, EPT faults (first-touch mappings), and device initialization I/O. After the guest settles, exit growth rate flattens and becomes workload-dependent.
Operational bursts: Process launches, heavy filesystem or package installs, and network activity cause bursts (lots of EPT activity, MSR/CR/IO exits).

**4) Most/Least Frequent Exit Types (observed)**

From the running logs:

Most frequent (typical):
CPUID (reason 10): very common during boot and userland bring-up.
EPT_VIOLATION (48): dominant during page population and I/O heavy work.
MSR/CR/IO group: CONTROL_REGISTER_ACCESS (28), MSR_READ (30), MSR_WRITE (31), IO_INSTRUCTION (32).

Least frequent (typical in a normal Linux guest):
APIC_WRITE (54), INVPCID (55), and other rare control-path exits.
SMI/SIPI/VMXON/VMXOFF reasons aren’t observed during ordinary guest operation.

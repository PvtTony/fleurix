cinc   = '-Isrc/inc -Isrc/inc/usr'
cflag  = %{
  -m32
  -Wall
  -nostdinc -fno-builtin -fno-stack-protector
  -finline-functions -finline-small-functions -findirect-inlining -finline-functions -finline-functions-called-once
}.split(/\s/).join(' ')

mkdir_p 'bin'
mkdir_p 'bin/usr'
mkdir_p 'bin/libsys'
mkdir_p 'root/bin'
mkdir_p 'root/dev'

# ----------------------------------------------------------

task :default => [:bochs]

task :bochs => :build do
  sh "bochs -q -f .bochsrc"
end

task :nm => :build do
  sh 'cat main.nmtab'
end

task :debug => :build do
  sh "bochs-dbg -q -f .bochsrc"
end

task :build => ['bin/kernel.img', :rootfs, :ctags]

task :clean do
  sh "rm -rf bin/* root/bin/* .bochsout"
end

task :werr do
  cflag += ' -Werror'
  Rake::Task['clean'].invoke
  Rake::Task['build'].invoke
end

## helpers ##
task :todo do
  sh "grep -r -n '\\(TODO\\|\\<bug\\>\\)' ./src --color"
end

task :ctags do
  sh "(cd src; ctags -R .)"
end

task :fsck do
  sh "fsck.minix -fl ./bin/rootfs.img"
end

# --------------------------------------------------------------------
# kernel.img = boot.bin + main.bin

# the kernel image, a concat of boot image and the main binary.
file 'bin/kernel.img' => ['bin/boot.bin', 'bin/main.bin'] do
  sh "cat bin/boot.bin bin/main.bin > bin/kernel.img"
end

# --------------------------------------------------------------------
# the bootloader part
# => boot.bin

file 'bin/boot.o' => ['src/boot/boot.S'] do
  sh "nasm -f elf -o bin/boot.o src/boot/boot.S"
end

file 'bin/boot.bin' => ['bin/boot.o', 'tool/boot.ld'] do
  sh "ld -m elf_i386 bin/boot.o -o bin/boot.bin -e c -T tool/boot.ld"
end

# ---------------------------------------------------------------------
# kernel's C part
# => main.bin

hfiles = Dir['src/inc/*.h']
cfiles = Dir['src/**/*.c']
sfiles = [
  'src/kern/entry.S'
]
ofiles = (sfiles + cfiles).map{|fn| 'bin/'+File.basename(fn).ext('o') }

cfiles.each do |fn_c|
  fn_o = 'bin/'+File.basename(fn_c).ext('o')
  file fn_o => [fn_c, *hfiles] do
    sh "gcc #{cflag} #{cinc} -o #{fn_o} -c #{fn_c} 2>&1"
  end
end

sfiles.each do |fn_s|
  fn_o = 'bin/'+File.basename(fn_s).ext('o')
  file fn_o => [fn_s, *hfiles] do
    sh "nasm -f elf -o #{fn_o} #{fn_s}"
  end
end

# ---------------------------------------------------------------------3

file 'bin/main.bin' => 'bin/main.elf' do
  sh "objcopy -R .pdr -R .comment -R .note -S -O binary bin/main.elf bin/main.bin"
end

file 'bin/main.elf' => ofiles + ['tool/main.ld'] do
  sh "ld -m elf_i386 #{ofiles * ' '} -o bin/main.elf -e c -T tool/main.ld"
  sh "(nm bin/main.elf | sort) > main.sym"
end

# ----------------------------------------------------------------------
# the rootfs part.
# => rootfs.img

# the root file system aka the hard disk image, 1mb yet and ignored
# partition
# note: on mounting, a root privilege is required.
task :rootfs => ['bin/rootfs.img']

# init and copy some thing into the hard disk image.
file 'bin/rootfs.img' => [:usr] do
  `rm -f bin/rootfs.img`
  sh "bximage bin/rootfs.img -hd=10M -imgmode=flat -mode=create -q"
  sh "mkfs.minix bin/rootfs.img"
  mkdir_p '/tmp/fx_mnt_root'
  `sudo umount /tmp/fx_mnt_root`
  sh "sudo mount -o loop -t minix bin/rootfs.img /tmp/fx_mnt_root"
  sh "sudo cp -r ./root/* /tmp/fx_mnt_root"
  `sudo mknod /tmp/fx_mnt_root/dev/tty0 c 1 0`
  sh "sudo umount /tmp/fx_mnt_root"
  sh "rm -rf /tmp/fx_mnt_root"
end

# ----------------------------------------------------------------------

usr_cfiles = Dir['usr/test/*.c'] + Dir['usr/*.c']
usr_ofiles = usr_cfiles.map{|fn| 'bin/usr/'+File.basename(fn).ext('o') }
usr_efiles = usr_cfiles.map{|fn| 'bin/usr/'+File.basename(fn).ext('') }

libsys_cfiles = Dir['usr/libsys/*.c']
libsys_sfiles = %w{
  usr/libsys/entry.S
}
libsys_ofiles = (libsys_sfiles+libsys_cfiles).map{|fn| 'bin/libsys/'+File.basename(fn).ext('o') }

task :libsys => libsys_ofiles

task :usr => usr_efiles

# => libsys.a
libsys_sfiles.each do |fn_s|
  fn_o = 'bin/libsys/'+File.basename(fn_s).ext('o')
  file fn_o => fn_s do
    sh "nasm -f elf -o #{fn_o} #{fn_s}"
  end
end

libsys_cfiles.each do |fn_c|
  fn = File.basename(fn_c).ext('')
  fn_o = 'bin/libsys/'+fn.ext('o')
  file fn_o => fn_c do
    sh "gcc -m32 -c #{cinc} -nostdinc -fno-builtin -fno-stack-protector #{fn_c} -o #{fn_o}"
  end
end

usr_cfiles.each do |fn_c|
  fn = File.basename(fn_c).ext('')
  fn_o = 'bin/usr/'+fn.ext('o')
  fn_e = 'bin/usr/'+fn
  file fn_e => [fn_c, :libsys, 'tool/user.ld'] do
    sh "gcc -m32 -c #{cinc} -nostdinc -fno-builtin -fno-stack-protector #{fn_c} -o #{fn_o}"
    sh "ld -m elf_i386 #{libsys_ofiles*' '} #{fn_o} -o #{fn_e} -e c -T tool/user.ld"
    sh "nm #{fn_e} > #{fn_o.ext('sym')}"
    sh "objdump -S #{fn_e} > #{fn_e.ext('S')}"
    sh "cp #{fn_e} root/bin/#{fn}"
  end
end

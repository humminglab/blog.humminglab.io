---
title: SD card 디스크 이미지 만들고 수정하는 방법 정리
date: "2017-10-25T18:00:00+09:00"
lastmod: "2017-10-25T18:00:00+09:00"
draft: false
authors: ["YSLee"]
tags: ["Linux"]
categories: ["Embedded Linux"]
aliases: [/how-to-make-sdcard-disk-image/]
---

[Yocto Project](https://www.yoctoproject.org)나 [Buildroot](https://buildroot.org)를 이용하여 embedded linux 시스템을 빌드하면 SD card나 MMC에 쓸수 있는 이미지까지 생성해 준다. 하지만 빌드되는 디스크 이미지 형태와 다르게 파티셔닝을 하려면 관련된 정보들을 알고 있어야 한다.

여기에서는 [dd](https://linux.die.net/man/1/dd), [truncate](https://linux.die.net/man/1/truncate), [fdisk](https://linux.die.net/man/8/fdisk), [parted](https://www.gnu.org/software/parted/), [mount](https://linux.die.net/man/8/mount), [losetup](https://linux.die.net/man/8/losetup) 등의 utility를 이용하여 디스크 이미지를 생성, 수정, 관리하는 방법을 정리한다.

## 물리적인 저장 디스크 관리

Linux의 경우 저장 디스크는 block device로 /dev 디렉토리에 아래와 같은 디바이스 파일이 생성된다. 아래의 예는 sda SSD 디스크로 한 개의 파티션(sda1)이 있다.

```sh
$ ls -l /dev/sd*
brw-rw---- 1 root disk 8, 0 10월 24 13:40 /dev/sda
brw-rw---- 1 root disk 8, 1 10월 24 13:40 /dev/sda1
```

해당 디스크의 파티션을 보거나 수정하려면 [fdisk](https://linux.die.net/man/8/fdisk)나 [parted](https://www.gnu.org/software/parted/)를 사용할 수 있다. 일반적인 용도로 사용시에는 크게 차이는 없으나 script로 배치 작업을 하기 위하여는 parted를 사용하는 것이 좋다.
```sh
$ sudo parted -s /dev/sda print
Model: VMware, VMware Virtual S (scsi)
Disk /dev/sda: 118GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End    Size   Type     File system  Flags
 1      1049kB  118GB  118GB  primary  ext4         boot
 ```

- 디스크를 엑세스 하는 것이므로 root 권한 필요
- -s: 스크립트 모드. print command를 실행

생성된 파티션에 이미지를 쓰기 위하여는 dd를 이용할 수 있다.
```sh
$ sudo dd if=image.ext4 of=/dev/sda1
```

dd의 각각의 필드는 다음과 같다.
- if: input file
- of: output file. 실제 write할 디스크 파티션

dd로 쓴 파티션은 mount를 이용하여 원하는 디렉토리에 마운트하여 사용할 수 있다. 아래의 예는 EXT4 형식의 파일 시스템인 /dev/sda1를 /media 디렉토리로 마운트 한 것이다.
```sh
$ mount -t ext4 /dev/sda1 /media
```

## 디스크 이미지

SD card나 MMC는 저장 방식은 다를 수 있지만 실제 사용 관점에서는 일반 하드디스크와 동일한 개념의 block device라고 보면 된다. 즉, sector 단위로 나뉘어진 공간에 읽거나 쓸수 있다.

초기화 되지 않은 SD card나 MMC는 위의 방법처럼 디스크에 직접 직접 쓸 수 있지만, 임베디드 시스템 개발 과정에서는 빌드 시 저장 장치에 쓸 디스크 이미지를 만들고, 이를 SD card나 MMC에 통으로 쓰는 경우도 많다.

이와 같은 가상의 디스크 이미지를 만드는 것도 직접 디스크에 쓰는 과정과 동일하다. 하나의 차이점이 있다면 물리적인 디스크 공간은 늘거나 줄지 않지만, 가상의 디스크 이미지는 파일 저장 공간만큼 늘리거나 줄일 수 있다는 것이다.

우선 필요한 만큼 파일로 공간을 할당하여야 한다. 아래 예는 1GiB 만큼의 공간을 할당하는 것이다.

```sh
$ dd if=/dev/null of=test.img bs=1kx1k seek=1k
0+0 records in
0+0 records out
0 bytes copied, 0.000174626 s, 0.0 kB/s

$ ls -l
total 1
-rw-rw-r--  1 yslee yslee 1073741824 10월 24 17:51 test.img
```

dd를 이용하여 블럭디바이스에 쓰는 것이나 파일에 쓰는 것이나 동일하다고 보면 된다.

- if=/dev/null: 입력을 null device로 부터 받는다. 즉, 입력이 없음
- of=test.img: test.img로 파일을 생성한다.
- bs=1kx1x: 1KiB x 1KiB, 즉 1MiB 단위로 block size를 설정. Block size는 실제 저장 장치의 block(sector) size 가 아니라 한번에 read 또는 write 하는 단위라고 생각하면 된다.
- seek=1k: 1KiB(1,024) 번째로 이동. 즉 1,024 * 1MiB(bs) = 1GiB 위치가 된다.

위와 같이 입력이 null로 된 상태에서 seek로 위치만 설정해 주면 해당 크기만큼의 파일이 생성된다.

이와 같은 방식으로 생성한 파일은 [sparse file](https://en.wikipedia.org/wiki/Sparse_file)로 Unix 계열이나, NTFS에서 지원된다. Sparse file이란 실제로 해당 공간만큼 물리적인 공간에 할당하는 것이 아니라 실제로 데이타가 있는 부분만 저장하는 방식이다.

![Sparse File](https://upload.wikimedia.org/wikipedia/commons/thumb/9/9f/Sparse_file_%28en%29.svg/495px-Sparse_file_%28en%29.svg.png)

*그림: Sparse Image의 개념([Wikipedia](https://en.wikipedia.org/wiki/Sparse_file))*

실제로 디스크에 할당된 공간은 du를 이용하거나 ls의 -s 옵션을 이용하여 확인할 수 있다.
```sh
$ du -h test.img
0       test.img

$ ls -lhs
4.0K -rw-rw-r--  1 yslee yslee 1.0G 10월 24 18:05 test.img
```

[dd](https://linux.die.net/man/1/dd) 이외에도 [truncate](https://linux.die.net/man/1/truncate)를 이용하여서도 sparse file을 생성할 수 있다.

```sh
$ truncate -s 1G test2.img

$ ls -l
total 2
-rw-rw-r--  1 yslee yslee 1073741824 10월 24 17:51 test.img
-rw-rw-r--  1 yslee yslee 1073741824 10월 24 18:03 test2.img

$ du -h test2.img
0       test2.img
```

이와 같이 디스크 이미지로 1GiB 공간을 할당하고, block device와 동일한 방식으로 설정을 할 수 있다.
예를 들어 fdisk로 partition을 할당할 수도 있다.
```sh
$ fdisk test.img

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-2097151, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-2097151, default 2097151):

Created a new partition 1 of type 'Linux' and of size 1023 MiB.

Command (m for help): p
Disk test.img: 1 GiB, 1073741824 bytes, 2097152 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x37672bca

Device     Boot Start     End Sectors  Size Id Type
test.img1        2048 2097151 2095104 1023M 83 Linux

Command (m for help): w
The partition table has been altered.
Syncing disks.
```

## MBR(Master Boot Record)

디스크 파티션을 이해하기 위하여는 마스터 부트 레코드를 이해하여야 하는데, 여기에서는 필요한 부분만 간단하게 설명한다.

마스터 부트 레코드는 디스크의 처음 실행을 위한 코드와 파티션 분할 정보가 들어가 있는 디스크의 첫번째 sector라고 할 수 있다. 하드디스크의 섹터 크기는 512bytes이고, setctor라는 개념은 하드디스크에서 읽고 쓰기를 하는 단위로 최근의 SD card 나 MMC에서도 대부분 논리적인 개념으로 512bytes sector 단위를 사용한다 (내부적으로 write/erase block size는 이보다 더 크지만).

MBR에는 앞부분에 일반 PC에서 사용하던 것처럼 작은 실행 코드가 들어 갈 수 있고(PC에서만 사용하는 것으로 임베디드에서는 의미 없음), offset 0x1be 부터 16bytes씩 4개의 partiton 정보가 들어간다. 이 4개가 보통 fdisk에서 보이는 primary partition이다. 다양한 변이들이 생기면서 [여러 버전](https://en.wikipedia.org/wiki/Master_boot_record#Sector_layout)이 있지만 최소한 primary partition 위치는 동일하다. 자세한 사항은 [MBR/EBR Partition Tables](http://thestarman.pcministry.com/asm/mbr/PartTables.htm) 등을 참고 할 수 있다 (MSDOS partition이 아닌 [GPT(GUID Partition Table)](https://en.wikipedia.org/wiki/GUID_Partition_Table)등을 사용하는 경우는 여기서 설명하지 않는다).

SD card나 MMC의 경우 MSDOS 파티션을 사용하는 경우 마찬가지로 512bytes 단위의 섹터를 사용하고 위치를 표시하기 위하여 32bits 크기로 기록된다. 즉, MBR 섹터는 0번, 그 다음 섹터는 1번 식으로 최대 2 tera bytes까지 할당 가능(2^32 x 512bytes)하다.

위의 예를 든 fdisk 파티션은 2,048 섹터에서 시작해서 2,095,151 섹터까지 총 2,095,104 섹터 크기이다. 크기는 2,095,104 x 512 = 1023MiB (1,072,693,248)가 된다.
```
Device     Boot Start     End Sectors  Size Id Type
test.img1        2048 2097151 2095104 1023M 83 Linux
```

dd를 이용하여 직접 파티션에 이미지를 쓸때 이 offset 정보가 필요하다.

## 디스크 이미지 샘플

이제 실제로 디스크 이미지를 만들어 보기로 한다.

dd를 이용하여 1GiB 공간의 디스크 이미지를 만든다.
```sh
$ dd if=/dev/null of=test.img bs=1kx1k seek=1k

$ ls -l
-rw-rw-r--  1 yslee yslee 1073741824 10월 25 17:48 test.img
```

parted의 script를 이용하여 각각 200MiB, 나머지 전체 공간의 두 파티션을 생성한다.
```sh
$ parted -s test.img mklabel msdos
$ parted -s test.img mkpart primary 2048s 200MiB
$ parted -s -- test.img mkpart primary 200MiB -1s
$ parted -s test.img print
Model:  (file)
Disk /home/yslee/tmp/test.img: 1074MB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size   Type     File system  Flags
 1      1049kB  210MB   209MB  primary
 2      210MB   1074MB  864MB  primary
```

- line 1: MSDOS 파티션을 설정
- line 2: 시작은 2048 sector, end를 200MiB로 파티션 할당 (리눅스 파티션)
- line 3: 시작은 200MiB 위치, 끝까지(-1s) 할당. 중간에 double dsah(--)는 -1s 를 option 으로 처리되지 않도록 하기
- line4: 파티션 정보 프린트

디스크 이미지에 파티션 정보를 만들고, 여기에 ext4 디스크 이미지를 만들어서 dd로 복사해 보기로 한다. 디스크 이미지는 [genext2fs](http://genext2fs.sourceforge.net)와 [tune2fs](https://linux.die.net/man/8/tune2fs)로 파일로 부터 직접 이미지를 생성할 수 있으나, 여기에서는 간단하게 loop 디바이스로 마운트 하여 이미지를 생성하도록 한다.

```sh
$ dd if=/dev/null of=test.ext4 bs=1kx1k seek=1
$ mkfs.ext4 test.ext4
$ mkdir test
$ sudo mount -t ext4 test.ext4 test
$ sudo sh -c 'echo "Hello, World!" > test/test.txt'
$ sudo umount test
```

- line 1: test.ext4 파일로 1MiB 이미지 생성
- line 2: 이미지를 ext4 형식으로 초기화
- line 3: test 디렉토리 만들기
- line 4: 이미지를 test 디렉토리에 마운트
- line 5: test.txt를 생성해 넣기
- line 6: 언마운트

첫번째 파티션의 시작 섹터 2,048에 ext4 이미지를 쓴다.

```sh
dd if=test.ext4 of=test.img bs=512 seek=2048 conv=notrunc
```
- if: 입력은 text.ext4 이미지
- of: 쓰는 곳은 test.img
- bs: 블럭 크기를 sector 크기로 설정
- seek: 2048s 에 쓰기
- conv=notrunc: 파일 형식의 이미지에 유효한 것으로, open()함수의 O_TRUNC flag를 끄기. 만일 이 옵션을 주지 않으면 결과 파일인 test.img는 모두 삭제된 후(O_TRUNC)에 입력이 써지게 된다. notrunc 옵션을 주면 overwrite이다. 일반 block device에서는 이렇게 크기가 줄어들 일이 없어 사용치 않는다.

이제 만들어진 이미지를 SD card에 dd로 write를 한 후에 확인을 해보아도 되나 loop device로 마운트 하여 확인할 수 있다.

```sh
$ sudo losetup -Pf --show test.img
/dev/loop0

$ ls -l /dev/loop0*
brw-rw---- 1 root disk   7, 0 10월 25 18:32 /dev/loop0
brw-rw---- 1 root disk 259, 0 10월 25 18:32 /dev/loop0p1
brw-rw---- 1 root disk 259, 1 10월 25 18:32 /dev/loop0p2
```

위와 같이 하면 loop 디바이스를 이용하여 이미지를 attach 할 수 있다. 즉, ls 를 해보면 위처럼 loop0으로 시작하는 디바이스가 생성된다. loop0은 이미지, loop0p1은 첫번째 파티션, loop0p2는 두번째 파티션이다.

첫번째 파티션을 마운트 해서 확인해 본다.
```sh
$ sudo mount -t ext4 /dev/loop0p1 test
$ ls test/
lost+found  test.txt
$ cat test/test.txt
Hello, World!
```

아까 test.img의 첫번째 파티션에 쓴 파일이 정상적으로 보여진다.

이런 식으로 해서 직접 이미지를 쓰는 것도 가능하다. 아래는 두번째 파티션을 초기화 하고 쓰는 예이다.
```sh
$ sudo mkfs.ext4 /dev/loop0p2
$ mkdir test2
$ sudo mount -t ext4 /dev/loop0p2 test2
$ sudo sh -c 'echo "Test2" > test2/test2.txt'
$ ls -l test2/test2.txt
-rw-r--r-- 1 root root 6 10월 25 18:39 test2/test2.txt
```

- line 1: 두번째 파티션을 mkfs.ext4를 이용하여 포멧
- line 2: 시험용 test2 디렉토리 만들기
- line 3: test2에 두번째 파티션 마운트
- line 4: 파일 쓰기

최종적으로 마운트를 종료한다.
```sh
$ sudo umount test2
$ sudo umount test
$ sudo losetup -D
```

- line 3: 이전에 losetup으로 attach 한 것을 모두 detach 한다.

이 과정을 거치게 되면 최종적으로 test.img에는 두개의 파티션이 각각 ext4로 초기화 되고, 첫번째 파티션에는 "Hello, World!" 내용의 test.txt가 두번째 파티션에는 "Test2" 내용의 test2.txt 파일이 있다.

## 정리

디스크 이미지를 만드는 것이 복잡해 보이나, 실제 위의 예를 보면 일반 디스크를 포멧, 마운트 하는 과정과 큰 차이가 없다는 것을 알 수 있다.

이 정도를 알면 디스크 이미지를 생성하거나 수정하는 것을 마음대로 할 수 있을 것이다.

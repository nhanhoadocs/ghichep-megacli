# MegaCLI
- CanhDX (NhanHoa Cloud RD Team)
- 2019-08-12

# MegaCLI sử dụng để convert ổ về Non-RAID 

- Server Model DELL R730 
- Raid Card H730 mini  
- OS CentOS7
- MODE: RAID 
- Disk: 
    + Slot0,1:  2 x 240GB RAID 1 Cài đặt OS 
    + Slot2,3:  2 x 1TB `Non-Raid` để kiểm tra `lsblk`
    + Slot4,5,6: 3 x 1TB `Ready` để test sử dụng MegaCLI cấu hình
- Kiểm tra trên iDRAC 

![](http://i.imgur.com/9W97BWD.png)

# Cài đặt trên CentOS 

Cài đặt MegaCLI
```sh 
wget https://github.com/nhanhoadocs/ghichep-megacli/blob/master/MegaCli8.07.14/Linux/MegaCli-8.07.14-1.noarch.rpm
rpm -ivh MegaCli-8.07.14-1.noarch.rpm 
echo "alias megacli='/opt/MegaRAID/MegaCli/MegaCli64'" >> /root/.bashrc
source /root/.bashrc
```

MegaCLI không cung cấp đầy đủ thông tin chúng ta cần như mapping device, các mức độ RAID cho nên chúng ta sẽ sử dụng thêm một vài công cụ bổ sung 
```sh 
yum install sg3_utils  -y
```

# Cài đặt trên Ubuntu

Cách 1: Cài đặt trực tiếp
```sh
sudo apt-get install wget sg3_utils  -y
wget https://github.com/nhanhoadocs/ghichep-megacli/blob/master/MegaCli8.07.14/Linux/megacli_8.07.14-1_all.deb
sudo dpkg -i megacli_8.07.14-1_all.deb
```

Tạo alias 
```sh 
echo "alias megacli='/opt/MegaRAID/MegaCli/MegaCli64'" >> /root/.bashrc
source /root/.bashrc
```


Cách 2: Chúng ta sẽ convert file từ `.rpm` qua `.deb`

Cài đặt `alien`
```sh 
sudo apt-get install alien dpkg-dev debhelper build-essential unzip -y 
```

Download 
```sh 
wget https://docs.broadcom.com/docs-and-downloads/raid-controllers/raid-controllers-common-files/8-07-14_MegaCLI.zip
unzip 8-07-14_MegaCLI.zip
```

Tạo file Debian từ `.rpm`
```sh 
cd Linux
sudo alien -k --scripts MegaCli-8.07.14-1.noarch.rpm
```

Cài đặt
```sh
sudo dpkg -i megacli_8.07.14-1_all.deb
```

## Các lệnh thao tác cơ bản với MegaCLI

Hiển thị thông tin chi tiết của SAS/SATA
```sh 
megacli -AdpAllInfo -aALL
megacli -CfgDsply -aALL
megacli -AdpEventLog -GetEvents -f events.log -aALL && cat events.log
```

Hiển thị thông tin ngắn gọn 
```sh 
megacli -EncInfo -aALL
```

Thông tin về các ổ đã được RAID (Virtual Devices)
```sh 
megacli -LDInfo -Lall -aALL
```

Thông tin về ổ đĩa vật lý trên Server (Physical Devices)
```sh 
megacli -PDList -aALL
megacli -PDList -a0
```

Thông tin về Pin của card RAID (Batery Backup Unit - BBU)
```sh 
megacli -AdpBbuCmd -aALL
```

## Các câu lệnh quản trị 

Báo động trong im lặng
```sh
megacli -AdpSetProp AlarmSilence -aALL
```

Vô hiệu hóa báo động 
```sh 
megacli -AdpSetProp AlarmDsbl -aALL
```

Bật lai báo động 
```sh 
megacli -AdpSetProp AlarmEnbl -aALL
```

Để xem thông tin về trạng thái đọc tuần tự và độ trễ giữa các lần đọc tuần tự
```sh 
megacli -AdpPR -Info -aALL
```

Thông tin về tốc độ đọc tuần tự 
```sh 
megacli -AdpGetProp PatrolReadRate -aALL
```

Để giảm mức sử dụng tài nguyên tuần tự xuống 2% để giảm thiểu tác động hiệu suất
```sh 
megacli -AdpSetProp PatrolReadRate 2 -aALL
```

Vô hiệu quá đọc tuần tự 
```sh 
megacli -AdpPR -Dsbl -aALL
```

Để bắt đầu một quá trình quét đọc tuần tự một cách thủ công
```sh 
megacli -AdpPR -Start -aALL
```

Để ngăn chặn quá trình quét đọc tuần tự
```sh 
megacli -AdpPR -Stop -aALL
```

Tắt bộ đệm vật lý tránh mất dữ liệu trong quá trình không có UPS dự phòng điện 
```sh 
megacli -LDGetProp EnDskCache -LAll -aALL
```

Enable bộ đệm 
```sh 
megacli -LDGetProp DisDskCache -LAll -aALL
```

Chi tiết các disk 
```sh 
megacli -ShowSummary -aALL
```

Kiểm tra cảnh báo đọc tuần tự 
```sh 
megacli -AdpEventLog -GetSinceReboot -warning -fatal -a0 
```

Kiểm tra RAID hiện có là 0,1,10... 
```sh 
megacli -LDInfo -L0 -a0 |grep -i raid
```

# Chuyển đổi disk từ RAID về Non-Raid dùng MegaCLI

## Bước 0: Kiểm tra các ổ đang nhận trên hệ thống dùng `lsblk`
```sh 
lsblk
```

Kết quả:
```sh 
[root@ceph03 ~]# lsblk 
NAME                   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda                      8:0    0 894,3G  0 disk 
├─sda1                   8:1    0    16M  0 part 
└─sda2                   8:2    0 894,2G  0 part 
sdb                      8:16   0 894,3G  0 disk 
├─sdb1                   8:17   0    16M  0 part 
└─sdb2                   8:18   0 894,2G  0 part 
sdc                      8:32   0   223G  0 disk 
├─sdc1                   8:33   0     1G  0 part /boot
├─sdc2                   8:34   0     8G  0 part [SWAP]
└─sdc3                   8:35   0   214G  0 part 
  └─centos_ceph03-root 253:0    0   214G  0 lvm  /
```

## Bước 1: List toàn bộ các Physical Device trên hệ thống
```sh 
megacli -PDList -aALL
```

Kết quả 
```sh 
Adapter #0

Enclosure Device ID: 32
Slot Number: 0
Drive's position: DiskGroup: 0, Span: 0, Arm: 0
Enclosure position: 1
Device Id: 0
WWN: 5002538e0006566d
Sequence Number: 2
Media Error Count: 0
Other Error Count: 0
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SATA

Raw Size: 223.570 GB [0x1bf244b0 Sectors]
Non Coerced Size: 223.070 GB [0x1be244b0 Sectors]
Coerced Size: 223.0 GB [0x1be00000 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  4096
Firmware state: Online, Spun Up
Device Firmware Level: 304Q
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x500056b3c64865c0
Connected Port Number: 0(path0) 
Inquiry Data: S45RNE0M218172      SAMSUNG MZ7LH240HAHQ-00005              HXT7304Q
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None 
Device Speed: 6.0Gb/s 
Link Speed: 6.0Gb/s 
Media Type: Solid State Device
Drive:  Not Certified
Drive Temperature :38C (100.40 F)
PI Eligibility:  No 
Drive is formatted for PI information:  No
PI: No PI
Drive's NCQ setting : N/A
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s 
Drive has flagged a S.M.A.R.T alert : No



Enclosure Device ID: 32
Slot Number: 1
Drive's position: DiskGroup: 0, Span: 0, Arm: 1
Enclosure position: 1
Device Id: 1
WWN: 5002538e00065665
Sequence Number: 2
Media Error Count: 0
Other Error Count: 0
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SATA

Raw Size: 223.570 GB [0x1bf244b0 Sectors]
Non Coerced Size: 223.070 GB [0x1be244b0 Sectors]
Coerced Size: 223.0 GB [0x1be00000 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  4096
Firmware state: Online, Spun Up
Device Firmware Level: 304Q
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x500056b3c64865c1
Connected Port Number: 0(path0) 
Inquiry Data: S45RNE0M218164      SAMSUNG MZ7LH240HAHQ-00005              HXT7304Q
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None 
Device Speed: 6.0Gb/s 
Link Speed: 6.0Gb/s 
Media Type: Solid State Device
Drive:  Not Certified
Drive Temperature :38C (100.40 F)
PI Eligibility:  No 
Drive is formatted for PI information:  No
PI: No PI
Drive's NCQ setting : N/A
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s 
Drive has flagged a S.M.A.R.T alert : No



Enclosure Device ID: 32
Slot Number: 2
Enclosure position: 1
Device Id: 2
WWN: 5002538e40f935da
Sequence Number: 2
Media Error Count: 0
Other Error Count: 2
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SATA

Raw Size: 894.252 GB [0x6fc81ab0 Sectors]
Non Coerced Size: 893.752 GB [0x6fb81ab0 Sectors]
Coerced Size: 893.75 GB [0x6fb80000 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  4096
Firmware state: JBOD
Device Firmware Level: 304Q
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x500056b3c64865c2
Connected Port Number: 0(path0) 
Inquiry Data: S45NNX0M433873      SAMSUNG MZ7LH960HAJR-00005              HXT7304Q
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None 
Device Speed: 6.0Gb/s 
Link Speed: 6.0Gb/s 
Media Type: Solid State Device
Drive:  Not Certified
Drive Temperature :38C (100.40 F)
PI Eligibility:  No 
Drive is formatted for PI information:  No
PI: No PI
Drive's NCQ setting : N/A
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s 
Drive has flagged a S.M.A.R.T alert : No



Enclosure Device ID: 32
Slot Number: 3
Enclosure position: 1
Device Id: 3
WWN: 5002538e40f93120
Sequence Number: 2
Media Error Count: 0
Other Error Count: 2
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SATA

Raw Size: 894.252 GB [0x6fc81ab0 Sectors]
Non Coerced Size: 893.752 GB [0x6fb81ab0 Sectors]
Coerced Size: 893.75 GB [0x6fb80000 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  4096
Firmware state: JBOD
Device Firmware Level: 304Q
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x500056b3c64865c3
Connected Port Number: 0(path0) 
Inquiry Data: S45NNX0M433643      SAMSUNG MZ7LH960HAJR-00005              HXT7304Q
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None 
Device Speed: 6.0Gb/s 
Link Speed: 6.0Gb/s 
Media Type: Solid State Device
Drive:  Not Certified
Drive Temperature :38C (100.40 F)
PI Eligibility:  No 
Drive is formatted for PI information:  No
PI: No PI
Drive's NCQ setting : N/A
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s 
Drive has flagged a S.M.A.R.T alert : No



Enclosure Device ID: 32
Slot Number: 4
Enclosure position: 1
Device Id: 4
WWN: 5002538e40f9359b
Sequence Number: 1
Media Error Count: 0
Other Error Count: 0
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SATA

Raw Size: 894.252 GB [0x6fc81ab0 Sectors]
Non Coerced Size: 893.752 GB [0x6fb81ab0 Sectors]
Coerced Size: 893.75 GB [0x6fb80000 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  4096
Firmware state: Unconfigured(good), Spun Up
Device Firmware Level: 304Q
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x500056b3c64865c4
Connected Port Number: 0(path0) 
Inquiry Data: S45NNX0M433855      SAMSUNG MZ7LH960HAJR-00005              HXT7304Q
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None 
Device Speed: 6.0Gb/s 
Link Speed: 6.0Gb/s 
Media Type: Solid State Device
Drive:  Not Certified
Drive Temperature :38C (100.40 F)
PI Eligibility:  No 
Drive is formatted for PI information:  No
PI: No PI
Drive's NCQ setting : N/A
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s 
Drive has flagged a S.M.A.R.T alert : No



Enclosure Device ID: 32
Slot Number: 5
Enclosure position: 1
Device Id: 5
WWN: 5002538e000cea93
Sequence Number: 1
Media Error Count: 0
Other Error Count: 0
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SATA

Raw Size: 894.252 GB [0x6fc81ab0 Sectors]
Non Coerced Size: 893.752 GB [0x6fb81ab0 Sectors]
Coerced Size: 893.75 GB [0x6fb80000 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  4096
Firmware state: Unconfigured(good), Spun Up
Device Firmware Level: 304Q
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x500056b3c64865c5
Connected Port Number: 0(path0) 
Inquiry Data: S45NNE0M308812      SAMSUNG MZ7LH960HAJR-00005              HXT7304Q
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None 
Device Speed: 6.0Gb/s 
Link Speed: 6.0Gb/s 
Media Type: Solid State Device
Drive:  Not Certified
Drive Temperature :38C (100.40 F)
PI Eligibility:  No 
Drive is formatted for PI information:  No
PI: No PI
Drive's NCQ setting : N/A
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s 
Drive has flagged a S.M.A.R.T alert : No



Enclosure Device ID: 32
Slot Number: 6
Enclosure position: 1
Device Id: 6
WWN: 5002538e40f93599
Sequence Number: 1
Media Error Count: 0
Other Error Count: 0
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SATA

Raw Size: 894.252 GB [0x6fc81ab0 Sectors]
Non Coerced Size: 893.752 GB [0x6fb81ab0 Sectors]
Coerced Size: 893.75 GB [0x6fb80000 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  4096
Firmware state: Unconfigured(good), Spun Up
Device Firmware Level: 304Q
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x500056b3c64865c6
Connected Port Number: 0(path0) 
Inquiry Data: S45NNX0M433853      SAMSUNG MZ7LH960HAJR-00005              HXT7304Q
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None 
Device Speed: 6.0Gb/s 
Link Speed: 6.0Gb/s 
Media Type: Solid State Device
Drive:  Not Certified
Drive Temperature :37C (98.60 F)
PI Eligibility:  No 
Drive is formatted for PI information:  No
PI: No PI
Drive's NCQ setting : N/A
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s 
Drive has flagged a S.M.A.R.T alert : No




Exit Code: 0x00
```

## Bước 2: Show ra Virtual Device (RAID1)
```sh 
megacli LDInfo -Lall -a0
```
Việc list ra ổ cài OS để kiểm tra và loại trừ các ổ đĩa vật lý cho Virtual Device này. Chúng ta sẽ ko sử dụng các ổ OS này để chuyển sang Non-RAID (JBOD) vì nó có thể xóa mất OS trên Server vì thế bước này chúng ta cần thao tác cẩn thận.

Kết quả: 
```sh 
[root@ceph03 ~]# megacli LDInfo -Lall -a0
                                     

Adapter 0 -- Virtual Drive Information:
Virtual Drive: 0 (Target Id: 0)
Name                :riad1os
RAID Level          : Primary-1, Secondary-0, RAID Level Qualifier-0
Size                : 223.0 GB
Sector Size         : 512
Is VD emulated      : Yes
Mirror Data         : 223.0 GB
State               : Optimal
Strip Size          : 64 KB
Number Of Drives    : 2
Span Depth          : 1
Default Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
Current Cache Policy: WriteBack, ReadAhead, Direct, No Write Cache if Bad BBU
Default Access Policy: Read/Write
Current Access Policy: Read/Write
Disk Cache Policy   : Disk's Default
Encryption Type     : None
Default Power Savings Policy: Controller Defined
Current Power Savings Policy: None
Can spin up in 1 minute: No
LD has drives that support T10 power conditions: No
LD's IO profile supports MAX power savings with cached writes: No
Bad Blocks Exist: No
PI type: No PI

Is VD Cached: No



Exit Code: 0x00
```

## Bước 3: Kiểm tra  Enclosure Device ID (ID Panel SAS gắn disk) bình thường sẽ giống nhau 
```sh 
megacli -PDList -a0 | grep -e '^Enclosure Device ID:' | head -1 | cut -f2- -d':' | xargs
```

Kết quả 
```sh 
32
```

Enclosure Device ID: 32

## Bước 4: Tìm kiếm Slot number (Khe cắm) cuẩ các disk muốn chuyển qua JBOD

Xác định các ổ đĩa nằm trên khe cắm nào chúng ta sẽ chuyển qua JBOD. Cách đơn giản nhất là xem lại danh sách `Physical Devie` ở bước 1. Loại trừ các Slot 0,1 RAID 0 cài OS, và Slot2,3 đã cấu hình Non-Raid trong Raid Manager. ID bao gồm 4,5,6

## Bước 5: Set các disk cần chuyển qua Non-RAID (JBOD) là `GOOD`

Điều này đánh dấu chúng là chưa được định cấu hình nhưng có xuất hiện
```sh 
megacli -PDMakeGood -PhysDrv[32:4, 32:5, 32:6] -Force -a0
```
> lưu ý các số 4 - 6 là số khe của các đĩa cần chuyển. 32 là `Enclosure Device ID` bước 3

Kết quả: 
```sh 
Adapter: 0: EnclId-32 SlotId-4 state changed to Unconfigured-Good.
Adapter: 0: EnclId-32 SlotId-5 state changed to Unconfigured-Good.
Adapter: 0: EnclId-32 SlotId-6 state changed to Unconfigured-Good.

Exit Code: 0x00
```

Nếu có lỗi xảy ra thường là ổ đã được MakeGood có thể thử thao tác lại một lần nữa sau khi MakeGood thành công kết quả sẽ như sau
```sh 
Adapter: 0: Failed to change PD state at EnclId-32 SlotId-4.

Exit Code: 0x01
```

## Bước 6: Kiểm tra chế độ JBOD được enable 
```sh 
megacli AdpGetProp EnableJBOD -aALL
```

Kết quả 
```sh 
Adapter 0: JBOD: Enabled

Exit Code: 0x00
```

## Bước 7: Enable JBOD nếu chưa được enable 
```sh 
megacli AdpSetProp EnableJBOD 1 -a0
```

## Bước 8: Set các disk trên về JBOD 
```sh 
megacli -PDMakeJBOD -PhysDrv[32:4, 32:5, 32:6] -a0
```

Kết quả: 
```sh 
Adapter: 0: EnclId-32 SlotId-4 state changed to JBOD.
Adapter: 0: EnclId-32 SlotId-5 state changed to JBOD.
Adapter: 0: EnclId-32 SlotId-6 state changed to JBOD.

Exit Code: 0x00
```

Nếu có lỗi xảy ra thường là ổ đã được chuyển thành JBOD thành công có thể thử thao tác lại một lần nữa sau khi chuyển thành công kết quả sẽ như sau
```sh 
Adapter: 0: Failed to change PD state at EnclId-32 SlotId-4.

Exit Code: 0x01
```

## Bước 9: Kiểm tra các disk bằng `lsblk`
```sh 
lsblk
```

Kết quả:
```sh 
[root@ceph03 ~]# lsblk 
NAME                   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda                      8:0    0 894,3G  0 disk 
├─sda1                   8:1    0    16M  0 part 
└─sda2                   8:2    0 894,2G  0 part 
sdb                      8:16   0 894,3G  0 disk 
├─sdb1                   8:17   0    16M  0 part 
└─sdb2                   8:18   0 894,2G  0 part 
sdc                      8:32   0   223G  0 disk 
├─sdc1                   8:33   0     1G  0 part /boot
├─sdc2                   8:34   0     8G  0 part [SWAP]
└─sdc3                   8:35   0   214G  0 part 
  └─centos_ceph03-root 253:0    0   214G  0 lvm  /
sdd                      8:48   0 894,3G  0 disk 
├─sdd1                   8:49   0    16M  0 part 
└─sdd2                   8:50   0 894,2G  0 part 
sde                      8:64   0 894,3G  0 disk 
├─sde1                   8:65   0    16M  0 part 
└─sde2                   8:66   0 894,2G  0 part 
sdf                      8:80   0 894,3G  0 disk 
├─sdf1                   8:81   0    16M  0 part 
└─sdf2                   8:82   0 894,2G  0 part 
```

Kiểm tra trên iDRAC 

![](http://i.imgur.com/nU7EE7u.png)

## Bước 10: Chuyển disk slot 4 từ JBOD về disk chưa sử dụng 

> ### Lưu ý bước này sẽ convert từ Non-RAID về chưa sửa dụng (Có thể cấu hình RAID) sẽ dẫn đến mất toàn bộ dữ liệu trên disk 

Sử dụng `MakeGood` để chuyển 
```sh 
megacli -PDMakeGood -PhysDrv[32:4] -Force -a0
```

Kết quả:
```sh 
Adapter: 0: EnclId-32 SlotId-4 state changed to Unconfigured-Good.

Exit Code: 0x00
```

Kiểm tra trên iDRAC 

![](http://i.imgur.com/xq64rEc.png)

# Create RAID 

## 1. Command syntax
```sh
MegaCli -CfgLdAdd -rX [E0:S0,E1:S1,...] [WT|WB] [NORA|RA] [Direct|Cached]
        [CachedBadBBU|NoCachedBadBBU] [-szXXX [-szYYY ...]]
        [-strpszM] [-Hsp[E0:S0,...]] [-AfterLdX] | [FDE|CtrlBased]  
        [-Default| -Automatic| -None| -Maximum| -MaximumWithoutCaching] 
        [-Cache] [-enblPI] [-Force]-aN 
```
## 2.Get adapter ID (it's 0 in the example below), Enclosure ID, Slot ID:
```sh
#MegaCli64 -PDList -aALL | egrep 'Adapter|Enclosure|Slot'
Adapter #0
Enclosure Device ID: 252
Slot Number: 0
Enclosure position: N/A
Enclosure Device ID: 252
Slot Number: 1
Enclosure position: N/A
Enclosure Device ID: 252
Slot Number: 2
Enclosure position: N/A
Enclosure Device ID: 252
Slot Number: 3
Enclosure position: N/A
Enclosure Device ID: 252
Slot Number: 4
Enclosure position: N/A
Enclosure Device ID: 252
Slot Number: 5
Enclosure position: N/A
Enclosure Device ID: 252
Slot Number: 6
Enclosure position: N/A
Enclosure Device ID: 252
Slot Number: 7
Enclosure position: N/A
Enclosure Device ID: 252
Slot Number: 8
Enclosure position: N/A
Enclosure Device ID: 252
Slot Number: 9
Enclosure position: N/A

In above example, we get

Atapter ID    --> 0
Enclosure ID  --> 252
Slot Number   --> 0,1,2,3,4,5,6,7,8,9
```

## 3. Double check if drive are being used for Logical Drive

This command lists all Logical Drives
```sh
MegaCli64 -LDInfo -Lall -aALL
```

This command lists all Logical and its Physical drive info
```sh
MegaCli64 -LdPdInfo -aALL
```

## 4. Create a RAID0,1,5,6 array with MegaCli

Use the Adapter ID, Enclosure ID, and Slot number that found in step 1. Suppose disk slot 2,3,4,5,6,7,8,9 are not being used.
```sh
MegaCli64 -CfgLdAdd -r0 [252:2,252:3] -a0
MegaCli64 -CfgLdAdd -r1 [252:2,252:3] -a0
MegaCli64 -CfgLdAdd -r5 [252:2,252:3,252:4,252:5] -a0
MegaCli64 -CfgLdAdd -r6 [252:2,252:3,252:4,252:5,252:6] -a0

Note:    to make RAID0, minimal 2 drives are required.
         to make RAID1, minimal 2 drives are required.
         to make RAID5, minimal 3 drives are required.
         to make RAID6, minimal 4 drives are required.
```

## 5. Create a RAID50,raid60 array with MegaCli

Use the Adapter ID, Enclosure ID, and Slot number that found in step 1. 

Suppose disk slot 2,3,4,5,6,7,8,9 are not being used.

The command is slightly different with the one created raid0.,raid1,raid5, and raid6. Because raid50,60 are netsted raid.
```sh
MegaCli64 -CfgLdAdd -r50 raid0[252:2,252:3,252:4] raid1[252:5,252:6,252:7] -a0
MegaCli64 -CfgLdAdd -r60 raid0[252:2,252:3,252:4,252:5] raid1[252:6,252:7,252:8,252,9] -a0
```
> Note: to make RAID50, minimal 6 drives are required to make RAID60, minimal 8 drives are required.

## 6. Show system summary
```sh
MegaCli64 -ShowSummary -aALL
                             
System
    Operating System:  Linux version 2.6.32-504.23.4.el6.x86_64 
    Driver Version: 06.803.01.00-rh1
    CLI Version: 8.07.14

Hardware
        Controller
                 ProductName       : ServeRAID M5015 SAS/SATA Controller(Bus 0, Dev 0)
                 SAS Address       : 500605b004a170a0
                 FW Package Version: 12.12.0-0098
                 Status            : Optimal
        BBU
                 BBU Type          : iBBU08
                 Status            : Healthy
        Enclosure
                 Product Id        : SGPIO           
                 Type              : SGPIO
                 Status            : OK

        PD 
                Connector          : Port 0 - 3<Internal>: Slot 1 
                Vendor Id          : IBM-ESXS
                Product Id         : ST9300653SS     
                State              : Online
                Disk Type          : SAS,Hard Disk Device
                Capacity           : 278.464 GB
                Power State        : Active
...
```

# Clear ổ Foregin 

![](http://i.imgur.com/uUuEkJT.png)

```sh 
root@cephbka01:~# megacli -PDMakeJBOD -PhysDrv[32:2] -a0
                                     
Adapter: 0: Failed to change PD state at EnclId-32 SlotId-2.

Exit Code: 0x01
root@cephbka01:~# 
```

Scan Foreign disk 
```sh 
root@cephbka01:~# megacli -CfgForeign -Scan -aALL
                                     
There are 1 foreign configuration(s) on controller 0.

Exit Code: 0x00
root@cephbka01:~#
```

```sh 
root@cephbka01:~# megacli -PDList -aALL | grep "Foreign State:"
Foreign State: None 
Foreign State: None 
Foreign State: Foreign 
Foreign State: None 
Foreign State: None 
Foreign State: None 
Foreign State: None 
Foreign State: None 
root@cephbka01:~# 
```

Clean foreign disk 
```sh 
root@cephbka01:~# megacli -CfgForeign -Clear -a0
                                     
Foreign configuration 0 is cleared on controller 0.

Exit Code: 0x00
root@cephbka01:~# megacli -PDList -aALL | grep "Foreign State:"
Foreign State: None 
Foreign State: None 
Foreign State: None 
Foreign State: None 
Foreign State: None 
Foreign State: None 
Foreign State: None 
Foreign State: None 
root@cephbka01:~# 
```

Make JBOD 
```sh 
root@cephbka01:~# megacli -PDMakeJBOD -PhysDrv[32:2] -a0
                                     
Adapter: 0: EnclId-32 SlotId-2 state changed to JBOD.

Exit Code: 0x00
root@cephbka01:~# lsblk 
NAME                    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda                       8:0    0 446.6G  0 disk 
├─sda1                    8:1    0   476M  0 part /boot
├─sda2                    8:2    0   7.5G  0 part [SWAP]
└─sda3                    8:3    0 438.7G  0 part /
sdb                       8:16   0   7.3T  0 disk 
└─vg_backup-vol_backups 252:0    0  36.4T  0 lvm  /backy2_backup
sdc                       8:32   0   7.3T  0 disk 
└─vg_backup-vol_backups 252:0    0  36.4T  0 lvm  /backy2_backup
sdd                       8:48   0   7.3T  0 disk 
└─vg_backup-vol_backups 252:0    0  36.4T  0 lvm  /backy2_backup
sde                       8:64   0   7.3T  0 disk 
└─vg_backup-vol_backups 252:0    0  36.4T  0 lvm  /backy2_backup
sdf                       8:80   0   7.3T  0 disk 
└─vg_backup-vol_backups 252:0    0  36.4T  0 lvm  /backy2_backup
sdg                       8:96   0   7.3T  0 disk 
root@cephbka01:~# 
```


# Tài liệu tham khảo 

[Non-RAID Architecture](https://en.wikipedia.org/wiki/Non-RAID_drive_architectures)

[Thao tác cơ bản](https://www.24x7serversupport.com/24x7serversupport-blog/how-to-install-megacli-on-centos7/)

[JBOD](https://effndc.wordpress.com/2014/07/16/switching-lsi-sas-2208-and-similar-chipsets-to-jbod-mode/)

[JBOD2](https://plone.lucidsolutions.co.nz/hardware/sas-controller/lsi-2208/configuring-jbod-with-a-lsi-2208-controller/view)

[RAID](https://danielparker.me/til/megacli/megaraid/mega-cli-TIL/)

[MegaCLI Cheatsheet](https://www.broadcom.com/support/knowledgebase/1211161498596/megacli-cheat-sheet--live-examples)

[Hardware_Raid_Setup_using_MegaCli](https://raid.wiki.kernel.org/index.php/Hardware_Raid_Setup_using_MegaCli)

[MegaCLI create Raid](http://fibrevillage.com/storage/374-megacli-raid-0-1-5-6-and-raid50-raid60-creation-examples)

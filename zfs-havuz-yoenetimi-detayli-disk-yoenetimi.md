---
description: Detaylı ZFS Havuz Yönetimi Komutları
---


# ZFS Havuz Yönetimi - Detaylı Havuz Yönetimi


Bu kısıma kadar ufak bir giriş yaptığımız ZFS disk yönetim sisteminin daha öncesinde bahsettiğimiz özelliklerine değineceğiz. Bunu yaparken bir önceki bölümde kısaca değindiğimiz aynalanmış disk sistemleri oluşturmayı, RAIDZ diskleri oluşturma ve bu disklerin özelliklerini açıklayacağım ve gerçek zamanlı bazı örneklerden bahsedeceğim.

## ZFS Disk Tipleri

Bir önceki bölümde bahsettiğim gibi temel olarak 5 bölüm bulunmakta.  Bunlar **disk \(varsayılan\):**, **dosya \(file\)**, **ayna \(mirror\)**, **raidz1/2/3**, **yedek \(spare\)**, **önbellek \(cache\)**, **günlük \(log\)**dir. Şimdi tek tek bunların özelliklerinden bahsedelim ve havuza bunları eklemeyi görelim.

### Dosya Türünde Diskleri Oluşturma ve Depolama Havuzuna Ekleme 
Belirtildiği gibi, önceden tahsis edilmiş dosyalar, mevcut dosya sisteminizde (ext4, xfs veya her neyse) `zpool`'ları kurmak için kullanılabilir. Bunun, üretim verilerini depolamak için değil, tamamen üretim ve test amaçlı olduğu unutulmamalıdır. 
Dosyaları kullanmak, sıkıştırma oranını, veri tekilleştirme tablosunun boyutunu veya diğer şeyleri gerçekten üretim verilerini taahhüt etmeden test edebileceğiniz bir sanal alana sahip olmanın harika bir yoludur.

Dosya VDEV'leri oluştururken, göreceli yollar kullanamazsınız, ancak mutlak yollar kullanmanız gerekir. Yani `/tmp/dosya` şeklinde kök dizininden başlayacak şeklinde yolunu belirtmeniz gerekmekte. Ayrıca, görüntü dosyalarına önceden bir boyut tahsis edilmiş olması veya bu boyutun seyrek veya kısmi olarak sağlanması gerekir. Bunun nasıl çalıştığını görelim. 

İlk olarak 1 GB'lık alan kaplayacak 4 adet dosya oluşturalım:

```
~# for i in {1..4}
do
     dd if=/dev/zero of=/tmp/file$i bs=1G count=4 &> /dev/null
done
```
Şimdi bu dosyaları kullanarak bir havuz oluşturalım.

```
~# zpool create filetank /tmp/file1 /tmp/file2 /tmp/file3 /tmp/file4
```

```
~# zpool status tank
pool: tank
state: ONLINE
config:

	NAME          STATE     READ WRITE CKSUM
	tank          ONLINE       0     0     0
	  /tmp/file1  ONLINE       0     0     0
	  /tmp/file2  ONLINE       0     0     0
	  /tmp/file3  ONLINE       0     0     0
	  /tmp/file4  ONLINE       0     0     0

errors: No known data errors

```

Bu tip havuzları sadece dosyalardan oluşacak şekilde yaratabileceğimiz gibi buna ek olarak farklı disklerle harmanlayarak oluşturabiliriz de. 
```
~# zpool create filetank  /dev/sdb /dev/sdc /tmp/file1 /tmp/file2 /tmp/file3 /tmp/file4
```

```
~# zpool status tank
pool: tank
state: ONLINE
config:

	NAME          STATE     READ WRITE CKSUM
	tank          ONLINE       0     0     0
      sdb         ONLINE       0     0     0
	  sdc         ONLINE       0     0     0
	  /tmp/file1  ONLINE       0     0     0
	  /tmp/file2  ONLINE       0     0     0
	  /tmp/file3  ONLINE       0     0     0
	  /tmp/file4  ONLINE       0     0     0


errors: No known data errors

```

.. Note:: Bu şekilde dosyalardan oluşmuş havuzların export edilmesi durumunda havuzun yeniden bulunması sorun yaşatabilir. Tekrar aynı havuzu import etmek için dosyalardan en az birinin bulunduğu yolu belirtmemiz gerekmekte. Örneğin:

```
~# zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  7.50G   214K  7.50G        -         -     0%     0%  1.00x    ONLINE  -
~# zpool export tank
~# zpool import tank
cannot import 'tank': no such pool available
```

Bu durumda `/tmp/file1`'i belirterek `tank` havuzunu import etmeye çalışınca
```
~# zpool import -d /tmp/file1 tank
~# zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  7.50G   300K  7.50G        -         -     0%     0%  1.00x    ONLINE  -
```

### Yansı Diskleri Oluşturma ve Depolama Havuzuna Ekleme 
### RAID ve RAIDZ Diskleri Oluşturma ve Depolama Havuzuna Ekleme 
### Yedek Diskleri Oluşturma ve Depolama Havuzuna Ekleme 
### Önbellek Diskleri Oluşturma ve Depolama Havuzuna Ekleme 



# ZFS Kopya Kağıdı

## Havuz İşlemleri

### Havuzları Listele

```shell
~# zpool list  
```

### Havuz Oluşturma

Tek bir diskte bir ZFS birimi / havuzu oluşturma:

```shell
~# zpool create vol0 /dev/sd[x]  
```

Not: Havuzunuz otomatik olarak bağlanacaktır /[pool name].

### Havuz Silme

#### Havuzu Sil

```shell
~# zpool destroy [pool name]  
```

#### Bir Havuzdaki Tüm Veri Kümelerini Silme:

```shell
zfs destroy -r [pool name] 
```


### Havuz Kontrolü 

#### Disk Durumlarını Kontrol Etme

Yedekli bir havuz stratejisine sahipseniz, herhangi bir sürücünün arada bir arıza yapıp yapmadığını kontrol etmek isteyebilirsiniz. Bu sadece havuzları kontrol ederek yapılır.

```shell
~# zpool status  
```

#### Havuzun Boş Alanını Kontrol Editme

Zaten veri içeren bir havuza eklerseniz, havuzunuz başlangıçta "dengesiz" olur ve havuza daha fazla veri yazılıncaya kadar dengesiz kalır. Bunun nedeni, ZFS'nin yeni diskleri kullanmak için mevcut verileri yayma zahmetine girmemesidir. Havuza veri eklemeye devam ederseniz, sonunda diskler arasında kullanılan alan açısından dengelenir, ancak mevcut verileriniz havuza yeniden yazmadığınız sürece yalnızca ilk disklere yazılır.

Havuzunuzun kalan alanını kontrol etmek için şunları yapın:

```shell
zpool list -v
```

Aşağıda, yakın zamanda 2 x 8 TB sürücüler eklediğim RAID10 havuzumda bu komutun bazı örnek çıktıları var. Gördüğünüz gibi, dizim oldukça dengesiz ve çok daha iyi bir performans elde etmek istiyorsam diziyi yeniden dengelemem gerekecek.

```shell
 NAME     SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT 
  zpool1  13.6T  4.21T  9.39T         -    15%    30%  1.00x  ONLINE  -
  mirror  3.62T  2.20T  1.43T         -    31%    60%
    sda      -      -      -         -      -      -
    sdb      -      -      -         -      -      -
  mirror  2.72T  1.65T  1.07T         -    32%    60%
    sdc      -      -      -         -      -      -
    sdd      -      -      -         -      -      -
  mirror  7.25T   368G  6.89T         -     2%     4%
    sde      -      -      -         -      -      -
    sdf      -      -      -         -      -      -
```

Bir diziyi yeniden dengelemenin en kolay yolu, muhtemelen yeni bir geçici veri kümesi oluşturmak ve var olan tüm verileri ona taşımak ve sonra tekrar geri getirmektir. İlk hareketin sonunda, diskler oldukça kullanılmalıdır, ancak tek tek dosyalar kullanılmayacaktır ve ikinci geçişte dosyalar da oldukça dengeli olacaktır.

Yalnızca mv komutunu kullanmaya dikkat edin, çünkü başlangıçta, orijinali silmeden önce tümü yazılana kadar verileri kopyalayacaktır ve boşluk alanınız kolayca alanınız tükenecektir. Dosyaları birer birer taşımak için burada gösterildiği gibi rsync gibi bir şey kullanmak daha iyi olacaktır .

### Havuz Sorunlarını Temizleme

#### Bir havuzu fırçalayın

```shell
~# zpool scrub [pool name]
```

Bir fırçalama işleminin ilerlemesini görmek için 

```shell
~# zpool status
```

## Veri kümeleri

### Veri Kümesi Oluşturun

```shell
~# zfs create [pool name]/[dataset name]  
```

ZFS, veri kümesini otomatik olarak / yol / havuz / [veri kümesi adı] konumuna bağlayacaktır.

Aşağıdaki gibi bir "alt" veri kümesi / dosya sistemi oluşturabilirsiniz:

```shell
~# zfs create [pool name]/[dataset name]/[descendent filesystem]
```

### Veri Kümelerini ve Havuzları Listeleme

```shell
~# zfs list  
```

### Veri Kümesini Sil

```shell
~# zfs destroy [pool name]/[dataset name]  
```

Veri kümesinin anlık görüntüleri veya klonları mevcutsa veri kümesi yok edilemez.

### Veri Kümesi Kayıt Boyutunu Ayarlama

Kayıt boyutunun gerçekte ne yaptığı hakkında daha fazla bilgi için burayı okuyun.

```shell
~# zfs set recordsize=[size] pool/dataset/name
```

Boyut, 16k, 128k veya 1M gibi bir değer olmalıdır.

### Veri Kümesi Kayıt Boyutunu Alın

```shell
~# zfs get recordsize pool/dataset/name
```

## ZFS'de Anlık görüntüler

### Anlık Görüntü Veri Kümeleri

```shell
zfs snapshot [pool]/[dataset name]@[snapshot name]  
```

### Anlık Görüntüleri Listele

```shell
~# zfs list -t snapshot  
```

### Anlık Görüntüleri Yeniden Adlandırma

```shell
zfs rename [pool]/[dataset]@[old name] [new name]  
```

### Anlık Görüntüyü Geri Yükle

```shell
zfs rollback -r [pool]/[dataset]@[snapshot name]  
```

Bu, [anlık görüntü adı] alındıktan sonra çekilen tüm anlık görüntüleri silecek !

Geri almak istediğiniz dosya sistemi, şu anda bağlıysa, çıkarılır ve yeniden bağlanır. Dosya sistemi çıkarılamazsa, geri alma başarısız olur. -fGerekirse seçenek kuvvetleri dosya sistemi, sistemden ayrıldı edilecek.

### Anlık Görüntüyü Silme

```shell
zfs destroy tank/home/cindys@snap1  
```
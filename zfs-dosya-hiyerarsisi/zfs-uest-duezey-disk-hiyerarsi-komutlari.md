---
description: ZFS'de üst düzey olarak nitelendirilebilecek hiyerarşi komutları.
---

# ZFS Üst Düzey Disk Hiyerarşi Komutları

## ZFS'ye Has Hiyerarşi Özellikleri

### Hiyerarşi Bazında Sıkıştırma Ayarlamak

Eğer etkinleştirirseniz, ZFS hiyerarşileri, ayrı ayrı sıkıştırmayı destekler ve bu işlemde veriler şeffaf bir şekilde tutulur. Havuzunuzda sakladığınız her dosya sıkıştırılabilir. Kullanıcı ise bu sıkıştırılmış dosyalara hiç sıkıştırılmamışçasına erişilebilir. Başka bir deyişle, geleneksel dosya sistemine bağlandığı andan itibaren hiyerarşiler kullanıcının ek bir işlem yapmasına gerek kalmadan sıkıştırılmasını ve kullanıldığı zaman açılarak kullanılmasını sağlar. ZFS geleneksel sıkıştırma yöntemlerinin aksine, dosya katmanının altında, diskteki veriler anında sıkıştırılır veya açılır. Ve CPU'da sıkıştırma yapmanın maliyeti az olduğu ve bazı algoritmalarda sıkıştırma son derece hızlı olduğu için kullanıcı tarafından çoğunlukla farkedilmemektedir.

Daha öncesinde aktardığım gibi hiyerarşiler içerisindeki veri kümeleri ayrı ayrı sıkıştırma ayarlamasına sahiptir. İstenilene sıkıştırma eklenebilir istenilenden de bu sıkıştırma özelliği devre dışı bırakılabilir. Ayrıca desteklenen sıkıştırma algoritmaları LZJB , LZ4, ZLE ve Gzip'tir. Son dönemde yapılan güncellemelerle ZSTD algoritması da öntanımlı sıkıştırmalar arasına eklenmiştir.

 Gzipte, 9 katmalı sıkıştırma vardır; 1. seviye olabildiğince hızlı, en az sıkıştırmayla ve 9. seviye olabildiğince sıkıştırma yapılarak 1'den 9'a kadar olan standart düzeyleri belirlemeye imkan sağlar. Varsayılan, GNU/Linux ve diğer Unix işletim sistemlerinde standart olduğu gibi 6'dır. 

LZJB ise ZFS'nin de yazarı olan Jeff Bonwick tarafından icat edilen bir sıkıştırma algoritmasıdır. LZJB, çoğu Lempel-Ziv algoritmasında standart olan sıkı sıkıştırma oranları ile hızlı olacak şekilde tasarlanmıştır. LZJB varsayılandır. ZLE, çok hafif sıkıştırma oranlarına sahip bir hız canavarıdır. LZJB, performans ve sıkıştırma açısından en iyi sonuçları sağlıyor gibi görünüyor.

Son dönemlerde pekçok dağıtımın ve paket yönetim sisteminin göç ettiği ZSTD ise bize GZIP kadar iyi sıkıştırma desteği verirken CPU açısından daha az maliyetli hesaplamayı desteklemektedir. Bu açıdan bakıldığında performans olarak en az LZJB kadar iyi sonuçlar sağlamaktadır.



**NOT:** Bir veri kümesinde sıkıştırmanın etkinleştirilmesi geriye dönük olarak verilerin sıkıştırılacağı anlamına gelmez! Yalnızca yeni kaydedilmiş veya değiştirilmiş veriler için geçerli olacak şekilde verileri sıkıştıracaktır. Veri kümesindeki önceki veriler sıkıştırılmamış olarak kalacaktır. Bu nedenle, sıkıştırmayı kullanmak istiyorsanız, verileri işlemeye başlamadan önce etkinleştirmelisiniz.



ZFS hiyerarşilerinde veri kümesinin sıkıştırma özelliği `compression` parametresi ile ayarlanmaktadır. Bu parametre sıkıştırma algoritmasının tipini parametre olarak almaktadır.

```text
~# zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  11.2G   230K  11.2G        -         -     0%     0%  1.00x    ONLINE  -
~# zfs create tank/deneme
~# zfs set compression=lz4 tank/deneme
~# zfs list
NAME          USED  AVAIL     REFER  MOUNTPOINT
tank          261K  10.9G     25.5K  /tank
tank/deneme    24K  10.9G       24K  /tank/deneme
```

Şimdi boyutu belirli bir boyutu kullanarak **`urandom`** ile elde edilmiş dosyayı kök dizine yazalım.

```text
~# dd bs=1024 count=100000 < /dev/urandom > dosya
100000+0 kayıt girdi
100000+0 kayıt çıktı
102400000 bytes (102 MB, 98 MiB) copied, 0,651381 s, 157 MB/s
~# ls -lh dosya
-rw-r--r--. 1 root root 100M Mar  3 13:15 dosya

```

Şimdi bu dosyayı **`/tank/deneme`** yoluna yaşıyalım.

```text
~# mv dosya /tank/deneme
~# ls -lh /tank/deneme/
toplam 98M
-rw-r--r--. 1 root root 98M Mar  3 13:15 dosya
```

Gördüğümüz gibi 2 MB kadarı sıkıştırmadan kazanıldı. Şimdi sıkıştırma oranını inceleyelim.

```text
~# zfs get compressratio tank/deneme
NAME         PROPERTY       VALUE  SOURCE
tank/deneme  compressratio  2.14x  -
```

### Hiyerarşileri Tekilleştirme

Hiyerarşilerde en optimum şekilde alanı kullanmak ve alandandan tasarruf etmek için sıkıştırma haricinde başka bir yol daha vardır. Buna tekilleştirme ismi verilir. Üç ana veri tekilleştirme türü vardır: dosya tekelleştirme, blok tekelleştirme ve bayt tekelleştirmedir. 

Dosya tekilleştirme, sistem kaynakları bazında en yüksek performanslı ve en az maliyetli olanıdır. Her dosyaya SHA-256 gibi bir karma algoritması uygulanır. Karma birden çok dosya için eşleşirse, yeni dosyayı diskte depolamak yerine, meta verilerde depolar ve orijinal verilere başvurmak için bu dosyaya başvururuz. Dosya tekilleştirme özellikle içerisindeki verileri sıkça tekrarlanan dosyaları depolarken önemli boyut tasarruflar sağlayabilir, ancak ciddi bir dezavantajı vardır. Dosyada tek bir bayt değişirse, karmalar artık eşleşmeyecektir, artık dosya sistemi meta verilerindeki tüm dosyaya referans veremeyeceğimiz anlamına gelir. Referansları kaybedilen veriler tamamen kaybolma riskine sahiptir. Bu nedenle, tüm blokların bir kopyalarını oluşturmamız ve yedekli olarak çalışmamız gerekir. Ayrıca, büyük dosyalar için tekelleştirme performansı negatif olarak etkileyecektir.

Bir diğer tekelleştirme türü ise, bayt tekilleştirmedir. Bu tekilleştirme yöntemi en maliyetli yöntemdir, çünkü tekilleştirilmiş ve benzersiz bayt bölgelerinin nerede başlayıp biteceğini belirlemek için "bağlantı noktalarını" tutmanız gerekir. Sonuçta, baytlar bayttır ve hangi dosyaların onlara ihtiyacı olduğunu bilmemek demek, bir dosyayı bulmak için samanlıkta iğne aramak demektir. Bu tür bir tekilleştirme, bir dosyanın birden çok kez depolanabildiği depolamada işe yarar.

Bir diğer tekelleştirme yöntemi olarak da blok tekilleştirme var. ZFS yalnızca blok tekilleştirmeyi kullanır. Blok tekilleştirme, farklı olan bloklar hariç, bir dosyadaki tüm aynı blokları paylaşır. Bu, yalnızca benzersiz blokları diskte depolamamıza ve RAM'de paylaşılan bloklara başvurmamıza izin verir. Bayt tekilleştirmeden daha verimli ve dosya tekilleştirmeden daha esnektir. Bununla birlikte, bir dezavantajı vardır, hangi blokların paylaşıldığını ve hangilerinin paylaşılmadığını takip etmek için büyük miktarda bellek kullanımı gerektirir. Bununla birlikte, dosya sistemleri verileri blok segmentlerinde okuyup yazıldığından, modern bir dosya sistemi için blok tekilleştirmeyi kullanmak en mantıklı olanıdır.

Paylaşılan bloklar, "tekilleştirme tablosu" adı verilen bölümde saklanır. Dosya sisteminde ne kadar çok kopyalanmış blok varsa, bu tablo o kadar büyür. Veriler her yazıldığında veya okunduğunda, tekilleştirme tablosuna başvurulur. Bu, tüm tekilleştirme tablosunun RAM'de tutmamız gerektiği anlamına gelir. Yeterli RAM'e sahip değilseniz, tablo diske taşacaktır ve bu, hem veri okuma hem de yazma açısından depolama alanınızda büyük performans etkileri olabilir.

#### Tekilleştirmenin Maliyeti

Tekilleştirmede RAM kullandığımız için akıllarımıza gelen soru şu: Tekilleştirme tablonuzu depolamak için ne kadar RAM'e ihtiyacınız var? Bu sorunun kolay bir cevabı yok. Birincisi tekilleştirme tablomuz, depolama havuzunuzdaki blok sayısı tekelleştirme tablosunu oluşturmak için esas alınır. Bu sebeple ne kadar çok blok gerekirse o kadar büyük bir tekelleştirme tablosu gerekecek dolayısı ile daha çok RAM gerekecektir. 

Blok bilgilerini `zdb` komutu ile öğrenebiliriz. 

```text
~# zdb -b tank
Traversing all blocks to verify nothing leaked ...

loading concrete vdev 3, metaslab 14 of 15 ...

        No leaks (block sum matches space maps exactly)

        bp count:                   916
        ganged count:                 0
        bp logical:           109821952      avg: 119892
        bp physical:          102536192      avg: 111939     compression:   1.07
        bp allocated:         102746624      avg: 112168     compression:   1.07
        bp deduped:                   0    ref>1:      0   deduplication:   1.00
        Normal class:            187392     used:  0.00%

        additional, non-pointer bps of type 0:         48
        Dittoed blocks on same vdev: 5
        indirect vdev id 0 has 4 segments (4 in memory)

```

Bu durumda, depolama havuzumuz içerisinde 916 kullanılmış blok vardır \("bp count" sayısı\). 

Havuzdaki her tekilleştirilmiş blok için yaklaşık 320 bayt RAM gerektirir. Dolayısıyla, blok başına 320 bayt ile çarpılan 916 blok için bize yaklaşık 0.2 MB RAM gerekmekte. Toplamda 10 GB için 100 milyon bloğa sahip olduğumuzu varsayarsak bu bizim 1.3MB tekelleştirme tablosuna ihtiyacımız vardır. Bu, her 1 GB dosya sistemi için 3,1 MB tekilleştirme tablosu, 1 TB disk başına 3,12 GB tekelleştirme tablosu ihtiyacı anlamına gelmektedir yanı 1TB diski tekilleştirdikten sonra bizim 3.12 GB boyutunda RAM kullanmamız gerekmektedir.

Depolamanızı önceden planlıyorsanız ve verileri işlemeden önce boyutunu bilmek istiyorsanız, ortalama blok boyutunuzun ne olacağını bulmanız gerekir.  
Bu durumda, verilere yakından aşina olmanız gerekir. ZFS, verileri 128 KB'lik bloklar halinde okur ve yazar. Ancak, çok sayıda yapılandırma dosyası, ana dizin vb. depoluyorsanız, dosyalarınız 128 KB'tan küçük olacaktır. Bu örnek için, yukarıdaki örneğimizde olduğu gibi ortalama blok boyutunun 100 KB olacağını varsayalım. Toplam depolama alanım 1 TB boyutundaysa, blok başına 100 KB'ye bölünen 1 TB yaklaşık 10 milyon blok demektir. Blok başına 320 bayt ile çarpıldığında, elimizdeki verileri tekilleştirmek için 3.2 GB RAM kullanmamız gerekiyor.

Sonuçta her ihtimali göz önünde tutarak, her 1 TB disk için 5 GB RAM ihtiyacımız olacağını varsaymak mantıklı olacaktır. Bu durum maliyet açısından oldukça sorunlu gibi dursa da, disk üzerinde, ciddi performans etkilerine sahiptir.

#### Toplam Tekilleştirmenin Ram Maliyeti

ZFS, RAM'de veri tekilleştirme tablosundan daha fazlasını depolar. Ayrıca ARC'yi ve diğer ZFS meta verilerini de depolar. Ve tahmin edin bu durum nelere sebep olabilir?

ZFS dökümanında şu belirtilmiştir: **Veri tekilleştirme tablosu, ARC'nin boyutunun% 25'i ile sınırlandırılmıştır**. Bu da demek ki, 1 TB depolama dizisi için 3.2 GB RAM'e ihtiyacınız olmadığı anlamına gelir. Veri tekilleştirme tablonuzun sığmasını sağlamak için 14 GB RAM gerekir demektir. Başka bir deyişle, tekilleştirme yapmayı planlıyorsanız, RAM ayak izinizi dört katına çıkardığınızdan emin olun, yoksa karnınız bayaa ağrıyacaktır.

#### L2ARC ile tekilleştirme

Tekilleştirme tablosu RAM'de tutulamayacak boyuta ulaştığı zaman diske kaydırılacaktır demiştim, veriler daha yavaş olan plakalı disklere aktarılıp tekelleştirmenin yavaşlamaması adına tekelleştirmeler L2ARC disklerine taşınabilir. L2ARC'niz hızlı SSD'lerden veya RAM sürücülerinden oluşuyorsa, her okuma ve yazma işleminde tekilleştirme tablosunu, plakalı disklerde olduğun kadar kötü etkilemeyecektir. Yine de, SSD'ler RAM'in sahip olduğu yüksek hızlara sahip olmadığı için, bunun hız üzerinde negatif bir etkisi olacaktır. Bu nedenle, gece veya haftalık yedekleme sunucuları gibi performansın kritik olmadığı depolama sunucuları için, L2ARC üzerindeki tekilleştirme tablosu mantıklı olabilir.

#### Tekilleştirmeyi Etkinleştirme

Bir veri kümesi için tekilleştirmeyi etkinleştirmek için "**`dedup`**" özelliğini etkinleştirmeniz gerekmektedir.Ancak burada aynı sıkıştırma konusunda olduğu gibi şöyle bir durum vardır. Tekilleştirmenin aktif edilmesinden itibaren yaşanacak veri değişimleri ve eklemeleri tekilleştirmeye dahil olacaktır ve tekilleştirme yalnızca bu veri lkümesine kaydedilen veriler, yinelenen bloklar için kontrol edilecektir. Ayrıca, tekilleştirilen veriler atomik bir işlem olarak diske aktarılmaz. Bunun yerine, bloklar her seferinde bir blok olacak şekilde diske seri olarak yazılır. Bu nedenle, bloklar yazılmadan önce bir elektrik kesintisi olması durumunda bu veri hiyerarşimizin patates olmasına yol açabilmektedir.

Sıkıştırmada olduğu gibi şimdi de tekilleştirme için `deneme` veri kümesini kullanalım.

```text
~# zfs set dedup=on tank/deneme
~# cd tank/deneme/
~# ls
dosya
~# cp dosya dosya2
~# ls
dosya dosya2
~# zfs get compressratio tank/deneme
NAME         PROPERTY       VALUE  SOURCE
tank/deneme  compressratio  1.74x  -
~# zpool get dedupratio tank
NAME  PROPERTY    VALUE  SOURCE
tank  dedupratio  1.42x  -
~# ls -lh
toplam 181M
-rw-r--r--. 1 root root 98M Mar  3 13:15 dosya
-rw-r--r--. 1 root root 83M Mar  3 14:04 dosya2

# zfs list tank/deneme
NAME          USED  AVAIL  REFER  MOUNTPOINT
tank/deneme   181M  10.8G  181M   /tank/deneme

```

Bu durumda, veriler önce sıkıştırılır, ardından tekilleştirilir. Ham veriler normalde yaklaşık 100 MB disk kaplar, ancak sıkıştırma ve tekilleştirme nedeniyle yalnızca 86 MB yer kaplar ki bu tamamı ile random oluşturulmuş bir dosya için önemli bir tasarruf miktarı demektir.

### ZVol

### iSCSI

### NFS ve Samba ile Disk Hiyerarşisi Paylaşımı


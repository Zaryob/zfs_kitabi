---
description: ZFS'de üst düzey olarak nitelendirilebilecek hiyerarşi komutları.
---

# ZFS Üst Düzey Disk Hiyerarşi Komutları

## ZFS'ye Has Hiyerarşi Özellikleri

### ZFS'de Anlık Görüntü Özelliği

ZFS'de anlık görüntüler \(snapshot\), Linux LVM anlık görüntülerine benzemektedir. Anlık görüntü, birinci sınıf salt okunur bir dosya sistemidir. Anlık görüntüyü aldığınız andaki dosya sisteminin durumunun aynalanmış bir kopyasıdır. Diskimizin o andaki verilerinin bir fotoğrafı gibidir. Disk verileri değişiyor olsa da, tam o fotoğrafı çektiğiniz anda diskin neye benzediğine dair bir imajımız olduğu için o ana geri dönerek verileri kurtarabiliriz. Sonuç olarak veri kümesinin parçası olan veri değişiklikleri, orijinal kopyayı anlık görüntünün kendisinde tutarsınız. Bu şekilde, o dosya sisteminin kalıcılığını koruyabilirsiniz.

Havuzunuzda 2^64'e kadar anlık görüntü tutabilirsiniz, ZFS anlık görüntüleri yeniden başlatma sırasında kalıcıdır ve herhangi bir ek yedekleme deposu gerektirmez; verilerinizin geri kalanıyla aynı depolama havuzunu kullanırlar. Bir ZFS anlık görüntüsü, bu ZFS veri ağacının bir kopyasıdır, ancak bu veri ağacının anlık görüntüsünün hiçbir zaman değiştirilmediğinden imajın %100 sağlam olduğundan emin olabilirsiniz.

Anlık görüntü oluşturmak neredeyse anlıktır ve maaliyeti çok azdır. Bununla birlikte, veriler değişmeye başladığında, anlık görüntü verileri depolamaya başlayacaktır. Birden fazla anlık görüntünüz varsa, tüm anlık görüntülerde birden çok delta izlenmektedir. Yani git'e ait commit yapısından farklı olarak het görüntü kendi deltalarına sahiptir

#### Anlık Görüntü Oluşturma

İki tür anlık görüntü oluşturabilirsiniz: havuz anlık görüntüleri ve veri kümesi anlık görüntüleri. Hangi tür anlık görüntü almak istediğiniz size kalmış. Ancak anlık görüntüye bir ad vermelisiniz. Anlık görüntü adının sözdizimi şöyledir:

```text
havuz/veri kümesi@anlık_görüntü-adı
havuz@anlık_görüntü-adı
```

Bir anlık görüntü oluşturmak için "zfs snapshot" komutunu kullanıyoruz. Örneğin, "tank/deneme" veri kümesinin anlık görüntüsünü almak için şunları çıkarırız:

```text
~# zfs snapshot tank/test@20210403
```

Anlık görüntü birinci sınıf bir dosya sistemi olsa da, standart ZFS veri kümeleri veya havuzlar gibi değiştirilebilir özellikler içermez. Aslında, anlık görüntü hakkındaki her şey salt okunurdur. Örneğin, bir anlık görüntüde sıkıştırmayı etkinleştirmek isterseniz, şu şekilde olur:

```text
# zfs set compression=lzma tank/test@20210403
cannot set property for 'tank/test@20210403': this property can not be modified for snapshots
```

#### Anlık Görüntüleri Listeleme

Anlık görüntüler iki şekilde görüntülenebilir: veri kümesinin kök dizininde yer alan gizli ".zfs" dizinine erişerek veya "zfs list" komutunu kullanarak görüntüleyebiliriz.

```text
~# ls -a /tank/test
./ ../ boot.tar text.tar text.tar.2
~# cd /tank/test/.zfs/
~# ls -a
./  ../  shares/  snapshot/
```

Varsayılan zfs detay dizini gizlidir. Ancak ZFS'deki her şey gibi bunu da değiştirebiliriz:

```text
# zfs set snapdir=visible tank/test
# ls -a /tank/test
./  ../  zfs/
```

Anlık görüntüleri görüntülemenin diğer yolu, "-t anlık snapshot" parametresi ile "zfs list" komutunu kullanmaktır:

```text
# zfs list -t snapshot
NAME                       USED  AVAIL  REFER  MOUNTPOINT
tank/test@20210403            0      -   525M  -
```

Bende bir tane aygıt havuzu ve bir tane snaphot olduğuna dikkat çekerim. Varsayılan olarak, tüm havuzlar için tüm anlık görüntüleri zfs list komutu ile gösterilmektedir.

Çıktıyla daha spesifik olmak istiyorsanız, ister veri kümesi ister depolama havuzu olsun, belirli bir kök sistemin tüm anlık görüntülerini görebilirsiniz. Özyineleme için yalnızca "-r" parametresini kullanmamız ve ardından kök sistemi sağlamanız gerekir. Bu durumda, yalnızca depolama havuzu "tank" ın anlık görüntülerini göreceğim ve diğer havuzların içindekileri göz ardı edeceğim:

```text
~# zpool create tank2
~# zfs snapshot tank2@firstcreated
~# zpool create tank3
~# zfs snapshot tank3@firstcreated
# zfs list -t snapshot
NAME                       USED  AVAIL  REFER  MOUNTPOINT
tank/test@20210403            0      -   525M  -
tank2@firstcreated            0      -   1M    -
tank2@firstcreated            0      -   1M    -
~# zfs list -r -t snapshot tank
NAME                       USED  AVAIL  REFER  MOUNTPOINT
tank/test@20210403            0      -   525M  -
```

#### Anlık Görüntüleri Yok Etme

Bir depolama havuzunu veya bir ZFS veri kümesini yok edeceğiniz gibi, anlık görüntüleri yok etmek için benzer bir yöntem kullanırız. Bir anlık görüntüyü yok etmek için, "zfs destroy" komutunu kullanırız, bu komuta parametre olarak yok etmek istediğiniz anlık görüntüyü vererek anlık görüntüleri yok edebiliriz:

```text
~# zfs destroy tank/test@20210403
```

Bilinmesi gereken önemli bir nokta, bir anlık görüntü varsa, veri kümesinin alt dosya sistemi olarak kabul edilmektedir. Bu nedenle, tüm anlık görüntüler ve iç içe geçmiş veri kümeleri yok edilene kadar bir veri kümesini kaldıramazsınız.

```text
~# zfs destroy tank/test 
cannot destroy 'tank/test': filesystem has children
use '-r' to destroy the following datasets:
tank/test@20210403
```

Anlık görüntüleri yok etmek, diğer anlık görüntülerin tuttuğu ek alanı boşaltabilir, ana disk alanına etkisi hiç omlayacak veya çok az olacaktır çünkü bunlar bu alan anlık görüntülere özgü bir alandır.

#### Anlık Görüntüleri Yeniden Adlandırma

Anlık görüntüleri yeniden adlandırabilirsiniz, ancak oluşturuldukları depolama havuzunda ve ZFS veri kümesinde yeniden adlandırılmaları gerekir. Bunun dışında, anlık görüntüleri yeniden adlandırmak disk hiyerarşisi veya havuzu yeniden adlandırmak kadar basittir:

```text
# zfs rename tank/test@20210403 tank/test@2021-mart-carsamba
```

#### Anlık Görüntüye Geri Dönme

Önceki bir anlık görüntüye geri dönmek, bu anlık görüntü ile geçerli saat arasındaki tüm veri değişikliklerini temizleyecektir. Ayrıca, varsayılan olarak, yalnızca en son anlık görüntüye geri dönebilirsiniz. Daha önceki bir anlık görüntüye geri dönmek için, geçerli saat ile geri dönmek istediğiniz anlık görüntü arasındaki tüm anlık görüntüleri yok etmeniz anlamına gelmektedir. ZFS'de tek kesinti yaşayabileceğimiz an geriye dönme işlemidir.

`zfs rollback` komutu ile bu işlem yapılmaktadır.

```text
~# zfs rollback tank/test@20210403
```

Eğer bu anlık görüntülerden sonra alınmış başka anlık görüntüler varsa bu işlem yapılmayacaktır.

```text
# zfs rollback tank/test@20210403
cannot rollback to 'tank/test@20210403': more recent snapshots exist
use '-r' to force deletion of the following snapshots:
tank/test@20210404
tank/test@20210405
```

#### ZFS Klonları

Bir ZFS klonu, anlık görüntülerden türetilmiş veya bir nevi "yükseltilmiş" yazılabilir bir dosya sistemidir. Klonlar yalnızca anlık görüntülerden oluşturulabilir ve anlık görüntüye bağımlıdır ve bu klonlar, klon var olduğu sürece kalır. Bu, bir anlık görüntüyü klonladıysanız yok edemeyeceğiniz anlamına gelir. Klon, anlık görüntünün verdiği verilere dayanır. Anlık görüntüyü yok etmeden önce klonu yok etmelisiniz.

Klonlar oluşturmak, tıpkı anlık görüntüler gibi neredeyse anlıktır ve başlangıçta herhangi bir ek yer kaplamaz. Klonların aksine, anlık görüntünün tüm başlangıç ​​alanını kaplar. Veriler klonda değiştirildikçe, anlık görüntüden ayrı bir yer kaplamaya başlar.

#### ZFS Klonları Oluşturma

Bir klon oluşturma, "zfs klonu" komutu, klonlanacak anlık görüntü ve yeni dosya sisteminin adı ile yapılır. Klonun, klonla aynı veri kümesinde bulunması gerekmez, ancak aynı depolama havuzunda bulunması gerekir. Örneğin, "tank/test@20210403" anlık görüntüsünü klonlamak ve ona "tank/klon1" adını vermek istersem, aşağıdaki şekilde bunu yapabilirim:

```text
~# zfs clone tank/test@20210403 tank/klon1
~# zfs list -r tank
NAME           USED   AVAIL  REFER  MOUNTPOINT
tank           161M   2.78G  44.9K  /tank
tank/test      37.1M  2.78G  37.1M  /tank/test
tank/klon1     37.1M  2.78G  37.1M  /tank/klon1
```

Klonlarla alakalı bir durum da onları hiyerarşi gibi kullanıyor olmamızdır. Ancak unutmayın ki bu bizim dosya havuzumuzdaki alandan yemektedir.

#### Klonları Yok Etmek

Veri kümelerini ve tabi ki anlık görüntüleri yok ederken olduğu gibi, "zfs destory" komutunu kullanıyoruz. Yine, klonları yok edene kadar bir anlık görüntüyü yok edemezsiniz. Ayrıca bir görüntü bir klona bağlıysa yine başta bu klonu yok etmeden görüntüyü yok edemezsiniz.

```text
~# zfs destroy tank/klon1
```

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

Bir ZVOL, sisteme blok cihaz olarak aktarılan bir ZFS hiyerarşi birimidir. Şimdiye kadar, ZFS dosya sistemi ile uğraşırken, havuzumuzu oluşturmak dışında, hiyerarşiler oluşturmayı ve veri setlerini monte etmeyi gösterdim. Bunları anlatırken blok cihazlarla hiç ilgilenmedim.

ZFS, geleneksel bir dosya sisteminden daha fazlası olduğunu belirtmiştim. ZFS disklerle aygıtlarla ve veri blokları ile çalışırken daha çok bir kullanıcı alanı uygulaması gibi bir kullanım sunuyor. Bu şu denemek, GNU/Linux'ta dosya sistemleriyle çalışırken, ister tam diskler, bölümler, RAID dizileri veya mantıksal birimler olsun, sürekli olarak blok aygıtlarla çalışmamız gerekir. Bunu yaparken çoğunlukla neredeyse alt düzey bazı komutlarla uğraşmamız, sistemde değişimler yapmamız ve hatta saçma sapan zorluklara sebep verecek bazı şeylerle boğuşmamız gerekmektedir. ZFS ise bu dizilerle neredeyse donanım ve çekirdek katmanla kullanıcıyı uğraştırmak yerine basit bir komut seti sayesinde bütün bunları kendisi otomatik bir şekilde halleder, veya neredeyse otomatik bir şekilde.

Şimdi ellerimizi ZVOL ile uğraşırken bütün bu yapıya daha biraz geleneksel disk yönetim sistemlerindeki gibi dalacağız. ZVOL, depolama havuzunuzda bulunan bir ZFS blok cihazıdır. Bu cihaz sayesinde, tek bloklu aygıtın aynalardan veya RAID-Z aygıtları gibi temel RAID dizilerinden ZVOL sayesinde yararlanabileceğimiz ve alt düzeyde bazı değişimler yapabileceğimiz anlamına gelir.

ZVOL bir taraftan disk hiyerarşisinin yararlandığı anlık görüntüler gibi yazma üzerine kopyalama özelliklerinin avantajlarından yararlanırken bir diğer noktada havuzlara ait çevrimiçi sorun temizleme, veri yayımlama, blok sıkıştırma ve veri tekilleştirme özelliklerinden ve hatta havuzlara ait ZIL ve ARC gibi disk sistemlerinden de faydalanır.

Meşru bir blok cihazı olduğu için, ZVOL'nuzla çok ilginç şeyler yapabilirsiniz. İlginç derken ciddi manada saçma sapan gelebilecek şeyleri bile yapabiliriz.

#### ZVOL oluşturmak

Bir ZVOL oluşturmak için, "zfs create" komutumuzla "-V" parametresini kullanıyoruz ve ona bir boyut veriyoruz. ZVOL'ü ZFS havuzları altında oluşturuyoruz, hiyerarşilerden tek farkımız ZVOL oluştururken boyut belirlememiz gerekmektedir. Şimdi 1G boyutunda bir ZVOL oluşturalım:

```text
# zfs create -V 1G tank/zvoldisk
# ls -l /dev/zvol/tank/zvoldisk
lrwxrwxrwx  2 root root   4096 Şub  1 14:41  /dev/zvol/tank/disk1 -> ../../zd145
# ls -l /dev/tank/zvoldisk
lrwxrwxrwx  2 root root   4096 Şub  1 14:41  /dev/tank/zvoldisk -> ../../zd145
```

ZVOL ile alakalı farkettiğimiz gibi resmen /dev dizini içerisinde bir aygıt oluşturmul gibi görünmekteyiz, bu diskin 1 GB boyutunda olduğu ve %100 gerçek bir blok cihaz olduğu için, başka herhangi bir blok cihazla yapacağımız her şeyi yapabiliriz. İşte bu esneklik sayesinde ZFS'ye mükemmel bir esneklik ekleme avantajını elde ederiz. Ayrıca, boyutundan bağımsız olarak bir ZVOL oluşturmak neredeyse anında gerçekleşir. Neredeyse bilgisayara yeni bir disk bağlamaktan daha hızlı olarak bu gerçekleştirilebilir.

Bu ZVOL aygıtının bir blok cihazı olduğunu belirtmiştim. Bu aygıt aynı bir img gibi bir blok aygıtı oluştup bağlamamamız mümkün. Ayrıca diğer herhangi bir blok cihazda olduğu gibi, onu biçimlendirebilir, takas için ve hatta ext4 formatında disk olarak bile kullanabiliriz. Ama ZVOL aygıtları bu kadar zarif ve geniş özelliklere sahip değil, hatta ciddi sınırlamalara bile sahip.

İlk olarak, varsayılan olarak ZVOL blok cihazlarınız için her bir havuz için yalnızca 8 cihaz oluşturabilme hakkımız vardır. Ancak bu miktarı havuz üzerinden değiştirebiliriz. ZFS ile varsayılan olarak maksimum 2^64 ZVOL aygıtı oluşturabiliriz. Ayrıca, dosya sisteminizin üstünde önceden tahsis edilmiş bir imaj gerektirir. Yani, ZFS sayesinde üç veri katmanını yönetebiliyoryz: blok aygıtı, dosya ve dosya sistemindeki bloklar. ZVOL'larla, blok cihazı, tıpkı diğer veri kümeleri gibi yönetim yapabilir, ve bu cihazlardan yararlanarak verilerimizi depolama havuzunun hemen dışına aktarabiliriz.

Bu ZVOL ile yapabileceğimiz bazı şeylere bakalım.

#### ZVOL'de Ext4

Bu kulağa tuhaf gelebilir, ancak ZVOL aynen bir donanım aygıtı gibidir, başka bir dosya sistemini ZVOL'ü biçimlendirmek için kullanabilir ve onu bir ZVOL'un üzerine bağlayabilirsiniz. Diğer bir deyişle, ext4 formatlı bir ZVOL oluşturup ve bunu /mnt'ye bağlayabiliriz. Hatta ZVOL'u alt bölümlere ayırabilir ve üzerine birden çok dosya sistemi koyabiliriz. Hadi bunu yapalım!

```text
~# zfs create -V 100G tank/ext4
~# fdisk /dev/tank/ext4
( follow the prompts to create 2 partitions- the first 1 GB in size, the second to fill the rest )
~# fdisk -l /dev/tank/ext4

Disk /dev/tank/ext4: 107.4 GB, 107374182400 bytes
16 heads, 63 sectors/track, 208050 cylinders, total 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 8192 bytes
I/O size (minimum/optimal): 8192 bytes / 8192 bytes
Disk identifier: 0x000a0d54

          Device Boot      Start         End      Blocks   Id  System
/dev/tank/ext4p1            2048     2099199     1048576   83  Linux
/dev/tank/ext4p2         2099200   209715199   103808000   83  Linux
~# ls -l /dev/tank/ext4p1 
lrwxrwxrwx  2 root root   4096 Şub  1 14:41  /dev/tank/ext4p1 -> ../../zd0p1
~# ls -l /dev/tank/ext4p2
lrwxrwxrwx  2 root root   4096 Şub  1 14:41  /dev/tank/ext4p2 -> ../../zd0p2
```

ZFS 2 ZVOL aygıtımızı /dev/zd0p1 ve /dev/zd0p2 olarak linklemiş oldu. Bunları biçimlendirelim.

```text
~# mkfs.ext4 /dev/zd0p1
~# mkfs.ext4 /dev/zd0p2
~# mkdir /mnt/zd0p{1,2}
~# mount /dev/zd0p1 /mnt/zd0p1
~# mount /dev/zd0p2 /mnt/zd0p2
```

Gördüğümüz gibi diskler oluşturup bunları bağlamış olduk ama bu ne işimize yaradı. Mesela ZFS'nin anlık görüntü oluşturma özelliğini ext4 ile kullanabiliriz.

```text
~# zfs snapshot tank/ext4@001
```

ext4'ün normalde size veremeyeceği şeylere ve ZFS'nin tüm avantajlarına ZVOL sayesinde sahibiz. Bu senaryoda, artık düzenli olarak verilerinizin anlık görüntüsünü alıyorsunuz, çevrimiçi fırçalamalar gerçekleştiriyor ve yedekleme için iş yeri dışına gönderiyorsunuz. En önemlisi, verileriniz oldukça tutarlı bir şekilde korunabiliyor ve ek bir diske gerek kalmadan yedeklemeler ve geriye dönük imajlandırma işlemleri yapılabiliyor.

### iSCSI, NFS ve Samba ile Disk Hiyerarşisi Paylaşımı

ZFS dosya hiyerarşileri ağ üzerinden aynı bir dosya paylaşım sisteminde olduğu gibi paylaşılmaktadır. ZFS hiyerarşileri, örneğin NFS gibi bir dosya paylaşım sunucusu aracılığıyla belirli bir veri kümesini paylaşabilirsiniz. Ancak, veri kümesini bağlamazsak, bu durumda dışa aktarma uygulama tarafından kullanılamaz ve NFS istemcisi engellenir.

Ağ paylaşımı ZFS dosya sisteminin doğasında olduğundan, veri tutarsızlıkları için endişe etmemize veya herhangi bir işlem yapmamıza gerek yoktur. ZFS'nin sağladığı bütün veri koruma yapıları ve sistemleri paylaşılmış hiyerarşiler için de uygulanmaktadır.

Veri paylaşımına genel bakışım şu. Ek bir kurulum yapmadan adeta dosya yöneticisinde dolaşır gibi başka bir bilgisayar aracılığı ile dosyalarımıza erişebilir ve bunları düzenleyebiliriz.Ancak her halükarda, paylaşımı sağlamak için gerekli daemonları kurmanız gerekir. Örneğin, bir veri kümesini NFS aracılığıyla paylaşmak istiyorsanız, NFS sunucusunu yüklememiz ve çalıştırmamız gerekir. Ardından, yapmanız gereken tek şey, veri kümesindeki paylaşım NFS anahtarını aktifleştirmektir ve bu işlemin hemen ardından hiyerarşimiz kullanılabilir olacaktır.

#### NFS aracılığıyla paylaşma

Bir veri kümesini NFS aracılığıyla paylaşmak için, önce NFS sunucusunu açmamız gerekmektedir. Debian ve Ubuntu'da NFS sunucusu "nfs-kernel-server" paketi tarafından sağlanmaktadır. Ayrıca, Debian ve Ubuntu ile, `/etc/export` dosyasında bir dışa aktarım talimatnamesi olmadığı sürece NFS arka plan programı başlamayacaktır. Bu nedenle paylaşıma başlamak için, yalnızca localhost tarafından kullanılabilen sahte bir dışa aktarma oluşturabilir veya başlangıç ​​komut dosyasını geçerli bir dışa aktarmayı engellemeyecek şekilde düzenleyebiliriz.

Bir diğer yandan OpenSUSE, Fedora, RHEL ve CentOS için `nfs-utils` paketi nfs paylaşımı için kullanılacaktır. Yine aynı şekilde `firewalld` üzerinden portları aktifleştirmemiz gerekmektedir.

Gerekli paketleri kurmamızın ardından NFS sunucumuzu başlatalım:

```text
~# service nfs start
```

Şimdi de ip adresimize bakalım

```text
~# ifconfig

enp2s0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether c8:d9:d2:eb:e9:d7  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 31570  bytes 10961690 (10.4 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 31570  bytes 10961690 (10.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlo1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.123  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::3eb2:86f:1c57:47c4  prefixlen 64  scopeid 0x20<link>
        ether 74:40:bb:e0:e3:db  txqueuelen 1000  (Ethernet)
        RX packets 174848  bytes 291792813 (278.2 MiB)
        RX errors 0  dropped 4118  overruns 0  frame 0
        TX packets 121922  bytes 29077165 (27.7 MiB)
        TX errors 0  dropped 3 overruns 0  carrier 0  collisions 0
```

Ve artık hazırız, ZFS veri kümesini paylaşalım

```text
~# zfs set sharenfs="rw=@192.168.1.123/24" tank/srv
~# zfs share tank/srv
```

#### Samba aracılığıyla paylaşma

NFS'de olduğu gibi, SMB/CIFS aracılığıyla bir ZFS veri kümesini paylaşmak için, arka plan sunucusunun kurulu ve çalışıyor olması gerekir. SMB 2.1 dosya paylaşım desteği, kümelenmiş dosya sunucuları ve çok daha fazlasını verimektedir. Samba 4 desteği ile beraber disk üzerinde NFS'den daha fazlasını daha esnekçe yapabileceğimiz bir yapıya sahibiz.

```text
~# zfs set sharesmb=on tank/srv
~# zfs share tank/srv
```

#### iSCSI aracılığı ile paylaşma

SMB ve NFS'de olduğu gibi, iSCSI arka plan programının kurulu ve çalışıyor olması gerekir.

```text
~# zfs set shareiscsi=on tank/srv
```

## ZFS Disk Hiyerarşilerinin Detaylı Öznitelikleri

ZFS disk hiyerarşilerinde aynı havuzlarda olduğu gibi bir öznitelik mantığı bulunmakta. Bayrak yapısını karşılayan bu öznitelikler havuzun bazı kısımlarının farklı bazı kısımlarının farklı davranması sağlanarak bütün bir havuzun farklı parçalara ayrılıp farklı şekillerde çalışmasına imkan sağlamayarak farklı diskler kullanmadan farklı amaçlar için özelleştirilmiş hiyerarşileri aynı havuz içerisinden kullanabiliriz. (Bayaa uzun cümle kurmuşum haa)

Bu bölümde sadece ZFS hiyerarşilerinin özniteliklerinden bahsedeceğiz. Ancak bunu havuz özniteliklerinden farklı olarak örneklerle göstereceğiz.

### Belli Başlı Bazı Öznitelikler

* **allocated**: Tüm ZFS hiyerarşileri tarafından havuza kaydedilen veri miktarıdır. Bu ayar salt okunurdur
* **type**: Tüm ZFS hiyerarşilerinin tipini belirlemede kullanılır
* **used**: Bu hiyerarşide kullanılan boyutu belirtir.
* **compressratio**: Dosya hiyerarşisinin sıkıştırma oranını belirtir.
* **guid**: ZFS hiyerarşisine donanım GUID değeri tarzı sanal değer atar. Bu değeri tutan parametredir.
* **quota**: Bu hiyerarşide kullanılacak maksimum boyutu belirler. Bu boyut sanal bir boyuttur. Belli bir hiyerarşinin havuzu domine etmesini önlemek için kullanabiliriz. 
* **sharenfs**: NFS ile hiyerarşinin paylaşılabilirliğini kontrol eden parametredir.
* **sharensmb**: SAMBA ile hiyerarşinin paylaşılabilirliğini kontrol eden parametredir.
* **readonly**: Hiyerarşiyi raw okunabilir konuma getirir.
* **compression**: Hiyerarşiyi sıkıştırmayı ayarlar.
* **copies**: Hiyerarşinin ayne ve klon sayısını belirtir.
* **version**: Hiyerarşinin versiyonunu belirler.

### Öznitellikleri Görüntüleme

Depolama havuzu öznitelliklerini görüntüleme ve ayarlamada olduğu gibi, hiyerarşi öznitelliklerini de alabileceğiniz birkaç yol vardır - tüm öznitelikleri bir kerede, yalnızca bir mülk veya birden fazla virgülle ayrılmış olarak alabilirsiniz. 

```shell
~# zfs get all tank/ROOT/root 
NAME            PROPERTY              VALUE                   SOURCE
tank/ROOT/root  type                  filesystem              -
tank/ROOT/root  creation              Prş Mar 18 20:26 2021  -
tank/ROOT/root  used                  24K                     -
tank/ROOT/root  available             14.5G                   -
tank/ROOT/root  referenced            24K                     -
tank/ROOT/root  compressratio         1.00x                   -
tank/ROOT/root  mounted               yes                     -
tank/ROOT/root  quota                 none                    default
tank/ROOT/root  reservation           none                    default
tank/ROOT/root  recordsize            128K                    default
tank/ROOT/root  mountpoint            /tank/ROOT/root         default
tank/ROOT/root  sharenfs              off                     default
tank/ROOT/root  checksum              on                      default
tank/ROOT/root  compression           off                     default
tank/ROOT/root  atime                 on                      default
tank/ROOT/root  devices               on                      default
tank/ROOT/root  exec                  on                      default
tank/ROOT/root  setuid                on                      default
tank/ROOT/root  readonly              off                     default
tank/ROOT/root  zoned                 off                     default
tank/ROOT/root  snapdir               hidden                  default
tank/ROOT/root  aclmode               discard                 default
tank/ROOT/root  aclinherit            restricted              default
tank/ROOT/root  createtxg             10                      -
tank/ROOT/root  canmount              on                      default
tank/ROOT/root  xattr                 on                      default
tank/ROOT/root  copies                1                       default
tank/ROOT/root  version               5                       -
tank/ROOT/root  utf8only              off                     -
tank/ROOT/root  normalization         none                    -
tank/ROOT/root  casesensitivity       sensitive               -
tank/ROOT/root  vscan                 off                     default
tank/ROOT/root  nbmand                off                     default
tank/ROOT/root  sharesmb              off                     default
tank/ROOT/root  refquota              none                    default
tank/ROOT/root  refreservation        none                    default
tank/ROOT/root  guid                  6730011657800072920     -
tank/ROOT/root  primarycache          all                     default
tank/ROOT/root  secondarycache        all                     default
tank/ROOT/root  usedbysnapshots       0B                      -
tank/ROOT/root  usedbydataset         24K                     -
tank/ROOT/root  usedbychildren        0B                      -
tank/ROOT/root  usedbyrefreservation  0B                      -
tank/ROOT/root  logbias               latency                 default
tank/ROOT/root  objsetid              131                     -
tank/ROOT/root  dedup                 off                     default
tank/ROOT/root  mlslabel              none                    default
tank/ROOT/root  sync                  standard                default
tank/ROOT/root  dnodesize             legacy                  default
tank/ROOT/root  refcompressratio      1.00x                   -
tank/ROOT/root  written               24K                     -
tank/ROOT/root  logicalused           12K                     -
tank/ROOT/root  logicalreferenced     12K                     -
tank/ROOT/root  volmode               default                 default
tank/ROOT/root  filesystem_limit      none                    default
tank/ROOT/root  snapshot_limit        none                    default
tank/ROOT/root  filesystem_count      none                    default
tank/ROOT/root  snapshot_count        none                    default
tank/ROOT/root  snapdev               hidden                  default
tank/ROOT/root  acltype               off                     default
tank/ROOT/root  context               none                    default
tank/ROOT/root  fscontext             none                    default
tank/ROOT/root  defcontext            none                    default
tank/ROOT/root  rootcontext           none                    default
tank/ROOT/root  relatime              off                     default
tank/ROOT/root  redundant_metadata    all                     default
tank/ROOT/root  overlay               on                      default
tank/ROOT/root  encryption            off                     default
tank/ROOT/root  keylocation           none                    default
tank/ROOT/root  keyformat             none                    default
tank/ROOT/root  pbkdf2iters           0                       default
tank/ROOT/root  special_small_blocks  0                       default
```

Örneğin, veri kümesinin yalnızca kotasını görüntülemek istediğimi varsayalım. Aşağıdaki komutu verebiliriz:

```shell
~# zfs get quota tank/ROOT/root 
NAME            PROPERTY  VALUE  SOURCE
tank/ROOT/root  quota     none   default
```

Birden fazla özniteliğin alınması için:

```shell
~# zfs get quota,compressratio,available tank/ROOT
NAME       PROPERTY       VALUE  SOURCE
tank/ROOT  quota          none   default
tank/ROOT  compressratio  1.00x  -
tank/ROOT  available      14.5G  -
```

### Öznitellikleri Ayarlama

Her bir hiyerarşide öznitelik ayarlamak bir parametre ile yapılabilir `zfs set` komutu ile öznitelik ayarlanabilir.

Örneğin, veri kümesinin yalnızca sıkıştırma oranını belirlemek istediğimi varsayalım. Aşağıdaki komutu verebiliriz:

```shell 
~# zfs set compression=true tank/ROOT/root
```

Şimdi de bu sıkıştırmaya bir oran verelim:
```shell
~# zfs set compressratio=1.25 tank/ROOT/root 
```

Diğer parametreleri de bu şekilde ayarlayabiliriz.

### Öznitelikleri Miraslama

Miraslama OOP mantığından çok aşina olduğumuz bir konu. Hemen gözümüz korkmasın. Miras olayı burada nesneler üzerinde değil hiyerarşiler üzerinde yapılabilmekte. Bunun temel amacı bir havuza ait bütün hiyerarşilere aynı özellikleri vermektir. `zfs inherit` komutu ile yapılır.

```shell
~# zfs inherit compression tank
~# zfs get -r compression tank
NAME            PROPERTY     VALUE     SOURCE
tank            compression  lzjb      local
tank/ROOT/root  compression  lzjb      inherited from tank
tank/ROOT/test  compression  lzjb      inherited from tank
```

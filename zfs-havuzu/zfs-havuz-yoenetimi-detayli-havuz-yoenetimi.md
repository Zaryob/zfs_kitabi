---
description: Detaylı ZFS Havuz Yönetimi Komutlarını içermektedir.
---

# ZFS Havuz Yönetimi - Detaylı Havuz Yönetimi

Bu kısıma kadar ufak bir giriş yaptığımız ZFS aygıt yönetim sisteminin daha öncesinde bahsetmediğimiz özelliklerine değineceğiz. Bunu yaparken bir önceki bölümde kısaca değindiğimiz aynalanmış disk sistemleri oluşturmanın altında yatan aygıtlar konusunu, özelleştirilmiş aygıtları oluşturma ve bu aygıtların özelliklerini açıklayacağım ve gerçek zamanlı bazı örneklerden bahsedeceğim.

## ZFS Aygıt Tipleri ve Havuzdaki Kullanımları

Bir önceki bölümde bahsettiğim gibi temel olarak 5 bölüm bulunmakta. Bunlar **disk \(varsayılan\):**, **dosya \(file\)**, **ayna \(mirror\)**, **raidz1/2/3**, **yedek \(spare\)**, **önbellek \(cache\)**, **günlük \(log\)** olarak adlandırılmaktadır. Şimdi tek tek bunların özelliklerinden bahsedelim ve havuza bunları eklemeyi görelim.

### Dosya Türünde ZFS Aygıtları Oluşturma

Belirtildiği gibi, önceden tahsis edilmiş dosyaları, mevcut dosya sisteminize \(**ext4**, **xfs** veya her neyse\) müdahale etmeden `zpool`'ları kurmak için kullanılabilir. Bunun, üretim verilerini depolamak için değil, tamamen üretim ve test amaçlı olduğu unutulmamalıdır. Dosyaları kullanmak, sıkıştırma oranını, veri tekilleştirme tablosunun boyutunu veya diğer şeyleri gerçekten üretim verilerini taahhüt etmeden test edebileceğiniz bir sanal alana sahip olmanın harika bir yoludur.

Dosya VDEV'leri oluştururken, göreceli yollar kullanamazsınız, ancak mutlak yollar kullanmanız gerekir. Yani `/tmp/dosya` şeklinde kök dizininden başlayacak şeklinde yolunu belirtmeniz gerekmekte. Ayrıca, görüntü dosyalarına önceden bir boyut tahsis edilmiş olması veya bu boyutun seyrek veya kısmi olarak sağlanması gerekir. Bunun nasıl çalıştığını görelim.

İlk olarak 1 GB'lık alan kaplayacak 4 adet dosya oluşturalım:

```text
~# for i in {1..4}
do
     dd if=/dev/zero of=/tmp/file$i bs=1G count=4 &> /dev/null
done
```

Şimdi bu dosyaları kullanarak bir havuz oluşturalım.

```text
~# zpool create filetank /tmp/file1 /tmp/file2 /tmp/file3 /tmp/file4
```

```text
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

```text
~# zpool create filetank  /dev/sdb /dev/sdc /tmp/file1 /tmp/file2 /tmp/file3 /tmp/file4
```

```text
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

```text
~# zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  7.50G   214K  7.50G        -         -     0%     0%  1.00x    ONLINE  -
~# zpool export tank
~# zpool import tank
cannot import 'tank': no such pool available
```

Bu durumda `/tmp/file1`'i belirterek `tank` havuzunu import etmeye çalışınca

```text
~# zpool import -d /tmp/file1 tank
~# zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  7.50G   300K  7.50G        -         -     0%     0%  1.00x    ONLINE  -
```

### Yansı Aygıtları Oluşturma

Yansıtılmış bir havuz oluşturmak için, `mirror` anahtar sözcüğünü ve ardından aynayı oluşturacak herhangi bir sayıda depolama aygıtını kullanın. Komut satırında mirror anahtar sözcüğü tekrarlanarak birden çok ayna belirtilebilir. Aşağıdaki komut, iki, iki yönlü aynaya sahip bir havuz oluşturur:

```text
~# zpool create tank mirror /dev/sdb /dev/sdc
```

Yansılanmış olan sanal cihazlar oluşturmak için en az iki adet cihaz gerekmektedir. Bu cihazlardan ikinde ana dosyalar depolanırken ikincisinde ise buradaki dosyaların anlık yansıları yani kopyaları bulunmaktadır. Bu sayede bu iki cihazdan birisi bozulursa diğerini kullanarak aygıt havuzumuzu kurtarabiliriz. Burda önemli olan nokta şu. Bu iki yansı diskinden en az bir tanesinin düzgün durumda olması gerekmektedir. Birden fazla diskten oluşan aynalarda da yine aynı durum geçerli. Bu disklerden en az bir tanesinin düzgün olması gerekmekte.

Buraya kadar anlattıklarım pratiğe dökülmediği için anlamsız olabilir. Hemen bir örnek ile izah edelim. Bu örnek için 2 tane sanal dosya aygıtı oluşturalım. İlki tek bir dosyadan oluşsun.

```text
~# zpool create tank /dev/sdb
~# zpool status
  pool: tank
  state: ONLINE
  config:

    NAME          STATE     READ WRITE CKSUM
    tank          ONLINE       0     0     0
      sdb         ONLINE       0     0     0

~# zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  3.75G   105K  3.75G        -         -     0%     0%  1.00x    ONLINE  -
```

Boyut görüldüğü şekildedir. Şimdi 2 diskten oluşan bir havuz oluşturalım.

```text
~# zpool create tank /dev/sdb /dev/sdc
~# zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  7.50G   128K  7.50G        -         -     0%     0%  1.00x    ONLINE  -
```

Bu durumda da boyut bu şekildedir. Ancak 2 diskten oluşan aynalanmış bir havuz için ise şöyle bir tablo karşımıza çıkar.

```text
~# zpool create tank mirror /dev/sdb /dev/sdc
~# zpool status 
  pool: tank
  state: ONLINE
  config:

    NAME            STATE     READ WRITE CKSUM
    tank            ONLINE       0     0     0
      mirror-0      ONLINE       0     0     0
        sdb         ONLINE       0     0     0
        sdc         ONLINE       0     0     0

errors: No known data errors
~# zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  3.75G   122K  3.75G        -         -     0%     0%  1.00x    ONLINE  -
```

Bizim için bu durumda `/dev/sdc` diski `/dev/sdb` için yansı ihtiva etmektedir.

### RAIDZ Aygıtları Oluşturma

Daha öncesinde belirttiğim gibi ZFS'nin kendi RAIDZ yönetimi bulunmakta. RAIDZ'yi anlamak için öncelikli olarak RAID yapısını anlamamız gerekmekte. Genellikle birden çok diske yayılmış verilerde birden çok veri kopyasına ihtiyaç duyulur. Donanımsal olarak bu özellik, bir **RAID** denetleyicisi kullanılarak elde edilebilir. Yani donanımsal olarak birden fazla diski birlikte kullanmaya imkan sağlayan bu yapıya biz RAID diyoruz.

RAID eşlik tabanlı seviyelere ihtiyaç duymakta ve her bir seviye RAID için farklı disk miktarına ve bazı donanımsal gereksinimlere ihtiyacımız var. Örneğin **RAID-5** dizisi için en az 3 aygıttan/diskten oluşan bir disk şeridine ihtiyacımız var. RAID, bunu sağlarken iki diskteki verileri birleştirir, daha sonra, kümedeki tüm üç şeridin **XOR**'u sıfır olarak hesaplanacak şekilde bir eşlik biti hesaplanır. Eşlik daha sonra diske yazılır. Bu, bir disk arızasına maruz kalmanıza ve verileri yeniden hesaplamanıza olanak tanır. Ayrıca, RAID-5'te, dizideki tek bir disk eşlik verileri için ayrılmamıştır. Bunun yerine, eşlik tüm disklere dağıtılır. Böylece, herhangi bir disk arızalanabilirse veriler yine de geri yüklenebilir. Ancak RAID-5'in bir sorunu var. Herhangi bir eşlik biti \(parity\) yazımı sırasında diskle veya donanımla alakalı bir hata meydana gelmesi durumunda bütün veriler kaybedilebilir. Kötü olan durum ise, çoğu donanım yazılım tabanlı RAID'in bir sorunun varlığını tespit edemeyip veriyi öylece yazmasıdır. Eşlik bitlerinin verilerle tutarsız olduğunu belirleyen yazılım çözümleri var, ancak bunlar yavaş ve güvenilir değiller. Sonuç olarak, yazılım tabanlı RAID, depolama yöneticilerinin çoğu için iyi bir çözüm değil. Donanım destekli RAID ise arızalara ve kararsızlıklara sebep olmasından dolayı yönetmesi ve sürekliliği hayli zordur. Çoğunlukla RAID cihazlarını kullanmak için BİOS üzerine güncellemeler yapılmalı, bazı durumlarda işletim sisteminin cihaz için uygun şekilde yamanmasını gerekmektedir.

ZFS, birim yöneticisi ve dosya sistemlerinin görevlerini birleştirir. Bu, yeni bir havuz oluştururken diskleriniz için aygıt düğümlerini belirtebileceğiniz anlamına gelir ve ZFS bunları tek bir mantıksal havuzda birleştirir ve daha sonra bu birimin üstünde farklı kullanımlar için veri kümeleri ve disk alanları oluşturabilirsiniz. RAID için bir alternatif olarak ZFS, RAID'in donanımsal katmanını tahliye etmek için kendi RAID yapısını getirmektedir.

ZFS kendi RAIDZ'si sayesinde donanım bazlı RAID'e aygıt yönetimi noktasında bir alternatif oluşturur. RAIDZ oluşturması sırasında şerit genişliğinin statik olarak ayarlanmasından ziyade, şerit genişliği dinamiktir. İşlemsel olarak diske aktarılan her blok kendi şerit genişliğine sahiptir ve bu sayede veri sağlaması yapılabilir. Bu da RAID'in sorunlarına karşı bir çözüm oluşturur. Ayrıca her RAIDZ yazımı, tam şeritli bir yazma sağlar. Ayrıca, eşlik biti şeritle eşzamanlı olarak işlenmektedir ve bu özelliği daha önce belirttiğim RAID-5'in yazma sorunlarını tamamen ortadan kaldırılır. Dolayısıyla, bir elektrik kesintisi durumunda, ya en son veri akışına sahip olursunuz ya da son veri akışının tamamlanmamış kısımlarını kaybedersiniz ancak bu sebeple herhangi bir disk sorunu yaşanmamış olursunuz.

Ayrıca ZFS'nin RAIDZ yapısı donanım tabanından daha az işlem maaliyetine sahip bir çözüm getirmekte. ZFS, sessiz hataları algılayabilir ve anında düzeltebilir. Bir an için dizideki diskte herhangi bir nedenle verilerin hatayla değiştirildiğini veya bozulduğunu varsayalım. Uygulama veriyi talep ettiğinde, ZFS şeridi az önce öğrendiğimiz gibi oluşturulur ve her bloğu `fletcher4` olarak bilinen bir sağlama toplamı ile karşılaştırır meta verilerdeki sağlama toplamları ile karşılaştırır. Okuma şeridi sağlama toplamı ile eşleşmiyorsa, ZFS bozuk bloğu bulur, ardından eşlik bitlerini okur ve kombinatoryal yeniden yapılandırma yoluyla düzeltir. Daha sonra uygulamaya düzeltilmiş veriler döndürür. Tüm bunlar, özel bir donanımın yardımı olmadan ZFS'nin kendisinde gerçekleştirilir.

RAIDZ seviyelerinin bir başka yönü de, şerit dizideki disklerden daha uzunsa, bir disk arızası varsa veya eşlikli veriler bozulursa, verileri yeniden yapılandıramaz. Bu nedenle, ZFS, bunun olmasını önlemek için şeritteki bazı verileri aynalayacaktır.

RAIDZ 5 katmanda incelenir: **RAIDZ-1**, **RAIDZ-2**, **RAIDZ-3**, **RAIDZ-4** ve **RAIDZ-5** olarak bunlar sıralanabilir. Sonda yer alan sayılar da eşlik biti sayısını göstermektedir. Son olarak, performans açısından, aynalar her zaman RAIDZ seviyelerinden daha iyi performans gösterecektir. Hem okurken hem de yazarken ayna yapısı RAIDZ'den daha az maliyete sahiptir. Ayrıca RAIDZ-1, RAIDZ-2'den daha iyi performans gösterecek ve RAIDZ-2'de RAIDZ-3'ten daha iyi performans gösterecektir. Ne kadar çok eşlik biti hesaplamak zorunda kalırsanız, verileri hem okumak hem de yazmak o kadar uzun sürecektir. Ayrıca aynı diskler için üretilen RAIDZ'lerin eşlik bitlerinin artması durumunda kullanılabilir boş alan azalacaktır. Elbette, depolama ve performansın bir kısmını en üst düzeye çıkarmak için **VDEV**'lerinize her zaman şerit ekleyebilirsiniz. RAID-1+0 gibi iç içe geçmiş RAID seviyeleri, eşliksiz diskleri kaybedebileceğiniz esneklik ve şeritten elde ettiğiniz verim nedeniyle "RAID seviyelerinin Kadillağı" olarak kabul edilir. Özetle, en hızlıdan en yavaşa, iç içe olmayan RAID düzeyleriniz şu şekilde performans gösterir:

1. RAID-0 \(en hızlı\)
2. RAID-1
3. RAIDZ-1
4. RAIDZ-2
5. RAIDZ-3 \(en yavaş\)

#### RAIDZ Aygıtları: RAIDZ-1

RAIDZ-1, dizideki tüm disklere dağıtılan tek bir eşlik biti olması açısından RAID-5'e benzer. Her bir disk şeridi için bir adet eşlik biti bulunurken, RAID5'den farklı olarak şerit genişliği değişkendir ve dizideki disklerin genişliğine bağlı olarak RAIDZ yapısı, daha az diski veya daha fazla diski kapsayabilir.

RAIDZ-1'de minimum 3 disk kullanılmalıdır. RAIDZ-1 verileri korumak için bir disk arızasına izin verir. İkici diskte yaşanan bir veri hatası veri kaybına neden olur. RAIDZ-1 için depolama kapasitesi, dizinizdeki en küçük diskin depolanma miktarı ile hesaplanır. Bunlar arasında bir tane disk, eşlik depolama için kullanılır. Örneğin 3 diskimiz bulunması durumunda bunların 2 tanesi veri depolama için kullanılır. Bir diğer yandan depolamayı da 2 disk arasında en düşük kapasiteye sahip disk belirler. Bu disklerden birinin boyutu 100 GB olsun diyelim, diğerinin boyutu 90 GB olması durumunda; toplam disk boyutunun tavan boyutu 180GB olacaktır.

RAIDZ-1 ile bir havuz kurmak için, "raidz1" **VDEV** anahtarını kullanıyoruz:

```text
~# zpool create tank raidz1 /dev/sdb /dev/sdc /dev/sdd
~# zpool status
  pool: tank
  state: ONLINE
  config:

    NAME                 STATE     READ WRITE CKSUM
    tank                 ONLINE       0     0     0
      raidz1-0           ONLINE       0     0     0
        sdb              ONLINE       0     0     0
        sdc              ONLINE       0     0     0
        sdd              ONLINE       0     0     0

errors: No known data errors
~# zpool list
NAME    SIZE   ALLOC    FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  100.5G   190K   100.5G        -         -     0%     0%  1.00x    ONLINE  -
```

#### RAIDZ Aygıtları: RAIDZ-2

RAIDZ-2, dizideki tüm disklere dağıtılan bir ikili eşlik biti olması açısından RAID-6'ya benzer. Şerit genişliği değişkendir. Bu RAIDZ yapısı verileri korumak için iki disk arızasına izin verir. Üçüncü diskte yaşanmış bir hata veri kaybına neden olur. RAIDZ-2'de minimum 4 disk kullanılmalıdır. Disklerden ilki eşlik bitlerini içerir. Veri depolama şeritinin boyutunu önceki durumdaki gibi en küçük boyutlu disk belirler.

RAIDZ-2 ile bir zpool kurmak için "raidz2" **VDEV** anahtarını kullanıyoruz:

```text
~# zpool create tank raidz2 /dev/sdb /dev/sdc /dev/sdd /dev/sde
~# zpool status
  pool: tank
  state: ONLINE
  config:

    NAME                 STATE     READ WRITE CKSUM
    tank                 ONLINE       0     0     0
      raidz2-0           ONLINE       0     0     0
        sdb              ONLINE       0     0     0
        sdc              ONLINE       0     0     0
        sdd              ONLINE       0     0     0
        sde              ONLINE       0     0     0

errors: No known data errors
~# zpool list
NAME    SIZE   ALLOC    FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  182.2G   270K   182.2G        -         -     0%     0%  1.00x    ONLINE  -
```

#### RAIDZ Aygıtları: RAIDZ-3

RAIDZ-3, herhangi bir RAID yapısına eşdeğer değildir. RAIDZ-3'te minimum 5 disk kullanılmalıdır. Şerit genişliği değişkendir. Bu RAIDZ yapısı verileri korumak için üç disk arızasına izin verir. Dördüncü diskte yaşanmış bir hata veri kaybına neden olur. Disklerden ilki eşlik bitlerini içerir. Veri depolama şeritinin boyutunu önceki durumdaki gibi en küçük boyutlu disk belirler.

RAIDZ-2 ile bir zpool kurmak için "raidz3" **VDEV** anahtarını kullanıyoruz:

```text
~# zpool create tank raidz3 /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf
~# zpool status
  pool: tank
  state: ONLINE
  config:

    NAME                 STATE     READ WRITE CKSUM
    tank                 ONLINE       0     0     0
      raidz3-0           ONLINE       0     0     0
        sdb              ONLINE       0     0     0
        sdc              ONLINE       0     0     0
        sdd              ONLINE       0     0     0
        sde              ONLINE       0     0     0
        sdf              ONLINE       0     0     0

errors: No known data errors
~# zpool list
NAME    SIZE   ALLOC    FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  273.2G   410K   273.2G        -         -     0%     0%  1.00x    ONLINE  -
```

#### RAIDZ Aygıtları: Hybrid RAIDZ

Ne yazık ki, eşlik tabanlı RAID, özellikle tek bir şeritte çok sayıda diskiniz olduğunda \(örneğin 48 diskli bir JBOD\) yavaş olabilir. İşleri biraz hızlandırmak için, tek büyük RAIDZ sanal diski birden fazla RAIDZ sanal disk şeridine ayırmak bir çözüm olabilir. Bu işleme Hybrid RAIDZ sanal disk şeridi demekteyiz. Bu hibrit sanal disk şeridi işleri kolaylaştırırken, depolamanın devamlılığının sağlanması için kullanılabilir disk alanına mal olur ancak performansı büyük ölçüde artırabilir.

Elbette, önceki RAIDZ sanal disklerinde olduğu gibi, şerit genişliği her yuvalanmış RAIDZ disk şeridi için değişkendir. Her RAIDZ seviyesi için, her sanal diskte o kadar çok diski kaybedebilirsiniz. Örneğin, üç RAIDZ-1 sanal disk şeridiniz varsa, sanal disk başına bir disk olmak üzere toplam üç disk arızasına maruz kalabilirsiniz. Kullanılabilir alan benzer şekilde hesaplanacaktır. Ve her sanal diskteki eşlik depolaması nedeniyle üç diski kaybedersiniz.

Bu kavramı açıklamak için, 12 diskli bir depolama sunucumuz olduğunu varsayalım ve şerit performansını en üst düzeye çıkarırken olabildiğince az disk kaybetmek istiyoruz. Bu nedenle, her biri 3 diskten oluşan 4 RAIDZ-1 sanal diski oluşturacağız. Bu bize 4 disk kullanılabilir depolamaya mal olacak, ancak aynı zamanda bize 4 disk arızasına maruz kalma yeteneği verecek ve 4 sanal diskteki şerit performansı artıracaktır.

4 RAIDZ-1 sanal diski ile bi havuz kurmak için, komutumuzda "raidz1" parametresini 4 kez kullanıyoruz

```text
~# zpool create tank raidz1 /dev/sdb /dev/sdc /dev/sdd raidz1 /dev/sde /dev/sdf /dev/sdg raidz1 /dev/sdh /dev/sdi /dev/sdk raidz1 /dev/sdl /dev/sdm /dev/sdn
~# zpool status
  pool: tank
  state: ONLINE
  config:

    NAME                 STATE     READ WRITE CKSUM
    tank                 ONLINE       0     0     0
      raidz1-0           ONLINE       0     0     0
        sdb              ONLINE       0     0     0
        sdc              ONLINE       0     0     0
        sdd              ONLINE       0     0     0
      raidz1-0           ONLINE       0     0     0
        sde              ONLINE       0     0     0
        sdf              ONLINE       0     0     0
        sdg              ONLINE       0     0     0
      raidz1-0           ONLINE       0     0     0
        sdh              ONLINE       0     0     0
        sdi              ONLINE       0     0     0
        sdk              ONLINE       0     0     0
      raidz1-0           ONLINE       0     0     0
        sdl              ONLINE       0     0     0
        sdm              ONLINE       0     0     0
        sdn              ONLINE       0     0     0

errors: No known data errors
~# zpool list
NAME    SIZE   ALLOC    FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  547.1G   6000K   347.1G        -         -     0%     0%  1.00x    ONLINE  -
```

### Yedek Aygıtları Oluşturma

ZFS, havuzun bir kısmı veya tamamını disk hasarlarından ve kayıplardan korumak için `spare` olarak da adlandırılan yedekleme altyapısını getirir. Geleneksel aygıt yöneticilerinde yedek yapısı mevcut dosya sisteminin tamamen yedeği olarak kullanılır. Bir veri kaybında manual eya bir ek program ile kaybolan verilere ulaşmayı sağlar. Ancak, bunların aksine ZFS, depolama havuzundaki arızalı veya hatalı bir aygıtı değiştirmek için kullanılabilecek diskleri tanımlamanıza olanak tanır. Yedekler, arızalı cihazın yerini alana kadar bir havuzda etkin yedek cihaz devre dışı kalır.

Yedekler etkin yedekler \(hot spares\) ve pasif yedekler \(cold spares\) olarak ikiye ayrılmaktadır.

Yedekleri otomatik ve elle ayarlayarak etkin yedekleri arızalı bir disk alanını düzeltmek için kullanabiliriz. Disk arızası durumunda arızalanan diski değiştirerek yedeğe bağlamamız halinde eski verilerimize erişebiliriz. Eğer ayarlarsanız `autoReplace` havuz özelliği üzerinde, yedek otomatik olarakyeni cihaz takıldığında havuz içerisine eklenerek tüm kayıp dosyaların geri getirilmesine izin verir.

Etkin olan bir yedeği, kısa süreliğine havuzdan ayırarak veya `offline` moduna alarak verilerin bir noktada dondurulmasını da sağlamamız mümkündür. Bunu yapmamız halinde yedek aktifleştirilene kadar devredışı bırakıldığı halde kalacak ve ancak elle geri getirme işlemlerinin yapılabilmesine imkan tanıyacaktır.

ZFS'de yedek aygıtları `spare` parametresi ile oluşturulur. Örneğin, 4 GB boyuta sahip 4 adet diskten oluşan bir aygıt havuzu oluşturalım. Bu aygıtlardan 3 tanesi bir ayna grubunda bulunsun ve bu havuz için 1 adet de yedek diskimiz bulunsun. Bunu şöyle bir şekilde oluşturabiliriz.

```text
~# zpool create tank mirror /dev/sdb /dev/sdc log /dev/sdd /dev/sde
~# zpool status
  pool: tank
  state: ONLINE
  config:

    NAME             STATE     READ WRITE CKSUM
    tank             ONLINE       0     0     0
      mirror-0       ONLINE       0     0     0
        sdb          ONLINE       0     0     0
        sdc          ONLINE       0     0     0
      spare    
         sdd             ONLINE       0     0     0
        sde            ONLINE       0     0     0

~# zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  3.75G   186K  3.75G        -         -     0%     0%  1.00x    ONLINE  -
```

Gördüğümüz gibi `mirror` blogu tek bir ana disk alanı ihtiva ederken `spare` buna ek olarak bir aygıt alanı getirmemektedir. Yani bu alan fiziksel depolama için kullanılamamaktadır. Ancak dikkat etmemiz gereken bir nokta var. Yedek aygıtlar, havuzdaki en büyük diskin boyutuna eşit veya daha büyük olmalıdır. Aksi taktirde bu cihaz arızalı bir cihazı değiştirmek için etkinleştirildiğinde, işlem başarısız olur:

```text
cannot replace sdd with sdb: device is too small
```

Bu sebeple yedeklerin boyutunu, havuzumuzda alan oluşturan cihazların toplam boyutundan büyük olacak şekilde ayarlamamız gerekmektedir.

### Günlükleme Aygıtları Oluşturma

**ZFS Amaç Günlüğü** veya **ZIL** yazılacak tüm verilerin depolandığı ve daha sonra işlemsel bir yazma olarak temizlendiği bir günlüğe kaydetme mekanizması. Ext3 veya ext4 gibi günlüklü dosya sistemlerinde kullanılan günlük işlevine benzer bir mekanizmaya sahiptir.

Temel olarak dosya sistemilerinde kullanılan amaç günlüğü diskin kayıt listelerini tutan ve diskte kısa süreli olarak geriye yönelik olarak blok değişimlerine izin veren yapıdır.

Genellikle kayıt listeleri disk alanında saklanır. ZFS'de kayıt listeleri geleneksel kayıt listelerinden farklı değildir. Bir kayıt listesi, ZIL blokları ve bir ZIL fragmanına işaret eden bir ZIL başlığından oluşur. ZIL, farklı yazmalar için dinamik olarak ayarlanan bir davranışa sahiptir. 64KB'den küçük veriler için \(varsayılan olarak\), ZIL yazma verilerini depolar. Daha büyük veriler için, yazma ZIL'de depolanmaz ve ZIL, günlük kaydında depolanan senkronize edilmiş verilere işaretçiler tutar.

ZFS'de bu günlükleme haricinde Ayrı Amaç Günlüğü \(Seperated Log\) veya SLOG bulunmakta. SLOG aslında ZIL'in eşzamanlı kayıt parçalarını daha yavaş olan disklere \(sabit disklere\) yazılmadan önce önbelleğe alan ayrı bir günlük kaydı cihazıdır. Bu tip kayıtları tutan diskler, pil destekli bir DRAM sürücüsü veya hızlı bir SSD olması gerekmekte. SLOG, yalnızca eşzamanlanmış verileri önbelleğe alır ve eşzamanlanmamış ham verileri önbelleğe almaz. Eşzamanlanmamış ham veriler doğrudan sabit disklere aktarılır. Ayrıca, bloklar, SLOG'a eşzamanlı işlemler olarak değil, bir seferde tüm blok olarak yazılır. SLOG mevcutsa, ZIL diğer disklerde kalmak yerine özelleştirilmiş SLOG diskine taşınacaktır. SLOG'daki her veri her zaman RAM üzerinde olacaktır. Sonuç olarak `Aygıt Havuzuna ZIL Eklemek` işlemi aslında ZIL'in bulunacağı yere özelleştirilmiş bir SLOG diski eklemeyi açıklıyor diyebiliriz. ZIL, SLOG'un bir alt kümesidir; SLOG bir fiziksel aygıttır, ZIL ise bu cihazdaki verilerdir.

ZIL ile alakalı tek bir problem vardır, tüm uygulamalar ZIL'den yararlanmaz. Veritabanları \(MySQL, PostgreSQL, Oracle\), NFS ve iSCSI gibi uygulamalar ZIL kullanır. Ancak dosya sistemindeki basit kopyalama taşıma işlemleri gibi işlemleri yapan uygulamalar ZIL kullanmaz.

Son ve en önemli nokta ise, ZIL eksik bir işlem olup olmadığını görmek için önyükleme dışında genellikle asla okunmaz. ZIL temelde "salt yazılır" ve daha çok yazma yoğunlukludur.

ZFS'de SLOG-ZIL aygıtları `log` parametresi ile oluşturulur. Örneğin, 4 GB boyuta sahip 4 adet diskten oluşan bir aygıt havuzu oluşturalım. Bu aygıtlardan 3 tanesi bir ayna grubunda bulunsun ve bu havuz için 1 adet de günlükleme diskimiz bulunsun. Bunu şöyle bir şekilde oluşturabiliriz.

```text
~# zpool create tank mirror /dev/sdb /dev/sdc log /dev/sdd /dev/sde
~# zpool status
  pool: tank
  state: ONLINE
  config:

    NAME             STATE     READ WRITE CKSUM
    tank             ONLINE       0     0     0
      mirror-0       ONLINE       0     0     0
        sdb          ONLINE       0     0     0
        sdc          ONLINE       0     0     0
      logs    
         sdd             ONLINE       0     0     0
        sde            ONLINE       0     0     0

~# zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  3.75G   186K  3.75G        -         -     0%     0%  1.00x    ONLINE  -
```

Gördüğümüz gibi `mirror` blogu tek bir ana disk alanı ihtiva ederken `log` buna ek olarak bir aygıt alanı getirmemektedir. Yani bu alan fiziksel depolama için kullanılamamaktadır.

### Önbellek Aygıtları Oluşturma \(The Adjustable Replacement Cache \(ARC\) \)

Önbellek aygıtları, ana bellek ve disk arasında ek bir önbelleğe alma katmanı sağlar. Özellikle statik verilerin rastgele okuma performansını iyileştirmek için oldukça işe yaramaktadır.

Geleneksel önbellekleme işlemleri en son kullanılan verilerin önbellekleme diskinde ön sıraya getirilmesi algoritması \( Least Recently Used Algorithm - LRU \) ile çalışmaktadır. Bu sayede sıklıkla kullanılan veriler daha az maaliyet ile erişebiliriz. Ancak diskte önbelleklenen veri her yeni veri girişinde geriye atılmaktadır. Bu durum şöyle bir senaryo için verimsizdir. Sunucudaki bir program bir adet ana dosyanın okuma ve yazılması üzerine işlemlerler yapmaktadır. Ancak bu programın haricinde başka programlar da rastgele başka dosyalara erişmeye çalışıyor diyelim. Bu durumda mantiken en çok kullanılan dosyanın daha önde olması hızı artıracaktır ama rastgele getirilen dosyalar yüzünden sürekli olarak bu dosya geriye atılır. Ve her ne kadar öncekinden daha hızlı olsa da yine de yavaş bir veri akışı sağlanmış olur. Geleneksel yöntemlerde, önbellek ayrıca bir FIFO \(ilk giren ilk çıkar\) algoritmasıdır. Böylece, önbellek dolduğunda, eski sayfalara daha sık erişilse bile önbellekten çıkarılacaktır. Tüm süreci aslında bir taşıma bandı gibi çalışmaktadır.

Bir diğer yandan geleneksel önbellekleme uygulamalarında kullanılan en az sıklıkla kullanılan verileri öne getiren \( Least Frequently Used \(LFU\) \) önbellekleme algoritmaları da de vardır. Ancak bu durumda da veri, yeterince sık okunmazsa, daha yeni verilerin önbellekten çıkarılır ve yine bu yaklaşım da bir takım yavaşlıklara ve sorunlara neden olmaktadır. Ayrıca bu yöntemin bir sorunu da, işletim sistemlerinin başlatılması veya bir sunucu sisteminin başlatılması esnasında bu çeşit önbellekleri bozan çok sayıda disk isteği yapılır ki bu da önbelleklerimizi zehirler \(snoffing\).

ZFS ayarlanabilir değiştirme önbelleği \(ARC\), her iki yöntemin avantajlarını kullanan ve bu sayede hem son blok isteklerini hem de sık blok isteklerini önbelleğe alan böyle bir önbellek mekanizmasına sahiptir. Aslında ZFS'nin kullandığı ARC yöntemi, patentli IBM ARC algoritmasının basitleştirilerek ZFS için adapte edilmesi ile elde edilen bir yöntemdir.

ZFS ARC önbelleğini anlamak için şu yapılara bakmamız gerekmekte:

* Ayarlanabilir Yedek Önbellek veya ARC - Fiziksel RAM'de bulunan bir önbellektir. Önceden belirttiğim şekilde hem en son kullanılan hem de en sık kullanılan verileri barındıran bir ilkin bellektir. Bu önbellek dizini, en sık kullanılan hayalet önbellekleri \(Ghost MRU ve Ghost MFU\) ve son kullanılan dosyaların aygıttaki işaretçilerini RAM üzerinde önbellek olarak indeksler.
* Önbellek Dizini - MRU, MFU, hayalet MRU ve hayalet MFU önbelleklerini oluşturan işaretçilerden oluşan bir önbellek dizini.
* MRU Önbelleği - ARC'nin en son kullanılan dosyalar önbelleği. Dosya sisteminden en son istenen bloklar burada önbelleğe alınır.
* MFU Önbelleği - ARC'nin en sık kullanılan dosyalar önbelleği. Dosya sisteminden en sık istenen bloklar burada önbelleğe alınır.
* Ghost MRU - MRU'da yer kazanmak için MRU önbelleğinden çıkan verilerin diske basılması ile elde edilen önbellektir. 
* Ghost MFU - MFU'da yer kazanmak için MFU önbelleğinden çıkan verilerin diske basılması ile elde edilen önbellektir. 
* İkinci Seviye Ayarlanabilir Yedek Önbellek veya L2ARC - RAM dışında, genellikle hızlı bir SSD'ye basılarak elde edilen bir dinamik önbellektir. RAM ARC'nin gerçek bir uzantısı iken, L2ARC fiziksel bir uzantısıdır.

ZFS ARC, kullanılabilir RAM'in maksimum 1/2'sini kaplar. Ancak bu değer statik değil. Kullandığınız bilgisayarda 32 GB RAM varsa, bu önbelleğin her zaman 16 GB'ının ZFS tarafından kullanılacağı anlamına gelmez. Toplam önbellek boyutunu kernel kararlarına göre ayarlayacaktır. Kernelin zamanlanmış bir işlem için veya bir uygulamanın anlık olarak daha fazla RAM'e ihtiyacı olması durumunda, ZFS ARC çekirdeğin ihtiyacı olan her şeye yer açmak için ayarlanacaktır. Ancak, ZFS sistemde önbellekleme için öncelik istemesi halinde, ARC'nin kaplayabileceği alan varsa, onu rezerve ederek kullanacaktır.

Önbelleklemelerin kararlılığı ve sistemsel arızaların yük yapmaması açısında, ZFS ARC tasarımı, bazı durumlarda bu önbelleğin tamamı veya büyük bir kısmı düşürülebildiği gibi, önbellek üzerinden kullanılan dosyalar ZFS ARC tarafından mutlak surette RAM üzerinde tutulmaya devam edecek şekilde oluşturulmuştur. Ayrıca hızlı disklerin kullanılması halinde önbelleklemedeki kritik veriler haricindeki önbellekler RAM üzerinden bu disklere aktarılacaktır.

..note :: L2ARC önbelleklemesi için kullanacağımız disk, sağlıklı bir şekilde kullanılması için kesinlikle ama kesinlikle manyetik veya optik okuyuculu bir disk olmamalıdır.

L2ARC önbellek aygıtları yansı aygıtlarında olduğu gibi toplam aygıt havuzunun boyutuna etki etmez. Daha çok bir yan görevlerde kullanılmak üzere özel alan teşkil etmektedir.

ZFS'de önbellek aygıtları `cache` parametresi ile oluşturulur. Örneğin, 4 GB boyuta sahip 4 adet diskten oluşan bir aygıt havuzu oluşturalım. Bu aygıtlardan 3 tanesi bir ayna grubunda bulunsun ve bu havuz için 1 adet de önbellekleme diskimiz bulunsun. Şöyle bir şekilde oluşturabiliriz.

```text
~# zpool create tank mirror /dev/sdb /dev/sdc /dev/sdd cache /dev/sde
~# zpool status
  zpool status
  pool: tank
  state: ONLINE
  config:

    NAME            STATE     READ WRITE CKSUM
    tank            ONLINE       0     0     0
      mirror-0      ONLINE       0     0     0
        sdb          ONLINE       0     0     0
        sdc          ONLINE       0     0     0
        sdd         ONLINE       0     0     0
    cache
      sde            ONLINE       0     0     0

errors: No known data errors
```

Şimdi boyutu inceleyelim:

```text
~# zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  3.75G   141K  3.75G        -         -     0%     0%  1.00x    ONLINE  -
```

Gördüğümüz gibi `mirror` blogu tek bir ana disk alanı ihtiva ederken `cache` buna ek olarak bir aygıt alanı getirmemektedir. Yani bu alan fiziksel depolama için kullanılamamaktadır.

## ZFS Havuzunda Aygıt Yönetim İşlemleri

ZFS havuzuna daha öncesinde belirttiğim gibi aygıt ekleme işlemleri yapılabildiği gibi aygıtları tamamen kapatmadan kapalı konuma getirerek aygıt bazlı işlem yapmamıza işlem sağlayacak altyapıya sahiptir. Yeni bir üst düzey sanal cihaz ekleyerek bir havuza dinamik olarak disk alanı ekleyebilir, özel amaçlı kullanılacak diskler sayesinde yansılama ve önbellekleme gibi işlemler için kullanabilirsiniz. Aynı şekilde bu disklerin bağlarını kaldırabilir ve havuz bakımını çalışma esnasında yapabilirsiniz. 

### ZFS Havuzundan Aygıt Eklemek

Bir havuza yeni bir sanal cihaz eklemek için **`zpool add`** komutunu kullanılır.

```text
~# zpool add tank mirror sdf sdg
~# zpool status 
  pool: tank
  state: ONLINE
  config:

    NAME            STATE     READ WRITE CKSUM
    tank            ONLINE       0     0     0
      mirror-0      ONLINE       0     0     0
        sdb         ONLINE       0     0     0
        sdc         ONLINE       0     0     0
      mirror-1      ONLINE       0     0     0
        sdd         ONLINE       0     0     0
        sde         ONLINE       0     0     0
      mirror-2      ONLINE       0     0     0
        sdf         ONLINE       0     0     0
        sdg         ONLINE       0     0     0
      
```

Sanal aygıtları belirtme biçimi, **`zpool create`** komutu ile aynıdır. Aygıt tipleri belirteçleri sayesinde o tipte aygıt tipi eklenebilir.  Aygıtlar eklenirken, kullanımda olup olmadıklarını belirlemek için kontrol edilir ve kullanımda olmadıkları durumda veya disklerle alakalı bir sorun olmadığı müddetçe komut, **`-f`** seçeneği olmadan havuz içerisine eklenemez. Komut ayrıca bu işlemi önizleyerek çalıştırmak için  **`-n`** seçeneğini vermek gerekmektedir. Örneğin:

```text
~# zpool add -n tank mirror sdf sdg
would update 'tank' to the following configuration:
    tank            ONLINE       0     0     0
      mirror-0      ONLINE       0     0     0
        sdb         ONLINE       0     0     0
        sdc         ONLINE       0     0     0
      mirror-1      ONLINE       0     0     0
        sdd         ONLINE       0     0     0
        sde         ONLINE       0     0     0
      mirror-2      ONLINE       0     0     0
        sdf         ONLINE       0     0     0
        sdg         ONLINE       0     0     0
```

#### Farklı ZFS aygıt çeşitlerinden oluşan aygıt havuzlarını eklemek

Bunu bir senaryo olarak anlatmak istedim. Sunucu için bütünü ile fonksiyonel bir aygıt havuzu oluşturalım. Ve nerdeyse her tipte bir aygıt tipi oluşturarak ekleyelim.

3 tane raid diskinden oluşan ve raidz yapısında olan bir disk havuzu oluşturalım

```text
~# zpool create tank raidz sdb sdc sdd
~# zpool status tank
  pool: tank
  state: ONLINE
  scrub: none requested
  config:

        NAME        STATE     READ WRITE CKSUM
        tank        ONLINE       0     0     0
          raidz1-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0
            sdd     ONLINE       0     0     0

errors: No known data errors
```

Şimdi bu havuza ayna aygıtları ekleyelim

```text
~# zpool add tank mirror sde sdf
~# zpool status tank
  pool: tank
  state: ONLINE
  scrub: none requested
  config:

        NAME        STATE     READ WRITE CKSUM
        tank        ONLINE       0     0     0
          raidz1-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0
            sdd     ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sde     ONLINE       0     0     0
            sdf     ONLINE       0     0     0

errors: No known data errors
```

Havuza şimdi 4 tane önbellekleme diski ekleyelim.



```text
~# zpool add tank cache sdg sdh sdk sdl
~# zpool status tank
  pool: tank
  state: ONLINE
  scrub: none requested
  config:

        NAME        STATE     READ WRITE CKSUM
        tank        ONLINE       0     0     0
          raidz1-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0
            sdd     ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sde     ONLINE       0     0     0
            sdf     ONLINE       0     0     0
          cache-0  ONLINE        0     0     0
            sdg     ONLINE       0     0     0
            sdh     ONLINE       0     0     0
            sdk     ONLINE       0     0     0
            sdl     ONLINE       0     0     0            
```

Bir de log diski ekleyelim



```text
~# zpool add tank log sdm
~# zpool status tank
  pool: tank
  state: ONLINE
  scrub: none requested
  config:

        NAME        STATE     READ WRITE CKSUM
        tank        ONLINE       0     0     0
          raidz1-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0
            sdd     ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sde     ONLINE       0     0     0
            sdf     ONLINE       0     0     0
          cache-0  ONLINE        0     0     0
            sdg     ONLINE       0     0     0
            sdh     ONLINE       0     0     0
            sdk     ONLINE       0     0     0
            sdl     ONLINE       0     0     0  
          logs
            sdm     ONLINE       0     0     0        
```

### ZFS Havuzundan Aygıt Çıkartmak

ZFS aygıt havuzuna disk ekleyebileceğimiz gibi aygıt da çıkartabiliriz

```text
~# zpool create  tank sdb sdc mirror sdd sde
~# zpool status
  pool: tank
  state: ONLINE
  config:

        NAME          STATE     READ WRITE CKSUM
        tank          ONLINE       0     0     0
          sdb         ONLINE       0     0     0
          sdc         ONLINE       0     0     0
          mirror-0    ONLINE       0     0     0
            sdd       ONLINE       0     0     0
            sde       ONLINE       0     0     0

errors: No known data errors
~# zpool remove tank sdb
~# zpool status
  pool: tank
  state: ONLINE
  config:

        NAME          STATE     READ WRITE CKSUM
        tank          ONLINE       0     0     0
          sdc         ONLINE       0     0     0
          mirror-0    ONLINE       0     0     0
            sdd       ONLINE       0     0     0
            sde       ONLINE       0     0     0

errors: No known data errors
```

Aygıt havuzundan disk çıkarmak için uymamız gereken bazı kurallar var. Bunlar:

* Aygıt kaldırma işlemi yalnızca etkin yedeklerin, günlük aygıtlarının ve önbellek aygıtlarının kaldırılmasını desteklemektedir. 
* Ana yansıtılmış havuz yapılandırmasının parçası olan aygıtlar, disk havuzundan kaldırılamaz. 
* Yedeksiz ve RAID-Z cihazları havuzdan kaldırılamaz.
* Ayrılma işleminin tamamlanmasının ardından disk ancak yeniden biçimlendirilerek havuz alanına dahil edilebilir.

### ZFS Havuzundaki Aygıtları Değiştirmek

Aygıt havuzundan kullanılmayan diskleri kaldırabileceğimiz gibi bazı diskleri yenileri ile değiştirebiliriz. 

```text
~# zpool create  sdc tank mirror sdd sde
~# zpool status
  pool: tank
  state: ONLINE
  config:

        NAME          STATE     READ WRITE CKSUM
        tank          ONLINE       0     0     0
          sdc         ONLINE       0     0     0
          mirror-0    ONLINE       0     0     0
            sdd       ONLINE       0     0     0
            sde       ONLINE       0     0     0

errors: No known data errors
~# zpool replace tank sdc sdb -f
invalid vdev specification
the following errors must be manually repaired:
sdb is part of active pool 'tank'
~# zpool status
  pool: tank
 state: ONLINE
remove: Removal of vdev 0 copied 37.5K in 0h0m, completed on Wed Mar  3 11:32:25 2021
    96 memory used for removed device mappings
config:

        NAME          STATE     READ WRITE CKSUM
        tank          ONLINE       0     0     0
          sdb         ONLINE       0     0     0
          mirror-2    ONLINE       0     0     0
            sdd       ONLINE       0     0     0
            sde       ONLINE       0     0     0

errors: No known data errors
```



Bu işlem sonunda ekleyeceğimiz diskin alanı az veya daha fazla olabilir. Bu durumda havuzun `autoexpand` parametresi açıksa yeniden boyutlandırma yapılır. Örnek vermek gerekirse bu parametresi açık olmayan disklerde havuzun boyutu `set` komutu ile bu parametre ayarlanana kadar sabit kalacaktır.

```text
~# zpool list pool
NAME   SIZE   ALLOC  FREE    CAP  HEALTH  ALTROOT
pool  16.8G  76.5K  16.7G     0%  ONLINE  -
~# zpool replace pool sdc sdb
# zpool list pool
NAME   SIZE   ALLOC  FREE    CAP  HEALTH  ALTROOT
pool  16.8G  88.5K  16.7G     0%  ONLINE  -
# zpool set autoexpand=on pool
# zpool list pool
NAME   SIZE   ALLOC  FREE    CAP  HEALTH  ALTROOT
pool  32.2G   117K  32.2G     0%  ONLINE  -
```

### ZFS Havuzundan Aygıtı Çevrimiçi ve Çevrimdışı Hale Getirmek

ZFS, bireysel cihazların çevrimdışına alınmasına veya çevrimiçine alınmasına izin verir. Donanım güvenilir olmadığında veya düzgün çalışmadığında, ZFS, durumun geçici olduğunu varsayarak cihazdan veri okumaya veya cihaza veri yazmaya devam eder. Durum geçici değilse, aygıt havuzunun kararlılığı için ZFS'ye cihazı çevrimdışına alarak yok sayması talimatını verebilirsiniz. ZFS, çevrimdışı bir cihaza herhangi bir istek göndermez.

Depolamanın geçici olarak bağlantısını kesmeniz gerektiğinde,**`zpool offline`**komutunu kullanabilirsiniz.

```text
~# zpool offline tank sdc
~# zpool status
  pool: tank
 state: DEGRADED
status: One or more devices has been taken offline by the administrator.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using 'zpool online' or replace the device with
        'zpool replace'.
  scan: resilvered 190K in 00:00:01 with 0 errors on Wed Mar  3 11:39:49 2021
remove: Removal of vdev 0 copied 37.5K in 0h0m, completed on Wed Mar  3 11:32:25 2021
    96 memory used for removed device mappings
config:

        NAME              STATE     READ WRITE CKSUM
        tank              DEGRADED     0     0     0
          sdb             ONLINE       0     0     0
          mirror-2        DEGRADED     0     0     0
            sdc           OFFLINE      0     0     0
            sdd           ONLINE       0     0     0
          sde             ONLINE       0     0     0

errors: No known data errors

```

Bu işlemi yaparken `remove` komutunda olduğu gibi bazı noktalara dikkat etmemiz gerekmekte.

* Bir havuzun bütününü hatalı hale gelene kadar çevrimdışı duruma getiremezsiniz.
* Bir raidz1 yapılandırmasında iki cihazı çevrimdışına alamazsınız veya üst düzey bir sanal cihazı çevrimdışına alamazsınız.
* Varsayılan olarak, çevrimdışına alınan diskin durumu kalıcıdır. Sistem yeniden başlatılsa bile durumu kalıcıdır.
* Bir cihaz çevrimdışına alındığında, depolama havuzundan ayrılmaz. 
* Çevrimdışına alınmış bir cihazı, orijinal havuz yok edildikten sonra bile başka bir havuzda kullanamazsınız.

Çevrimdışı durumuna alınmış bir cihazı ya değiştirebiliriz veya geri çevrimiçi haline alabiliriz. Değiştirme komutunu gösterdim. diski geri çevrimiçi konuma almak için **`zpool online`** komutu kullanılır.

```text
~# zpool online tank sdc
~# zpool status
  pool: tank
  state: ONLINE
  scan: resilvered 16.5K in 00:00:01 with 0 errors on Wed Mar  3 11:56:37 2021
  remove: Removal of vdev 0 copied 37.5K in 0h0m, completed on Wed Mar  3 11:32:25 2021
    96 memory used for removed device mappings
  config:
        NAME              STATE     READ WRITE CKSUM
        tank              ONLINE       0     0     0
          sdb             ONLINE       0     0     0
          mirror-2        ONLINE       0     0     0
            sdc           ONLINE       0     0     0
            sdd           ONLINE       0     0     0
          sde             ONLINE       0     0     0

errors: No known data errors

```

## ZFS'de Havuz Aktarım İşlemleri

ZFS'de aygıt havuzları sadece sıfırdan oluşturma yolu ile değil aktarım yolu ile de elde edilebilir. Geleneksel disk yönetim sistemlerinde diskler takıldığı anda diski bağlamak için tek yapmamız gereken diskin bağlama komutunu vermek olacaktır. Çoğunlukla diski bağlamak için kullandığımız komutlar işletim sistemleri tarafından karşılanmaktadır. Ancak, ZFS'de bu diskleri uygun şekilde bulup bağlamamız gerekmektedir.

Yine aynı şekilde geleneksel disk yönetim sistemlerinde diskleri ayırmak için kullanılan disk domutları çoğunlukla işletim sistemi tarafından gelmektedir. ZFS'de ise geleneksel yöntemlerin aksine bu işlem için `zpool` havuz yöneticisi tarafından kullanılan komutları kullanmamız gerekmektedir.

Bir diğer önemli husus ise geleneksel disk yönetim sistemlerinin aksine ZFS'de bu işlemlerde basit bir güç kesme işleminin aksine bazı önemli işlemlerin tamamlanmasını beklemek gerekmektedir, ki bu işlemler tamamen veri güvenliğini sağlamak için kullanılmaktadır. Örneğim dışa aktarma esnasında, yazılmamış verileri diske temizler, diske dışa aktarmanın yapıldığını belirten verileri yazar ve havuzla ilgili tüm bilgileri sistemden kaldırır. İçe aktarma esnasında ise diskleri okuyarak havuzu oluşturur, aynalamaları kontrol eder ve veri kayıplarını tespit eder, kayıp kaynakları bulup eşleştirmeler yolu ile kayıp olan kaynakları telafi etmeye çalışır ve bütün verileri kök sisteme `veya nereye isteseniz` bağlar.

ZFS'de içe aktarma ve dışa aktarma yapmayı daha öncesinde göstermiştim. `zpool import` ve `zpool export` komutları içe ve dışa aktarma işleri için kullanılmaktadır.

### ZFS Havuzunu Dışa Aktarmak

Bir havuzu dışa aktarmak için `zpool export` komutunu kullanılır.

```text
~# zpool export tank
```

Bu komut, devam etmeden önce havuzdaki bağlı dosya hiyerarşilerini kaldırmaya çalışır. Dosya hiyerarşilerini daha detaylandırmadım lakin dosya hiyerarşiler bir şekilde işletim sistemi ile havuz arasındaki bağlantıyı sağlıyor diyebilirim. Bazen dışa aktarma işlemi esnasında işletim sistemine ait işlemlerin bitmesi beklenir. Bu sebeple dosya hiyerarşileri havuzları kilitleyebilir. Dosya hiyerarşilerinin bu kilit durumunu kaldırmak için, `-f` parametresini kullanabilir ve bunları zorla kaldırabilirsiniz. Örneğin:

```text
~# zpool ihracat tankı
'/ export / home / eschrock' bağlantısı kesilemiyor: Cihaz meşgul
~# zpool export -f tankı
```

Havuzda ZFS birimleri kullanılıyorsa, havuz `-f` seçeneğiyle bile dışa çıkarılamaz. ZFS'de hacimli bir havuzu dışa aktarmak için, öncelikle birimin üzerindeki çalışan işleticilerin ve göreletin tamamlanmış ve artık aktif olmadığından emin olun.

Bu komutun yerine getirilmesinden sonra aygıt havuz artık sistemden ayrılmış olur.

**NOT:** Dışa aktarma sırasında havuza ait bütün cihazlar mevcut değilse, cihazlar temiz bir şekilde dışa aktarılmış olarak tanımlanamaz. Bu durum potansiyel olarak bazı havuz hatalarına sebep olabilir. Bu yüzden bütün diskleri aktarım işlemleri sırasında kontrol ediniz.

`zpool` tarafından kullanılan bütün aktif havuzları kaldırmak için ise `-a` parametresini kullanmamız gerekmektedir.

```text
~# zpool list
NAME      SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank      3.75G   141K  3.75G        -         -     0%     0%  1.00x    ONLINE  -
tank2     3.75G   141K  3.75G        -         -     0%     0%  1.00x    ONLINE  -
filetank  3.75G   141K  3.75G        -         -     0%     0%  1.00x    ONLINE  -
hometank  3.75G   141K  3.75G        -         -     0%     0%  1.00x    ONLINE  -
serv      3.75G   141K  3.75G        -         -     0%     0%  1.00x    ONLINE  -

~# zpool export -a
~# zpool list
no pools available
```

### ZFS Havuzunu İçe Aktarmak

Havuz sistemden kaldırıldıktan sonra \(açık bir dışa aktarım yoluyla veya cihazları zorla kaldırarak\), cihazları başka bir sisteme ekleyebilirsiniz. ZFS, yalnızca `mandatory` olan bazı cihazların mevcut olduğu bazı durumlarda içe aktarma yapabilir, ancak başarılı bir havuz aktarımı, cihazların genel sağlığına bağlıdır. Ancak, başka bir sistem tarafından bir depolama ağı üzerinden kullanımda olan bir havuzu içe aktarmak, her iki sistem de aynı depolama alanına yazmaya çalıştığından veri bozulmasına ve paniğe neden olabilir.

Ek olarak, cihazların aynı cihaz adı altında bağlanması gerekmez. ZFS, taşınan veya yeniden adlandırılan cihazları algılar ve yapılandırmayı uygun şekilde ayarlar. Kullanılabilir havuzları keşfetmek için, zpool içe aktarma komutunu seçenek olmadan çalıştırın. Örneğin:

```text
~# zpool import

 pool: tank
    state: ONLINE
    action: The pool can be imported using its name or numeric identifier.
    config:

        tank        ONLINE
          mirror-0  ONLINE
            sdb     ONLINE
            sdc     ONLINE
```

Bu komut sadece içe aktarılacak aygıların listesini bize verecektir. Bu aşamadan sonra **`zpool import tank`** diyerek içeri aktarımı tamamlayabiliriz. Bu komutun avantajı içe aktarım yapmadan önce aktarımda yaşanabilecek öngörülebilir hataları bize vermesidir.

Havuzdaki bazı cihazlar mevcut değilse ancak kullanılabilir bir havuz sağlamak için yeterli yedek veri mevcutsa, havuz BOZULMUŞ durumda görünür. Örneğin:

```text
~# zpool import
    pool: tank
    state: DEGRADED
    status: One or more devices are missing from the system.
    action: The pool can be imported despite missing or damaged devices.  The
        fault tolerance of the pool may be compromised if imported. 

    config:

        NAME        STATE     READ WRITE CKSUM
        tank        DEGRADED     0     0     0
          mirror-0  DEGRADED     0     0     0
            sdc     UNAVAIL      0     0     0  cannot open
            sdd     ONLINE       0     0     0
```

Bağlama yapılacağı zaman yedekleri kullanarak asıl havuzu inşaa edebilir, yani import işlemi için hala izin verebilir. Ancak diskleri kaybedebileceğimiz bir hatada havuz bağlanamaz durumda olabilir.

```text
~# zpool import
  pool: tank
  state: FAULTED
  action: The pool cannot be imported. Attach the missing
          devices and try again.
  config:
        raidz1-0       FAULTED
          sdc          ONLINE
          sdd          FAULTED
          sde          ONLINE
          sdf          FAULTED
```

Bu durumda import işlemi için izin verilmeyecektir. `FAULTED` olarak işaretlenmiş olan aygıt havuzu bağlanamayacaktır.

## ZFS Disk Havuzunu Yükseltmek

OpenZFS yakında zfs-2.0.0 ismindeki bir sürüme geçmeyi planlıyor. Her yeni ana sürümün getirdiği bazı özellikler var. Eğer eski sürümlerden kalma ZFS depolama havuzlarınız varsa, mevcut sürümdeki havuz özelliklerinden yararlanmak için havuzlarınızı `zpool upgrade` komutuyla yükseltebilirsiniz. Ek olarak, `zpool status` komutu, havuzlarınız eski sürümleri çalıştırdığında sizi bilgilendirmek için **action** çıktısı verecektir. Örneğin:

```text
~# zpool status
   pool: tank
   state: ONLINE
   status: The pool is formatted using an older on-disk format.  The pool can
           still be used, but some features are unavailable.
   action: Upgrade the pool using 'zpool upgrade'.  Once this is done, the
           pool will no longer be accessible on older software versions.
   scrub: none requested

   config:
        NAME        STATE     READ WRITE CKSUM
        tank        ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0
errors: No known data errors
```

Belirli bir sürüm ve desteklenen sürümler hakkında ek bilgileri tanımlamak için aşağıdaki sözdizimini kullanabilirsiniz:

```text
~# zpool upgrade -v
This system supports ZFS pool feature flags.

The following legacy versions are also supported:

VER  DESCRIPTION
---  --------------------------------------------------------
 1   Initial ZFS version
 2   Ditto blocks (replicated metadata)
 3   Hot spares and double parity RAID-Z
 4   zpool history
 5   Compression using the gzip algorithm
 6   bootfs pool property
 7   Separate intent log devices
 8   Delegated administration
 9   refquota and refreservation properties
 10  Cache devices
 11  Improved scrub performance
 12  Snapshot properties
 13  snapused property
 14  passthrough-x aclinherit
 15  user/group space accounting
 16  stmf property support
 17  Triple-parity RAID-Z
 18  Snapshot user holds
 19  Log device removal
 20  Compression using zle (zero-length encoding)
 21  Deduplication
 22  Received properties
 23  Slim ZIL
 24  System attributes
 25  Improved scrub stats
 26  Improved snapshot deletion performance
 27  Improved snapshot creation performance
 28  Multiple vdev replacements

For more information on a particular version, including supported releases,
see the ZFS Administration Guide.
```

Ardından, tüm havuzlarınızı yükseltmek için zpool yükseltme komutunu çalıştırabilirsiniz. Örneğin:

```text
~# zpool upgrade -a
```

**Not**: Havuzunuzu daha sonraki bir ZFS sürümüne yükseltirseniz, havuza daha eski bir ZFS sürümünü çalıştıran bir sistemde erişilemez.

## ZFS Havuzunda Disk Kontrolleri

Geleneksel disk yönetim sistemlerinde veri kontrolleri yardımcı programlar aracılığı ile yapılır. Windows'ta otomatik bazı araçlar yardımı ile yapılırken; GNU/Linux'ta, diskteki veri bütünlüğünü doğrulamak, bir dizi dosya sistemi kontrol pogramları aracılığı ile yapılmaktadır. Bu işlem, `fsck` araç kiti aracılığıyla yapılır.

Bununla birlikte, `fsck` sisteminin birkaç büyük dezavantajı vardır. İlk olarak, veri hatalarını düzeltmeyi düşünüyorsanız diski çevrimdışı olarak kontrol etmeniz gerekir. Bu, diskin kontrol ve düzeltme esnasında kullanılamaz halde olması demektir. Bu nedenle, `fsck`'den önce disklerinizi ayırmak için `umount` komutunu kullanmalısınız.

Bu örnek vermek gerekirse, kök dizinin bağlı bulunduğu disk bozulması durumunda, kontrolü yapmak için bir CDROM veya USB bellek gibi başka bir ortamdan bir sistem önyüklemesi yapmamız gerekmektedir. Disklerin boyutuna bağlı olarak saatler sürebilen bu işlem, büyük sistemler için saatlerce sürecek sistem kesintilerine sebep olabilir.

İkincisi ise şudur ki, `ext3` veya `ext4` gibi geleneksel dosya sistemleri, `LVM` veya `RAID` gibi üst düzey veri yapıları ve disk sistemleri ile bağlantı katmanı yoktur. Örneğin RAID dizisi için bir diskte bozuk bir bloğunuz olabilir, ancak başka bir diskte herhangi bir sorun bulunmayabilir ancak, Linux'taki disk yazılımı RAID'in bağlanması ve kontrol edilmesi esnasında hangisinde veri kaybı olduğu hakkında hiçbir fikri yoktur. `ext3` veya `ext4` açısından, sorunsuz diskten bloklar okunduğunda düzgün veri alınırken bozuk bloğu içeren diskten okurken, ister istemez, bozuk veri alınır. Verilerin hangi diskten çekileceği ve bozulmanın düzeltilmesi üzerinde herhangi bir kontrol yapısı da bulunmamaktadır. Bu hatalar **"sessiz veri hataları"** olarak bilinir ve standart GNU/Linux dosya sistemi yığınıyla bu konuda yapabileceğiniz hiçbir şey yoktur.

ZFS'nin belki de en iyi yanı bu sessiz hataları kendi içerisinde halletmektedir. Bu işlemi herhangi bir hizmet kesintisi olmadan sağlayabilmektedir. Bu işlemi yaparken de tüm diskler uygun şekilde yeniden yapılandırılabilmekte ve tamamen kullanıcı müdahalesine gerek kalmadan bu işlemler yapılmaktadır.

Linux'ta ZFS ile sessiz veri hatalarının tespiti ve düzeltilmesi, diskleri temizleyerek yapılır. Bu, ECC RAM'leri ile teknik olarak benzerdir; burada ECC DIMM'de bir hata varsa, sistem otomatik olarak iyi verileri içeren başka bir kayıt bulabilir ve onu kötü kaydı düzeltmek için kullanabilir.

Bu tip sessiz veri düzeltmeleri bir süredir pek çok donanımda kullanılmaktadır. Ayrıca, bu yöntem sayesinde bir sistemde ECC RAM'i kesinti olmadan temizleyebildiğiniz gibi, disklerinizi kesinti olmadan da temizleyebilmelisiniz. Ancak geleneksel dosya sistemlerinde bu tarz bir yapı bulunmamaktadır. Bunun temelindeki sebep ise hız gerektiren durumlarda disk üzerinde bu şekilde işlem yapan bir yapı istenmemesidir. `BTRFS` gibi sonradan geliştirilen disk yönetim sistemlerinde her ne kadar bu yer almaya başladıysa bile hala aktif olarak pek çok geleneksel sistemde bulunmamaktadır.

ZFS bu işlemleri kolaylaştırmak için temizleme ve yeniden yayımlama işlemi yapan iki adet altyapı ile gelmektedir.

ZFS, havuzunuzda bir temizleme işlemi yaparken, depolama havuzundaki her bloğun bilinen sağlama toplamına göre kontrol eder. Yukarıdan aşağıya her blok, varsayılan olarak uygun bir algoritma kullanılarak sağlama toplamı alınır. Daha öncesinde belirttiğim gibi bu sağlama topları, 256 bitlik bir algoritma olan **"fletcher4"** algoritması ile alınmaktadır. Diğer dosya sistemlerinin aksinde **fletcher4** kullanılmasının sebebi, SHA-256 sağlama toplamının hesaplanması fletcher4'ten daha maliyetli olmasındandır. Ancak ZFS yine de SHA256 sistemini kendi içerisinde de getirmektedir ve önerilmese de, SHA-256 algoritması blok kontrol yapısında kullanılabilir.

ZFS, disklerdeki verileri kurtarmak ve düzeltmek için yine bu ikili sağlama toplamları kullanılmaktadır. Burada önemli olan nokta herhangi bir yedek ya da aynalama diskine sahip bir havuza sahip olup olmamızdır. Bu şekilde bir yedekli yapımız bulunuyorsa ZFS'de veri kendi kendine düzeltilebilmektedir. Bu işleme `self-healing` yani kendini iyileştirme işlemi denmektedir.

### ZFS'de Sorun Temizleme İşlemi: Scrub

ZFS depolama havuzlarını temizlemek, otomatik olarak gerçekleşen bir işlem değildir. Bu işlemi manuel olarak tetiklememiz gerekir. Verileri bu şekilde kontrol etmek ve temizlemek için bize gerekecek süre disk türüne bağlı olarak değişmekle beraber sıklıkla yapılması gerekmektedir.

Bu temizlik işlemi `zpool scrub` komutu ile yapılmaktadır.

```text
~# zpool scrub tank
~# zpool status
  pool: tank
  state: ONLINE
  scan: scrub repaired 0B in 00:00:01 with 0 errors on Mon Mar  1 11:02:01 2021
  config:

    NAME           STATE     READ WRITE CKSUM
    tank           ONLINE       0     0     0
      mirror-0     ONLINE       0     0     0
        sdc        ONLINE       0     0     0
        sdd        ONLINE       0     0     0
    logs    
        sde        ONLINE       0     0     0
        sdf        ONLINE       0     0     0

errors: No known data errors
```

Gördüğümüz gibi `scan:` olarak görülen yeni bir durum tagı açıldı. Ve burada ilerlememizi kolayca görebiliriz. Devam eden bir temizlik işlemini de aşağıdaki komutla durdurabiliriz.

```text
~# zpool scrub -s tank
```

### Resilvering Kavramı ve Bozuk Disklerin Tasfiye İşlemi

Verileri yeniden yayımlamak `(resilvering)`, verileri yeni diskte diziye yeniden oluşturmak veya yeniden eşitlemek demektir. Ancak, donanım bazlı RAID denetleyicileri ve diğer geleneksel RAID uygulamaları ile hangi blokların gerçekte sorunsuz olduğu ve hangilerinin olmadığı arasında bir ayrım yapamazlar. Bu nedenle, yeniden oluşturma diskin başlangıcında başlar ve diskin sonuna ulaşıncaya kadar durdurulamaz.

ZFS, RAIDZ yapısını ve dosya sistemi meta verilerini bildiğinden, verileri yeniden oluşturma konusunda daha akıllıca bir yol izler. ZFS, veri bloklarının saklanmadığı boş diskte boşa zaman harcamak yerine, sadece veri blokları bulunan canlı bloklarla işlem yapmaktadır. Bu, depolama havuzu kısmen doluysa önemli ölçüde zaman tasarrufu sağlayabilir. Örneğin havuzun yalnızca %25'i doluysa, bu, sürücülerin yalnızca %25'inde işlem yapmak ve bir diski düzeltmek için kullanılacak zamanın %25'inin bütün diski düzeltmek için yeteceği anlamına gelmektedir.

Ne yazık ki zamanla disk havuzundaki diskler ölecek ve değiştirilmeleri gerekecek. Depolama havuzunuzda bu diskleri karşılayabilecek fazlalık diskler olması ve bazı arızaları karşılayabilmeniz koşuluyla \(örneğin daha öncesinde belirttiğim gibi RAIDZ1 için 3 diskte sadece 1 disk arızası yaşanması koşulu gibi\), havuz "DEGRADED" modunda olsa bile uygulamalara veri gönderip alabilir ve diskleri onarabilirsiniz.

Sistem çalışır durumdayken disk değiştirme lüksüne sahipseniz, diski kesinti olmadan değiştirebilirsiniz, değilse, yine de ölü diski tanımlamanız ve değiştirmeniz gerekecektir. Havuzunuzda çok sayıda disk varsa, bu bir angarya olabilir, çünkü takılı disklerin hepsini tanıyıp elle değiştirmeniz gerekmekte. Hepsinin seri numarasını "hdparm" adlı bir yardımcı programla tespit etseniz de diskleri söküp yerine yenisini takmak için bütün disk havuzundaki diskleri tek tek bulup doğru diski tespi etmemiz gerekmektedir. Örneğin, havuz içerisinde kullandığımız 5 tane disk olsun diyelim, bunlar aşağıdaki gibi id'lenmiş olsun.

```text
/dev/sdc: WD-WX21A689CJ0H
/dev/sdd: HSGT-HG23F829AJ9F
/dev/sde: K34X52ADC2RX
/dev/sdf: /dev/sde: No such file or directory
/dev/sdg: AX231DEF
```

Görünüşe göre `/dev/sde` diskinin seri numarasına ulaşılamadı yani bu disk benim için ölü disk ve havuzda görülemiyor.

Diski çıkarıp, yenisiyle değiştirelim ve bunu başka bir diskle değiştirelim. Eklediğimiz disk `/dev/sdh` üzerine bağlanmış olsun. Bu diskleri `zpool replace` komutu ile değiştirelim.

```text
~# zpool replace tank sde sdh
~# zpool status tank
   pool: tank
   state: ONLINE
   status: One or more devices is currently being resilvered.  The pool will
           continue to function, possibly in a degraded state.
   action: Wait for the resilver to complete.
   scrub: resilver in progress for 0h2m, 16.43% done, 0h13m to go
   config:

        NAME          STATE       READ WRITE CKSUM
        tank          DEGRADED       0     0     0
          mirror-0    DEGRADED       0     0     0
            sdc       ONLINE         0     0     0
            sdd       ONLINE         0     0     0
          mirror-1    ONLINE         0     0     0
            replacing DEGRADED       0     0     0
            sdh       ONLINE         0     0     0
            sdf       ONLINE         0     0     0
          mirror-2    ONLINE         0     0     0
            sdg       ONLINE         0     0
```

### ZFS Havuzunun Sorunlarını Tespit Etmek

ZFS aygıt havuzunun durumunu tespit etmek için `zpool status` komutunu kullandığımızı hatırlıyor olmalıyız. Bu komuta verilecek `-x` parametresi aygıt havuzumuza dair detaylı durumu bize aktaracaktır.

```text
~# zpool status -x
all pools are healthy
```

## ZFS Havuzunun Detaylı Öznitelikleri

`ext4` ve GNU/Linux'taki birçok dosya sisteminde çeşitli bayraklar yardımı ile disk alanının davranışlarını ayarlayabiliriz. Disk bayraları, varsayılan montaj seçeneklerini ve diğer ayarları ayarlamak gibi işlere yaramaktadır.

ZFS'de de durum farklı değil ancak ZFS'deki bayrak yapısı çok daha ayrıntılıdır. Bu özellikler, hem havuz hem de içerdiği veri kümeleri için her tür değişkeni değiştirmemize izin verir. Böylece, dosya sistemini kendi zevkimize veya ihtiyaçlarımıza göre "ayarlayabiliriz". Ancak, bazı öntanımlı özellikler vardır ki bunlar maalesef ayarlanamaz. Bazı özellikler ise salt okunurdur. Ancak, her bir özelliğin ne olduğunu ve havuzu nasıl etkilediğini tanımlayacağız.

Bu bölümde sadece ZFS havuzunun özelliklerinden bahsedeceğiz. Dosya hiyerarşisinin de kendi özellikleri vardır ancak bunları hiyerarşi altında inceleyeceğiz.

* **allocated**: Tüm ZFS veri kümeleri tarafından havuza kaydedilen veri miktarıdır. Bu ayar salt okunurdur.
* **altroot**: Alternatif bir kök dizini tanımlar. Ayarlanırsa, bu dizin havuzu ayarlanan bağlama noktasının başına ekler. Bu özellik bilinmeyen bir havuzu incelerken, bağlama noktalarına güvenilemiyorsa, bağlama noktasında başka bir dizin veya havuz bulunuyorsa veya tipik yolların geçerli olmadığı alternatif bir önyükleme ortamına ZFS diski bağlanıyorsa kullanılarak alternatif bir diske bağlama sağlanır. "cachefile = none" olarak ayarlanmış havuzlarda, bu geçersiz kılınabilir.
* **ashift**: Yalnızca havuz oluşturma sırasında ayarlanabilir, sonrasında salt okunur olarak işaretlenir. Havuz sektör boyutu 2'nin üstelleri olarak ayarlanır yani bu şu demek. Varsayılan değer olan 9 için, 2^9 = 512 bize boyut sınırını oluşturur ve I/O işlemleri, belirtilen boyut sınırlarına göre hizalanır. Standart sektör boyutu, işletim sistemi yardımcı programları için verileri okumak ve yazmak için kullanır. Örneğin, 4 KiB sınırına sahip gelişmiş formatlı sürücüler için, değer "ashift = 12" olarak 2 ^ 12 = 4096 olarak ayarlanmalıdır ki bu sayede bloklar diskin donanımına uygun olarak ayarlanmış olur.
* **autoexpand**: Havuzunuzdaki ilk sürücüyü değiştirmeden önce ayarlanmalıdır. Temel **LUN** büyüdüğünde otomatik havuz genişletmeyi kontrol eder. Varsayılan "kapalı" **`(off)`**dır. Havuzdaki tüm sürücüler daha büyük sürücülerle değiştirildikten sonra, havuz otomatik olarak yeni boyuta büyür. Bu ayar için değerler **`on`** veya **`off`**'dur.
* **autoreplace**:  Havuzunuzdaki "yedek" bir **VDEV**'nin otomatik cihaz değişimini kontrol eder.  Varsayılan "kapalı" **`(off)`**dır. Bu nedenle, cihaz değişimi "**`zpool replace`**" komutu kullanılarak manuel olarak başlatılmalıdır. Bu ayar için değerler **`on`** veya **`off`**'dur.
* **bootfs**: Havuzdaki önyüklenebilir ZFS diskleri için veri kümesini tanımlayan salt okunur ayardır. Bu genellikle kernel tarafından denetlenen bir önyükleme programı için ayarlanmaktadır.
* **cachefile**: Havuz yapılandırmasının önbelleğe alındığı yeri kontrol eder. Bir sistemdeki bir `zpool`'u içe aktarırken, ZFS disklerdeki meta verileri kullanarak sürücü geometrisini ve havuz dağılımı algılayabilir. Ancak, bazı kümeleme ortamlarında, otomatik olarak içe aktarılmayacak havuzlar için önbellek dosyasının farklı bir konumda depolanması gerekebilir. Bu değer bu konumu belirler, önbellek dosyası bir konuma ayarlanabilir, ancak çoğu ZFS kurulumu için önerilen "**`/etc/zfs/zpool.cache`**" varsayılan konumu olmalıdır.
* **capacity**: Kullanılan havuz alanı yüzdesini tanımlayan salt okunur değerdir. Havuz genişledikçe otomatik olarak ayarlanır.
* **comment**: Havuz hatalı olsa bile kullanılabilen bir parametredir. 32'den fazla yazdırılabilir ASCII karakterinden oluşan bir metin dizesi yazmayı sağlar. Bu ayarı kullanarak bir havuz hakkında ek bilgi yazabilir ve gelecekte bu bilgileri kullanarak havuzu düzenleyebilirsiniz.
* **dedupditto**: Bir blok tekilleştirme eşiği ayarlar ve tekilleştirilmiş bir bloğun referans sayısı eşiğin üzerine çıkarsa, bloğun bir kopyası otomatik olarak saklanır. Varsayılan değer 0'dır. Herhangi bir pozitif sayı olabilir.
* **dedupratio**: Bir havuz için belirtilen salt okunur tekilleştirme oranı, çarpan olarak ifade edilir
* **delegation**: Ayrıcalıklı olmayan bir kullanıcıya veri kümesi için tanımlanan erişim izinlerinin verilip verilmediğini denetler. Ayar bir boolean'dır, bu ayar için değerler **`on`** veya **`off`**'dur ve varsayılan değer **`on`**.
* **expandsize**:  Havuzun toplam kapasitesini artırmak için kullanılabilecek havuz veya cihazdaki kullanılmamış alan miktarı. Kullanılmamış alan, EFI etiketli bir vdev üzerindeki çevrimiçi duruma getirilmemiş herhangi bir alandan oluşur \(yani **`zpool online -e`**\). Bu boşluk, bir **LUN** dinamik olarak genişletildiğinde oluşur.
* **failmode**: Katastrofik havuz arızası durumunda sistem davranışını kontrol eder. Bu durum, tipik olarak, temeldeki depolama cihazlarına bağlantı kaybının veya havuz içindeki tüm cihazların arızalanmasının bir sonucudur. Böyle bir olayın davranışı şu şekilde belirlenir:
  * **wait**: Cihaz bağlantısı kurtarılıncaya ve hatalar giderilene kadar tüm I/O erişimini engeller. Bu, varsayılan davranıştır.
  * **continue**: EIO'yu yeni yazma I/O isteklerine döndürür, ancak kalan sağlıklı cihazlardan herhangi birine okumaya izin verir. Henüz diske işlenmemiş yazma istekleri engellenecektir.
  * **panic**: Konsola bir mesaj yazdırır ve bir sistem kilitlenme dökümü oluşturur.
* **feature@xxx**: OpenZFS'nin gelecekte eklenecek ve şu an beta olarak getirilen özellikleri tanımlayacaktır.
* **free**: Havuzdaki ayrılmamış blokların sayısını tanımlayan salt okunur değer.
* **guid**: Havuz için benzersiz tanımlayıcıyı tanımlayan salt okunur özellik. Ext4 dosya sistemleri için UUID dizesine benzer bir özelliğe sahiptir, bu guid ile bağlama ve ayrılmaya izin verir.
* **health**: Havuzun mevcut durumunu tanımlayan salt okunur özelliktir; `status` komutunda buna ait detaylar paylaşılmıştır.
* **listsnapshots**: Bu havuzla ilişkili anlık görüntü bilgilerinin "**`zfs list`**" komutuyla görüntülenip görüntülenmeyeceğini denetler. Bu özellik devre dışı bırakılırsa, anlık görüntü bilgileri "**`zfs list -t snapshot`**" komutuyla görüntülenebilir. Bu ayar için değerler **`on`** veya **`off`**'dur ve varsayılan değer **`off`**'dur.
* **readonly**: Bu ayar için değerler **`on`** veya **`off`**'dur ve varsayılan değer **`off`**'dur. Yazmaları ve veri bozulmalarını önlemek için havuzu salt okunur moda ayarlamayı denetler. 
* **version**: Havuzun geçerli disk üzerindeki sürümünü tanımlayan yazılabilir ayardır. "**`zpool upgrade -v`**" komutunun çıktısına kadar herhangi bir değer olabilir. Bu özellik, geriye dönük uyumluluk için belirli bir sürüm gerektiğinde kullanılabilir.

### ZFS'de Parametreleri Ayarlamak

Bundan daha öncesinde bahsetmiştim. `zpool get` ve `zpool set` sayesinde parametreleri görebiliriz. Bütün parametrelerini görüntülemek için `zpool get all havuz_adi` komutu kullanılır.

```text
~# zpool get all tank
NAME  PROPERTY                       VALUE                          SOURCE
tank  size                           3.75G                          -
tank  capacity                       0%                             -
tank  altroot                        -                              default
tank  health                         ONLINE                         -
tank  guid                           16008357831728779071           -
tank  version                        -                              default
tank  bootfs                         -                              default
tank  delegation                     on                             default
tank  autoreplace                    off                            default
tank  cachefile                      -                              default
tank  failmode                       wait                           default
tank  listsnapshots                  off                            default
tank  autoexpand                     off                            default
tank  dedupratio                     1.00x                          -
tank  free                           3.75G                          -
tank  allocated                      180K                           -
tank  readonly                       off                            -
tank  ashift                         0                              default
tank  comment                        -                              default
tank  expandsize                     -                              -
tank  freeing                        0                              -
tank  fragmentation                  0%                             -
tank  leaked                         0                              -
tank  multihost                      off                            default
tank  checkpoint                     -                              -
tank  load_guid                      2495128167517849836            -
tank  autotrim                       off                            default
tank  feature@async_destroy          enabled                        local
tank  feature@empty_bpobj            enabled                        local
tank  feature@lz4_compress           active                         local
tank  feature@multi_vdev_crash_dump  enabled                        local
tank  feature@spacemap_histogram     active                         local
tank  feature@enabled_txg            active                         local
tank  feature@hole_birth             active                         local
tank  feature@extensible_dataset     active                         local
tank  feature@embedded_data          active                         local
tank  feature@bookmarks              enabled                        local
tank  feature@filesystem_limits      enabled                        local
tank  feature@large_blocks           enabled                        local
tank  feature@large_dnode            enabled                        local
tank  feature@sha512                 enabled                        local
tank  feature@skein                  enabled                        local
tank  feature@edonr                  enabled                        local
tank  feature@userobj_accounting     active                         local
tank  feature@encryption             enabled                        local
tank  feature@project_quota          active                         local
tank  feature@device_removal         enabled                        local
tank  feature@obsolete_counts        enabled                        local
tank  feature@zpool_checkpoint       enabled                        local
tank  feature@spacemap_v2            active                         local
tank  feature@allocation_classes     enabled                        local
tank  feature@resilver_defer         enabled                        local
tank  feature@bookmark_v2            enabled                        local
tank  feature@redaction_bookmarks    enabled                        local
tank  feature@redacted_datasets      enabled                        local
tank  feature@bookmark_written       enabled                        local
tank  feature@log_spacemap           active                         local
tank  feature@livelist               enabled                        local
tank  feature@device_rebuild         enabled                        local
tank  feature@zstd_compress          enabled                        local
```


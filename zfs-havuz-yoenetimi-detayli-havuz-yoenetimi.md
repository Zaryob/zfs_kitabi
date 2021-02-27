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

## ZFS'de Havuz Aktarım İşlemleri

### ZFS Havuzunu Dışa Aktarmak

### ZFS Havuzunu İçe Aktarmak

## ZFS Havuzunda Veri Kontrolleri

### Scrub

### Resilvering

## ZFS Havuzunda Snapshot İşlemleri

## ZFS Havuzunun Detaylı Öznitelikleri


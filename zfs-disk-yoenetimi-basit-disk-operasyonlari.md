---
description: ZFS'nin havuz işlemleri ve vdev işlemleri ile başlayan ufak bir ısınma turu
---

# ZFS Disk Yönetimi - Basit Disk Operasyonları

ZFS'de disk alanları ve dosya sistemleri, sanal disk havuzunun altında oluşturulan katmandır. Farklı ZFS diskler ve farklı bölümleri aynı havuz içerisine kaydedilerek bu havuza ekleme ve çıkarma işlemi yapılabilir. ZFS dosya sistemleri, herhangi bir temel disk alanı ayırmanıza veya biçimlendirmenize gerek kalmadan dinamik olarak oluşturulabilir ve yok edilebilir. Dosya sistemleri çok hafif olduğundan ve ZFS'de merkezi yönetim noktası olduklarından, bunlar hem oldukça esnek hem de manüplasyonu tehlikeli yapılardır. Yani yaptığımız düzenlemeler kesinlikle çok dikkatlice yapılmalıdır. 

Başlamak için, ZFS bunları dahili olarak kapsamlı bir şekilde kullandığından, sanal cihazları veya **VDEV**'leri anlamamız gerekir. Temel olarak, bir veya daha fazla fiziksel cihazı temsil eden bir meta cihazımız var. Linux yazılım RAID'inde, 4 diskten oluşan RAID-5 dizisini temsil eden bir "/dev/md0" aygıtınız olabilir. Bu durumda, "/dev/md0" sizin "**VDEV**" aygıtınız olacaktır.

ZFS'de yedi tür VDEV vardır:

* **disk \(varsayılan\):** Sisteminizdeki fiziksel sabit sürücüler. 
* **dosya \(file\):** Önceden ayrılmış dosyaların / görüntülerin mutlak yolu. 
* **ayna \(mirror\):** Standart yazılım RAID-1 aynası
* **raidz1/2/3:** Standart olmayan dağıtılmış eşlik tabanlı yazılım RAID seviyeleri. 
* **yedek \(spare\):** ZFS RAID için "etkin yedek" olarak işaretlenmiş sabit sürücüler. 
* **önbellek \(cache\):** Seviye 2 uyarlanabilir okuma önbelleği \(L2ARC\) için kullanılan cihaz. 
* **günlük \(log\):** "ZFS Amaç Günlüğü" veya ZIL adı verilen ayrı bir günlük \(SLOG\).



ZFS dosya sistemleri, `zfs` komutu kullanılarak yönetilir. ZFS komutu, dosya sistemlerinde belirli işlemleri gerçekleştiren bir dizi alt komut sağlar. Bu alt komutlar basit disk işlemleri \(disk alanı ekleme, çıkarma, kontrol etme\) haricinde daha öncesinde bahsettiğim aynalama \(`mirror`\) ve yedekleme \(`spare`\) işlerini de sağlar.

ZFS dosya sistemlerini oluşturmak ve yönetmek için ilk yapmamız gereken işlem bir ZFS disk havuzu oluşturmak. İlk adımda bu havuzları nasıl oluşturup nasıl manüple edeceğimizi göreceğiz.

Bu aşamada temel işlemleri ikiye ayıracağım. ZFS Disk yönetim sistemi için bir disk havuzu oluşturmak zorundayız. Bu bizim birinci kısmımızı oluşturacak. İkincisi ise bu havuzlar içerisinde ZFS disk bölümleri oluşturabilmeyi ve üzerinde işlemler yapmayı öğreneceğiz.

## ZFS Disk Havuzunda Temel İşlemler

ZFS yönetimi, basitlik göz önünde bulundurularak tasarlanmıştır. Tasarım hedefleri arasında, kullanılabilir bir dosya sistemi oluşturmak için gereken komutların sayısını azaltmaktır. Örneğin, yeni bir havuz oluşturduğunuzda, otomatik olarak yeni bir ZFS dosya sistemi oluşturulur ve bağlanır. Bağlama konumu \(siz aksini belirtmedikçe\) sisteminizin kök dizinidir.

ZFS'de disk havuzu işlemleri `zpool` komutu ile yönetilir.

### ZFS Disk Havuzu Oluşturmak

İlk olarak `zpool` komutu ile elimizdeki disk havuzu listesini görelim.

```text
~# zpool list
no pools available
```

Bu komutun çıktısı herhangi bir disk havuzu oluşturulmadığını bize bildiriyor. ZFS'de çeşitli dosya sistemleri oluşturabilme seçeneklerimiz var. Bunlardan ilki gerçek donanıma bir ZFS dosya sistemi oluşturmak.

Bu aşama için `/dev/sdb` yoluna bağlı diski kullanacağım.

```text
~# zpool create tank /dev/sdb1
```

Bu komut hengi bir çıktı vermeyecek. `zpool create` komutu bizim bir disk havuzu oluşturmamızı sağlar. `tank` bizim ZFS havuzumunuzun adıdır \(diğer ZFS dökümanlarında hep tank olarak koymuşlar, sanırım bu bir gelenek ve bozmak istemedim\). `/dev/sdb1` ise `/dev/sdb`'ye bağlı diskin ilk bölümünü işaret ediyor. Bu komutun tamamlanması ile daha öncesinde belirttiğim gibi disk alanımız kök dizine bağlanacaktır.

```text
~# ls -ali / |grep tank
     34 drwxr-xr-x.   2 root root     2 Şub 24 18:44 tank
```

`/dev/sdb1` üzerine bu disk havuzunu oluşturmamın sebebi ZFS'yi tek bir disk alanında kullanmak. Eğer ki diskinizin içerisinde birden fazla bölüm varsa ve bu bölümleri kaybemek istemiyorsanız kesinlikle bu şekilde bölüm belirtmenizi tavsiye ederim. Örneğin ZFS on Linux kurulumu yapacaksanız bu şekilde bir havuz oluşturmak daha mantıklı olacaktır çünkü aynı diskte, `boot` ve `EFI` için ayrı disk alanlarına ihtiyaç duyabilirsiniz.

Ancak tüm bir diski ZFS olarak kullanmak istiyorsanız şunu yapmanız gerekmekte.

```text
~# zpool create tank /dev/sdb
```

Şimdi bu işlemlerin sonucunda bir disk havuzu oluşturduğumuzu varsayalım. En başta yapıtığımız gibi zpool listemizi kontrol edelim. Bunu da `zpool list` komutu ile yapabiliyoruz.

```text
~# zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  14.5G   100K  14.5G        -         -     0%     0%  1.00x    ONLINE  -
```

Bu komutun çıktısında gördüğümüz gibi disk havuzunun toplam boyutu, kullanılan ve boş alan, disk sağlığı durumu ve ek bazı detaylar var. Bu komut sayesinde sisteminizde yer alan bütün ZFS havuzlarının listesini kolaylıkla görebilirsiniz.

###  Birden Fazla Diskle ZFS Disk Havuzu Oluşturmak

`zpool` ile aynı anda birden fazla diski de tek havuzda kullanabileceğimizden bahsetmiştim. Bunu ilk kurulum aşamasında yapabiliriz. Bunun için şu komutu vermemiz yeterlidir.

```text
~# zpool create tank /dev/sdb /dev/sdc
```

Bu komutla iki tane 16 GB'lık diski bir zpool içerisine eklemiş olduk. Bu işlem sonucunda disk alanımız şu şekilde görünecektir.

```text
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  29.0G   148K  29.0G        -         -     0%     0%  1.00x    ONLINE  -
```

Eklediğimiz yeni disk alanını, daha fazla disk alanına sahip olmak için kullanmak zorunda değiliz. Bunu daha öncesinde belirttiğim aynalama yani `mirror` işlemi için de kullanmak isteyebiliriz. Bu durumda bize `RAID` yuvalaması \(nesting\) yardımıza koşuyor. Bunu bir sonraki kısımda özellikle detaylandırarak anlatacağım ama şimdi bahsetmeden geçmemek istedim. Bu özellik sayesinde eklediğimiz diskleri gruplayarak ekleyebiliriz. Kimi diskleri günlükleme kimi diskleri geçici depolama ve önbellekleme için kullanabiliriz. Bu durumda disk alanını oluştururken şunu yapmamız yeterlidir.

```text
~# zpool create tank mirror /dev/sdb /dev/sdc
```

ZFS havuzumuza ait detayları görüntüleyelim şimdi. Bunun için de bir diğer komuta ihtiyacımız var. `zpool status` ile havuzumuza bağlı bütün diskleri ve durumlarını görebiliriz.

```text
~# zpool status
  pool: tank
  state: ONLINE
  config:

    NAME        STATE     READ WRITE CKSUM
    tank        ONLINE       0     0     0
      mirror-0  ONLINE       0     0     0
        sdb     ONLINE       0     0     0
        sdc     ONLINE       0     0     0

errors: No known data errors
```

Bu durumda `/dev/sdb` ve `/dev/sdc` disklerini aynalama için kullanmış olacağız.

Birden fazla aynalama grubu oluşturmak istediğimizde ise

```text
~# zpool create tank mirror /dev/sdb /dev/sdc mirror /dev/sdd /dev/sdf
```

```text
  pool: tank
  state: ONLINE
  scan: none requested
  config:

    NAME        STATE     READ WRITE CKSUM
    tank        ONLINE       0     0     0
      mirror-0  ONLINE       0     0     0
        sdb     ONLINE       0     0     0
        sdc     ONLINE       0     0     0
      mirror-1  ONLINE       0     0     0
        sdd     ONLINE       0     0     0
        sdf     ONLINE       0     0     0
```

Bunlara ek olarak disk havuzunda birden fazla grup da ekleyebiliriz

```text
~# zpool create tank mirror /dev/sdb  /dev/sdc mirror /dev/sdd /dev/sde log mirror /dev/sdf /dev/sdg  cache /dev/sdh /dev/sdi
```

```text
  pool: tank
  state: ONLINE
  scan: none requested
  config:

    NAME            STATE     READ WRITE CKSUM
    tank            ONLINE       0     0     0
      mirror-0      ONLINE       0     0     0
        sdb         ONLINE       0     0     0
        sdc         ONLINE       0     0     0
      mirror-1      ONLINE       0     0     0
        sdd         ONLINE       0     0     0
        sde         ONLINE       0     0     0
    logs
      mirror-2      ONLINE       0     0     0
        sdf         ONLINE       0     0     0
        sdg         ONLINE       0     0     0
    cache
      sdh           ONLINE       0     0     0
      sdi           ONLINE       0     0     0
```

Gördüğünüz gibi tek bir zpool havuzu sayesinde birden fazla diske ait bütün alanı eşzamanlı olarak kullanabiliyoruz. Ayrıca farklı disk alanlarını aynı havuz içerisinde farklı amaçlarla da kullanabiliyoruz. Bu sayede birden fazla diske sahip olan sunucularda ayrı ayrı diskleri bölümlendirmek ve bağlamakla uğraşmak yerine bir seferde bunları tek bir disk alanı gibi kullanabiliyoruz ve bir tek konuma bağlayabiliyoruz.

### ZFS Disk Havuzuna Yeni Diskler Eklemek

Şimdi bir senaryo ile geleyim. Halihazırda `/dev/sdb` ve `/dev/sdc` disklerini eklediğimiz bir havuz var ve tüm depolama alanının dolmuş olduğunu varsayalım. Eğer yeterli boş disk slotumuz varsa ek diskler ekleyerek yeni boş alanlar oluşturabiliriz. Ama dur bir saniye. Bütün bu disk havuzunu sıfırlamamız mı lazım. Tabi ki hayır. Geleneksel disk yönetim sistemlerinin aksine ZFS'de bunu havuza eklemek için basit bir komut yeterlidir.

```text
~# zpool add tank /dev/sdd
```

Bu işlemle `/dev/sdd` diskini havuz içerisine kolayca bağlamış oluruz.

```text
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank    43G   182K  43.0G        -         -     0%     0%  1.00x    ONLINE  -
```

```text
pool: tank
state: ONLINE
config:

NAME        STATE     READ WRITE CKSUM
tank        ONLINE       0     0     0
    sdb       ONLINE       0     0     0
    sdc       ONLINE       0     0     0
    sdd       ONLINE       0     0     0
```

Aynı şekilde farklı bir türle ekleme de yapabiliriz. Aşağıdaki gibi görünen bir havuzumuz olduğunu varsayalım.

```text
pool: tank
state: ONLINE
config:

NAME              STATE     READ WRITE CKSUM
tank              ONLINE       0     0     0
    mirror-0      ONLINE       0     0     0
        sdb       ONLINE       0     0     0
        sdc       ONLINE       0     0     0
```

Şimdi de yeni bir mirror seti ekleyelim.

```text
~# zpool add tank mirror /dev/sdd /dev/sdf
```

```text
  pool: tank
  state: ONLINE
  scan: none requested
  config:

    NAME        STATE     READ WRITE CKSUM
    tank        ONLINE       0     0     0
      mirror-0  ONLINE       0     0     0
        sdb     ONLINE       0     0     0
        sdc     ONLINE       0     0     0
      mirror-1  ONLINE       0     0     0
        sdd     ONLINE       0     0     0
        sdf     ONLINE       0     0     0
```

### ZFS Disk Havuzlarının İçe Aktarılması

Diyelim ki bir başka bilgisayarda uğraştığımız bir ZFS havuzu var. Bu havuzu başka bir bilgisayara bağlamak için `zpool import` komutu kullanılır.

```
~# zpool import tank
```
Bu işlem bütün disklerin yapılandırması ve uygun şekilde bağlanması için biraz süre gerekecektir. Bu sürenin ardından uygun şekilde bağlanacak ve kök sisteme bu havuz bağlanacaktır.



### ZFS Disk Havuzundan Disk Çıkarmak

Şimdi yaptığımız işleri birazcık da geri sardıralım.

Diyelim ki bu disklerden bir tanesini çıkarıp yerine bir başka disk takmamız lazım. Veya artık bu kadar çok bir depolama alanına ihtiyacımız yok. Bu durumda bütün bir disk havuzuna zarar vermeden havuzdan diskleri çıkartabiliriz.

```text
~# zpool remove tank /dev/sdc
```

Bu komut diskin içerisindeki veriye bağlı olarak uzun sürecektir. Şöyle ki ZFS bu diski kaldırmadan önce bu diske ait olan veriyi elinden geldiğince kaybetmemeye çalışır. Bu sebeple veriyi diğer disklere kaydıracaktır. İşlem bittiğinde de `zpool status` komutumuzun çıktısında `sdc` yer almayacaktır.

```text
~# zpool status tank
  pool: tank
  state: ONLINE
  config:

    NAME        STATE     READ WRITE CKSUM
    tank        ONLINE       0     0     0
      sdb       ONLINE       0     0     0

errors: No known data errors
```

Direk diskleri tek tek çıkarabildiğimiz gibi önceki adımlarda oluşturduğumuz alt özelliklere sahip alt kısımları da birlikte çıkarabiliriz. Örneğin aşağıdaki disk alanı için

```text
pool: tank 
state: ONLINE 
scan: none requested 
config:
  NAME        STATE     READ WRITE CKSUM
  tank        ONLINE       0     0     0
    mirror-0  ONLINE       0     0     0
      sdb     ONLINE       0     0     0
      sdc     ONLINE       0     0     0
    mirror-1  ONLINE       0     0     0
      sdd     ONLINE       0     0     0
      sdf     ONLINE       0     0     0
```

Bu havuzdan `mirror-1` 'i çıkarabiliriz.

```text
~# zpool remove tank mirror-1
```

Bu durumda havuzumuz şöyle görünecektir.

```text
pool: tank 
state: ONLINE 
scan: none requested 
config:
  NAME        STATE     READ WRITE CKSUM
  tank        ONLINE       0     0     0
    mirror-0  ONLINE       0     0     0
      sdb     ONLINE       0     0     0
      sdc     ONLINE       0     0     0
```



### ZFS Disk Havuzunu Yok Etmek

Bütün bir disk havuzunu silmek için ise `zpool destroy` komutunu kullanabiliriz.

```text
~# zpool destroy tank
```

```text
~# zpool status
no pools available
```


## ZFS Disk Hiyerarşisinde Temel İşlemler

Verilerinizi depolamak için bir depolama havuzu oluşturduktan sonra, dosya sistemi hiyerarşinizi oluşturabilirsiniz. Hiyerarşiler, bilgiyi organize etmek için basit ama güçlü mekanizmalardır. Bir dosya sistemi kullanmış olanlara da çok aşinadırlar.

Aslında ZFS geleneksel disk yönetim sistemi kullananlar için biraz karmaşık bir yapıya sahip. Ancak basitleştirmek gerekirse ZFS, dosya sistemlerinin, her dosya sisteminin yalnızca tek bir ana öğeye sahip olduğu hiyerarşiler halinde düzenlenmesine izin verir. Hiyerarşi aslında UNIX kök dosya yapısına benzemektedir. Hiyerarşinin kökü her zaman havuz adıdır. ZFS, özellik mirasını destekleyerek bu hiyerarşiden yararlanır, böylece ortak özellikler, tüm dosya sistemleri ağaçlarında hızlı ve kolay bir şekilde ayarlanabilir.

### ZFS Dosya Sistemi Hiyerarşini Belirleme

ZFS dosya sistemleri, merkezi yönetim noktasıdır. Hafiftirler ve kolayca oluşturulabilirler. Kullanılacak iyi bir model, kullanıcı veya proje başına bir dosya sistemi oluşturmaktır, çünkü bu model özelliklerin, anlık görüntülerin ve yedeklemelerin kullanıcı başına veya proje bazında kontrol edilmesine izin verir.

ZFS, dosya sistemlerinin hiyerarşiler halinde düzenlenmesine izin verir, böylece benzer dosya sistemleri gruplanabilir. Bu model, özellikleri kontrol etmek ve dosya sistemlerini yönetmek için merkezi bir yönetim noktası sağlar. Benzer dosya sistemleri ortak bir isim altında oluşturulmalıdır.

Çoğu dosya sistemi özelliği, özellikler tarafından kontrol edilir. Bu özellikler, dosya sistemlerinin nereye monte edileceği, nasıl paylaşılacağı, sıkıştırma kullanıp kullanmadıkları ve herhangi bir kotanın geçerli olup olmadığı gibi çeşitli davranışları kontrol eder.

ZFS Dosya Sistemleri, NFS kullanılarak paylaşılabilir ve sıkıştırma etkinleştirilebilir. Ek olarak, kullanıcı veya projeler için için kotalar uygulanabilir.

ZFS dosya sistemi komutları, `zfs` komutu ile yönetilir.

### ZFS Dosya Sistemi Hiyerarşi Oluşturma

ZFS'de dosya sistemi `zfs create` komutu ile oluşturulur. Parametre olarak da ZFS disk havuzunu ve disk yolunu belirtmeniz yeterlidir.

```text
~# zfs create tank/home
```

Oluşturulan dosya sistemi otomatik olarak kök dizinde yer alan havuzumuzun bağlandığı klasör altına bağlanır \(yine siz aksini belirtmedikçe\).

Ayrıca ZFS'de recursive olarak dosya hiyerarşisi oluşturabiliriz.

```text
~# zfs create tank/home/zaryob
~# zfs create tank/home/sulo
```

### ZFS Dosya Sistemi Hiyerarşilerini Görüntülemek

ZFS'de `zfs list` komutu ile dosya sisteminin detaylarını görüntülemek için kullanılır.

```text
~# zfs list
NAME                   USED  AVAIL  REFER  MOUNTPOINT
tank                  100.0K  67.0G   19K   /tank
tank/home             18.0K   67.0G     6K  /tank/zfs
```

### ZFS Dosya Sistemi Hiyerarşinin Özelliklerini Belirtmek

ZFS'de `zfs set` komutu ile dosya sisteminin özelliklerini belirlemek için kullanılır. Aslında özellikleri tek tek açıklamak çok uzun sürecektir çünkü pek çok özellik var ve hepsini `zfs set --help` komutu ile görebilirsiniz.

Bir diğer yandan temel bazı özellikleri değiştirelim.

```text
~# zfs set mountpoint=/mnt/zfs/home tank/home
```

`mountpoint` parametresi belirttiğimiz dosya sisteminin belirttiğimiz klasöre bağlanmasını sağlar. Bu örnek için `tank/home` dosya sistemi `/mnt/zfs` yoluna bağlanacaktır.

```text
~# zfs set sharenfs=on tank/home
```

`sharenfs` parametresi belirtilen dosya sistemini NFS üzerinden paylaşılmasını sağlar.

```text
~# zfs set compression=on tank/home
```

`compression` parametresi ise dosya sistemini sıkıştırarak kaydetmeyi sağlar.

```text
~# zfs set quota=10G tank/home
```

`quota` parametresi ise dosya sistemine bir sanal bir kota verir. Bu sayede dosya sistemini limitleyebiliriz.

Bütün bu özellikleri ise `zfs get` komutu ile öğrenebiliriz.

```text
~# zfs get compression tank/home
NAME             PROPERTY       VALUE                      SOURCE
tank/home        compression    on                         local
```

### ZFS Dosya Sistemi Hiyerarşi Yeniden Adlandırma

Bir ZFS dosya sistemini yeniden adlandırmak için `zfs rename` komutu kullanılır.

```text
~# zfs rename tank/home/zaryob tank/home/zaryob_old
```

Bu bir hiyerarşiyi başka bir dizine taşımayı da sağlar.

```text
~# zfs rename tank/home/zaryob tank/zaryob/
```

### ZFS Dosya Sistemi Hiyerarşi Yok Etme

Bir ZFS dosya sistemini yok etmek için `zfs destroy` komutu kullanılır. İmha edilen dosya sistemi otomatik olarak ayrılır ve paylaştırılmaz.

```text
~# zfs destroy tank/home
```

İmha edilecek dosya sistemi meşgulse ve bağlantısı kesilemezse, `zfs destroy` komutu başarısız olur. Aktif bir dosya sistemini yok etmek için `-f` seçeneğini kullanın. Etkin dosya sistemlerini kaldırabileceği, paylaşımını kaldırabileceği ve yok ederek beklenmedik uygulama davranışına neden olabileceği için bu seçeneği dikkatli kullanın.

```text
~# zfs destroy -f tank/home
```

Eğer ki `tank/home` dosya sistemi alt başka sistemlere de sahipse bu durumda `-R` parametresi gerekmektedir.

```text
~# zfs destroy -R tank/home
```


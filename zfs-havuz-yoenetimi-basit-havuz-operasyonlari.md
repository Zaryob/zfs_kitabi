---
description: ZFS'nin havuz işlemleri ve vdev işlemleri ile başlayan ufak bir ısınma turu
---

# ZFS Havuz Yönetimi - Basit Havuz Operasyonları

ZFS yönetimi, basitlik göz önünde bulundurularak tasarlanmıştır. Tasarım hedefleri arasında, kullanılabilir bir dosya sistemi oluşturmak için gereken komutların sayısını azaltmaktır. Örneğin, yeni bir havuz oluşturduğunuzda, otomatik olarak yeni bir ZFS dosya sistemi oluşturulur ve bağlanır. Bağlama konumu \(siz aksini belirtmedikçe\) sisteminizin kök dizinidir.

ZFS'de aygıt havuzu işlemleri `zpool` komutu ile yönetilir.

## ZFS Aygıt Havuzu Oluşturmak

İlk olarak `zpool` komutu ile elimizdeki aygıt havuzu listesini görelim.

```text
~# zpool list
no pools available
```

Bu komutun çıktısı herhangi bir aygıt havuzu oluşturulmadığını bize bildiriyor. ZFS'de çeşitli dosya sistemleri oluşturabilme seçeneklerimiz var. Bunlardan ilki gerçek donanıma bir ZFS dosya sistemi oluşturmak.

Bu aşama için `/dev/sdb` yoluna bağlı diski kullanacağım.

```text
~# zpool create tank /dev/sdb1
```

Bu komut hengi bir çıktı vermeyecek. `zpool create` komutu bizim bir aygıt havuzu oluşturmamızı sağlar. `tank` bizim ZFS havuzumunuzun adıdır \(diğer ZFS dökümanlarında hep tank olarak koymuşlar, sanırım bu bir gelenek ve bozmak istemedim\). `/dev/sdb1` ise `/dev/sdb`'ye bağlı olan diskin ilk bölümünü işaret ediyor. Bu komutun tamamlanması ile daha öncesinde belirttiğim gibi disk alanımız kök dizine bağlanacaktır.

```text
~# ls -ali / |grep tank
     34 drwxr-xr-x.   2 root root     2 Şub 24 18:44 tank
```

`/dev/sdb1` üzerine bu aygıt havuzunu oluşturmamın sebebi ZFS'yi tek bir disk alanında kullanmak. Eğer ki diskinizin içerisinde birden fazla bölüm varsa ve bu bölümleri kaybemek istemiyorsanız kesinlikle bu şekilde bölüm belirtmenizi tavsiye ederim. Örneğin ZFS on Linux kurulumu yapacaksanız bu şekilde bir havuz oluşturmak daha mantıklı olacaktır çünkü aynı diskte, `boot` ve `EFI` için ayrı disk alanlarına ihtiyaç duyabilirsiniz.

Ancak tüm bir diski ZFS olarak kullanmak istiyorsanız şunu yapmanız gerekmekte.

```text
~# zpool create tank /dev/sdb
```

Şimdi bu işlemlerin sonucunda bir aygıt havuzu oluşturduğumuzu varsayalım. En başta yapıtığımız gibi zpool listemizi kontrol edelim. Bunu da `zpool list` komutu ile yapabiliyoruz.

```text
~# zpool list
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  14.5G   100K  14.5G        -         -     0%     0%  1.00x    ONLINE  -
```

Bu komutun çıktısında gördüğümüz gibi aygıt havuzunun toplam boyutu, kullanılan ve boş alan, aygıt havuzundaki disklerin sağlığı durumu ve ek bazı detaylar var. Bu komut sayesinde sisteminizde yer alan bütün ZFS havuzlarının listesini kolaylıkla görebilirsiniz.

## Aygıt Havuzunun Durumunu Görüntülemek

Basit bir disk havuzu oluşturalım
```text
~# zpool create tank /dev/sdb
```

ZFS havuzumuza ait detayları görüntüleyelim şimdi. Bunun için de bir diğer komuta ihtiyacımız var. `zpool status` ile havuzumuza bağlı bütün diskleri ve durumlarını görebiliriz.

```text
~# zpool status
  pool: tank
  state: ONLINE
  config:

    NAME        STATE     READ WRITE CKSUM
    tank        ONLINE       0     0     0
        sdb     ONLINE       0     0     0

errors: No known data errors
```

"Zpool status" komutundaki satırlar, havuz hakkında size çoğu kendinden açıklamalı hayati bilgiler verir. Çıktıyı anlaması ve regex ile işlemesi oldukça basittir. Üstten aşağı olarak bazı tag'lar altında bazı bilgiler bize aktarılmakta. Hepsi burada görünmese bile ben bütün tanımları aktarmak istiyorum:

  * **pool:** Havuzun adı.
  * **state:** Havuzun mevcut sağlığı. 
  * **status:** Havuzda neyin yanlış olduğuna dair bir açıklama. Sorun bulunmazsa bu alan yazılmayacaktır. Ancak hatalar yaşanması durumunda detaylı bilgiler bu anahtar başlık altına basılacaktır.
  * **action:** Hata bulunması durumunda onarmak için önerilen eylem. Bu alan, kullanıcıyı dokumantasyon bölümlerinden veya daha öncesinde açılmış hata bildilerinden birine yönlendiren kısaltılmış bir formdur. Sorun bulunmazsa bu alan yazılmayacaktır.
  * **see:** Ayrıntılı onarım bilgilerini içeren bir bilgi makalesine referans verir. Sorun bulunmazsa bu alan yazılmayacaktır.
  * **scrub:** Son disk düzenlemesinin tamamlandığı tarih ve saati, devam eden bir düzenleme işleminin durumunu veya herhangi bir düzenleme talebi istenmemişse dahil olabilen bir düzenleme işleminin mevcut durumunu tanımlar.
  * **errors:** Bilinen veri hatalarını veya bilinen veri bloğu hatalarının yokluğunu tanımlar.
  * **config:** Havuzu oluşturan cihazların konfigürasyon düzeninin yanı sıra durumlarını ve cihazlardan üretilen hataları açıklar. Durum belirteçleri şunlardan biri olabilir: **ONLINE**, **FAULTED**, **DEGRADED**, **UNAVAILABLE**, veya **OFFLINE**. Durum **ONLINE** dışında bir şeyse, havuzun hata toleransı havuzu tehlikeye atılmıştır.

  Durum çıktısındaki sütunlar ise şu şekilde tanımlanır:

    * **NAME:** Havuzdaki her bir **VDEV**'in iç içe sırayla sunulan adı.
    * **STATUS:** Havuzdaki her **VDEV**'nin durumu. Durum, yukarıdaki "config" de bulunan durumlardan herhangi biri olabilir.
    * **READ:** Bir okuma talebi yayınlanırken yaşanan I/O hatalarını aktarır.
    * **WRITE:** Bir yazma talebi sırasında yaşanan I/O hatalarını aktarır.
    * **CHKSUM:** Sağlama toplamı hataları. Cihaz, bir okuma isteğinin sonucu olarak bozuk veriler döndürürse bunu bildirir.

## Birden Fazla Diskle ZFS Aygıt Havuzu Oluşturmak

`zpool` ile aynı anda birden fazla diski de tek havuzda kullanabileceğimizden bahsetmiştim. Bunu ilk kurulum aşamasında yapabiliriz. Bunun için şu komutu vermemiz yeterlidir.

```text
~# zpool create tank /dev/sdb /dev/sdc
```

Bu komutla iki tane 16 GB'lık diski bir zpool içerisine eklemiş olduk. Bu işlem sonucunda havuz alanımız şu şekilde görünecektir.

```text
NAME   SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank  29.0G   148K  29.0G        -         -     0%     0%  1.00x    ONLINE  -
```

Eklediğimiz yeni disk alanını, daha fazla disk alanına sahip olmak için kullanmak zorunda değiliz. Bunu daha öncesinde belirttiğim aynalama yani `mirror` işlemi için de kullanmak isteyebiliriz. Bu durumda bize `RAID` yuvalaması \(nesting\) yardımıza koşuyor. Bunu bir sonraki kısımda özellikle detaylandırarak anlatacağım ama şimdi bahsetmeden geçmemek istedim. Bu özellik sayesinde eklediğimiz diskleri gruplayarak ekleyebiliriz. Bu grupların her birisi özel kullanım için ayrılmış disk alanlarıdır, Havuz oluştururken gruplamak için grup olarak kullanılacak diskler bu gruba ait parametre belirtilerek aygıt havuzları oluşturulur. Kimi diskleri günlükleme kimi diskleri geçici depolama ve önbellekleme için kullanabiliriz. Her grubun kendi amacı ve parametreleri hakkında detayli bilgiyi ilerleyen sayfalarda vereceğim. Ancak örnek oluşturması için bazı aygıt havuzları oluşturalım. Bunu oluştururken şunu yapmamız yeterlidir.

```text
~# zpool create tank mirror /dev/sdb /dev/sdc
```

ZFS havuzumuza ait detayları görüntüleyelim şimdi.

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

Bunlara ek olarak aygıt havuzunda birden fazla grup da ekleyebiliriz. Örneğin `mirror`, `log` ve `cache` diskleri ekleyelim.

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

## ZFS Disk Havuzuna Yeni Diskler Eklemek

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

## ZFS Aygıt Havuzlarının İçe Aktarılması

Diyelim ki bir başka bilgisayarda uğraştığımız bir ZFS havuzu var. Bu havuzu başka bir bilgisayara bağlamak için `zpool import` komutu kullanılır.

```text
~# zpool import tank
```

Bu işlem bütün disklerin ve temel aygıtların yapılandırması, bellek şeritlerinin okunması ve uygun şekilde bağlanması için biraz süre gerekecektir. Bu sürenin ardından uygun şekilde bağlanacak ve kök sisteme bu havuz bağlanacaktır.

## ZFS Aygıt Havuzundan Disk Çıkarmak

Şimdi yaptığımız işleri birazcık da geri sardıralım.

Diyelim ki bu disklerden bir tanesini çıkarıp yerine bir başka disk takmamız lazım. Veya artık bu kadar çok bir depolama alanına ihtiyacımız yok. Bu durumda bütün bir aygıt havuzuna zarar vermeden havuzdan diskleri çıkartabiliriz.

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

## ZFS Aygıt Havuzunu Yok Etmek

Bütün bir aygıt havuzunu silmek için ise `zpool destroy` komutunu kullanabiliriz.

```text
~# zpool destroy tank
```

```text
~# zpool status
no pools available
```

## ZFS Aygıt Havuzlarının Dışa Aktarılması

Diyelim ki üzerinde çalıştığımız ZFS havuzunu başka bir bilgisayara aktarmak istiyoruz. Halihazırda ekleme yapmayı yukarıda anlattım ama başka bir bilgisayara bağlamadan önce güvenli bir şekilde çalıştığımız bilgisayardan kaldırmamız lazım. Bu işlem için `zpool export` komutu kullanılır.

```text
~# zpool export tank
```

Bu işlem bütün disklerin yapılandırması ve verilerin senkronize edilerek ayrılması biraz uzun sürecektir. Bu sürenin ardından uygun şekilde bağlanacak ve kök sisteme bu havuz bağlanacaktır.

## ZFS Havuz Geçmiş

ZFS aygıt havuzu sistemi, bu yaptığımız işlemleri adım adım görmemize imkan sağlayan bir geçmiş günlüğü yapısına sahiptir. Bütün bu değişimleri `zpool history` ile görebiliriz. Bu komuta ek olarak havuzumuzun adını verirsek, özel olarak o havuza ait geçmişi verecektir. Eğer herhangi bir parametre vermezsek bütün havuzlara ait geçmişleri verecektir. Bu geçmiş bilgisi disklerin aygıt havuzunu belirten hearder kısımları içerisine toplanır, yani başka bir bilgisayara bu diskleri aktardığımız zaman silinemez. Ancak `zpool destroy` komutu ile bu aygıt havuzunu yokettiğimizde bu geçmişi de kaybederiz.

```text
~# zpool history tank

History for 'tank':
2021-02-25.13:16:26 zpool create tank /dev/sdb1 /dev/sdb2 /dev/sdb3 /dev/sdb4 /dev/sdb5 /dev/sdb6
2021-02-25.13:19:38 zpool remove tank sdb5
2021-02-25.13:23:28 zpool remove tank sdb6
2021-02-25.13:25:42 zpool add tank mirror sdb4 sdb5 sdb6
2021-02-25.13:31:12 zpool export tank
2021-02-25.13:36:23 zpool import tank
```


---
description: Karmaşadan düzen oluşturmak...
---

# ZFS'yi Anlamak

## ZFS Terminolojisi ve Teknik detayları

### ZFS Disk Havuzu: zpool

**zpool**, ZFS için disk ve sanal disk havuzu oluşturan katmandır. **ZFS hiyerarşisinde** en üstündeki yapı **zpool’**lardır.

**zpool**'lar altında sanal diskler olarak bilinen **vdev** aygıtlarını içerir. **vdev**lerin eşzamanlaması, düzenlenmesi ve bilimum yönetiminden **zpool** sorumludur. Fiziksel olarak bir cihazda birden fazla **zpool** bulunabilir. **zpool** bir disk üzerinde kurulu olacağı gibi birden fazla diski birleştirmek için de kullanılabilir. Alt sistem olan **vdev**ler için ise durum biraz farklıdır. Bir **zpool**'a dahil olan **vdev** bir başka **zpool** tarafından kullanılamaz, yani birden fazla havuz tek **vdev**'i paylaşamaz.

Modern **vdev**'ler sistem kesintilerine karşı koruma sağlamaktadır. Ancak disk günlüklemesi yapan **vdev**'ler veya özel amaçlı bazı **vdev**'lerin yok olması durumunda **zpool**'da tanılı verilerde bozulma yaşanabilir ve hatta tüm **zpool** yok olabilir.

Bir diğer yandan **zpool** yapısına dahil olan yazma ve dağıtım mekanizması sayesinde normalin üzerinde disk yükü binmesi durumunda yaşanan gecikmeler daha azdır.

Ayrıca ZFS’de disk yazma-okuma işlemleri sadece **zpool**'lardan ibaret değildir. ZFS dosya sisteminde **zpool** klasik bir **RAID** mantığı ile çalışmaz. **zpool**, **JBOD** isminde bir mekanizmayı içermektedir. Bu kompleks bir dağıtım mekanizmasıdır. ZFS'de yazılan veriler boş alana göre uygun **vdev**'ler arasında dağıtılarak uygun **vdev**'ler dolu hale getirilir. Ayrıca **vdev**'leri uygun şekilde ayarlamak **JBOD** sistemi ile mümkündür. Öncelik ayarlarması yapılması sayesinde eğer bir **vdev** diğerlerine göre daha yoğun veya kritik öneme sahipse **vdev**'in atlanarak geçeçi süreliğine başka **vdev**'lere yazılır. Yoğunluğun düzelmesi durumunda da **vdev**'ler yeniden düzenleneyerek veri daha sıralı bir şekilde dağıtılması sağlanabilir.

### ZFS Sanal Diskleri: vdev

**vdev** \(**virtual devices**\) **\*\*disk havuzunu oluşturan sanal aygıtlara verilen addır. Her bir** zpool**'un içerisinde bulunan** vdev\*\*'ler, bir veya birden fazla diskten meydana gelebildiği gibi birden fazla disk alanından da meydana gelebilir.

**vdev**'ler veri depolamanın yanısıra önbellekleme \(**CACHE**\), günlükleme \(**LOG**\) ve özel kullanımlar için yapılandırılabilir. **ZFS** **vdev**'leri **RAIDz** yapısını kullanır.

Daha önce de belirdiğim gibi **RAIDz ZFS'**sinin donanım katman RAID yapısına oluşturduğu bir çözümdür. **RAIDz** yapısı **vdev**'in temelini oluşturur.

Beş farklı **RAIDz** modu vardır ve bu modlar **vdev** tarafından davranışsal oalrak yönetilir. **RAIDz** modları: **şeritleme** \([RAID 0](https://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_0)'a benzer, artıklık sunmaz\), **RAIDz1** \([RAID 5](https://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_5)'e benzer, bir diskin arızalanmasına izin verir\), **RAIDz2** \([RAID 6](https://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_6)'ya benzer, iki diskin başarısız olmasına izin verir\), **RAIDz3** \(RAID 7 yapılandırmasına benzer, üç diskin başarısız olmasına izin verir\) ve **ikizleme** \([RAID 1](https://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_1)'e benzer, disklerden biri dışında tümünün başarısız olmasına izin verir\).

Bu yapılardan üçü; **RAIDz1**, **RAIDz2**, **RAIDz3**, “**diagonal parity RAID**” olarak isimlendirilen özel yapılardır. Bu özel yapılar veri yollarını ve her bir veri yoluna ayrılan eşlik bloklarını ifade etmek için kullanılır. Bu şekilde, çeşitli eşliklere bölünmüş diskler yerine **RAIDz**'ler kullanarak dağıtılabilir ve bu yöntem farklı eşliklere ayrılmış disklerden daha optimal bir şekilde çalışır. Ancak eşlik bloklarının kaybedilmesi durumunda tüm disk kaybedilebilir. Özel **RAIDz**'lerin kaybedilmesi ise zpool'un tamamının yokolmasına neden olabilir.

**Aynalama** yapısında kullanılan **vdev'**lerde ise her bir yansı bir **vdev** üzerine, her bir blok ise bir **vdev** cihazında barındırılır. Yaygın olarak tek **vdev** içeren yansılama kullanılmasında rağmen birden fazla cihaz içeren aynalama da yapılabilir. Ancak daha az hata ile karşılamak ve sistem üzerine olan yükü azaltmak için genellikle büyük aygıt yapılarında RAIDz3 yapısı kullanılır. Bu sayede yansı **vdev**’i içindeki herhangi bir aygıt sağlıklı devam ettiği sürece yaşanacak olası problemlerden kurtulabilir.

Tek aygıta bağlı **vdev**'ler veri kayıplarına oldukça açık bir yapıya sahiptir. Bu tip **vdev**'lerde herhangi bir problem çıkması durumunda **zpool**'un tamamı kaybolur. **CACHE**, **LOG** veya **SPECIAL** vdev yapıları bahsi geçen **RAIDz** yöntemlerindenden biri kullanılarak oluşturulabilmekte lakin herhangi bir özel bir vdev’in kaybolması **zpool**'un tamamen yokolmasında sebep verir. Bu nedenle kritik diskler için aynalama yapılması ya da ekstra bir yapı kurulması tavsiye edilir.

### ZFS Cihazları: Z Devices

Bildiğiniz gibi **zpool**'lar **vdev**'lerden, **vdev**'ler de çeşitli blok aygıtlardan oluşmaktadır. Bu blok cihazlarına biz ZFS cihazları diyebiliriz. Bazı kaynaklar bu cihazları Z Devices olarak adlandırmaktadır.

Sabit diskler \(**HDD**\), katı hal diskleri \(**SSD**\), hibrit diskler \(**HHD**\) veya diğer tipte diskler bu çeşitte cihazlar olabilir. Ayrıca ham veri \(**RAW**\)'de direk ZFS için cihaz olarak kullanılabilir.

Ayrıca cihazların çeşitli kullanım alanları mevcuttur. Bunlardan en yaygın olanı **vdev**'ler içerisinde veri depolama görevini üstlenmesidir, bir diğer yandan bu cihazlar özelleştirilebilir. Örneğin bir cihaz özelliğini Hotspare için kullanarak **vdev**'lerden bağımsız olarak veri depolayabilirsiniz. Bu cihaz normal cihazlardan farklı olarak tek **vdev**’e ait değil, tüm **zpool**'a aittir.

**zpool** bileşimi, benzer cihazlarla sınırlı değildir,  **ZFS**'nin sorunsuz bir şekilde bir araya topladığı ve daha sonra farklı dosya sistemlerine gereken alanı bıraktığı **geçici cihazlar \(Temporary devices\)**, **heterojen cihaz koleksiyonlarından \(HDC\)** ve özelleştirilmiş başka cihazlardan oluşan büyük bir yapıdır. 

Ayrıca cihazlar **zpool**'un boyutunu artırmak için de kullanılabilir. Özelleştirilerek **zpool** içerisine eklenen cihazlar havuz boyutunun artırılmasına ve depolama kapasitesinin düzenlenmesine yardımcı olur.

Bu açıdan bakıldığında cihazlar aslında **ZFS**'nin bir diğer önemli yapı taşını ihtiva etmektedir. Şimdi bazı sık kullanılan cihazlara göz atalım.

#### Yansı Cihazları 

Yansı cihazları daha önceki sayfalardan aşina olduğumuz bir terim. Yansılama ile anlık verileri birden fazla cihazda tutabilir, verileri koruyabilirsiniz. Ayrıca bu cihazlar bir başka amaçla daha kullanılır. **ZFS** dosya sistemleri diğer havuzlara, ayrıca ağ üzerindeki uzak ana bilgisayarlara taşınabilmesi bu yansı cihazları ile mümkündür. Bu akış cihazı ile, belirli bir anlık görüntüde dosya sisteminin tüm içeriğini veya anlık görüntüler arasında bir delta içeriğini yerel veya ağ üzerinden yansılayabiliriz. Bu cihaz, örneğin site dışı yedeklemeleri veya bir havuzun yüksek kullanılabilirlik aynalarını senkronize etmek için verimli bir strateji sağlar.

#### Özel vdev Cihazları 

**ZFS** 0.8 ve sonrasında, dosya sistemi meta verilerini tercihli olarak depolamak için Özel bir **vdev** sınıfı ve isteğe bağlı olarak Veri Tekilleştirme Tablosu \(DDT\) kullanarak küçük dosya sistemi blokları yapılandırmak mümkündür. Bu, örneğin, meta verileri depolamak için hızlı katı hal depolamada özel bir vdev oluşturup, normal dosya verilerin daha yavaş olan disklerde depolanması için kullanılabilir. Aslında bu özellik önceki sayfalardan aşina olduğumuz **ARC** ve **L2ARC** yapısından türetilmiştir**.** Bu cihazları kullanarak, tüm dosya sistemini katı hal depolamadan bir başka disk üzerine depolama masrafı olmadan geçişi, düzeltme ve yeniden yapılandırma gibi yoğun meta veri işlemlerini hızlandırabiliriz.

#### 



#### 


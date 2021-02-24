# ZFS'nin Özellikleri

## ZFS'nin Özellikleri

2001 yılında ZFS sadece bir disk yönetim sistemi ihtiyacından dolayı değil mevcut mimarinin dezavantajlarını kapatacak yeni bir format geliştirmek idi. İlerleyen zamanda verilerin tutulması, yönetilmesi ve korunması için en uygun sistemlerden birisi oldu. Peki nedir ZFS'yi bu denli önemli yapan özellikler?

### Ana hatları ile ZFS

ZFS disk yapısının getirdiği sistem mevcut disk yapıları ve RAID dizilerinden farklılık gösterir. Birden fazla diski ve sanal aygıtları \(**virtual devices - vdev**\) aynı anda bir disk havuzu **\(zpool\)** üzerinden birleştirir.

**Dosyalama sistemi ve Kapasite**

**ZFS**, **"128bit"** bir dosya sistemidir. Yani **128bit**, içindeki herhangi bir birim için en büyük boyutlu adrestir. Bu boyut, öngörülebilir gelecekte herhangi bir zamanda sınırlandırılması muhtemel olmayan kapasitelere ve boyutlara izin verir. Örneğin, uyguladığı teorik sınırlar dizin başına 2^48 giriş, maksimum dosya boyutu **16 EB** \(2 ^ 64 veya ~ 16 \* 2 ^ 18 bayt\) ve "**zpool**" başına maksimum 2^64 sanal aygıtı içerebilir.

ZFS 128-bit adresleme şeması sayesinde 256 katrilyon zettabayt depolayabilir, bu da **1000 PB \(petabayt\)** depolama kapasitesini aşan ölçeklenebilir bir dosya sistemine dönüşürken, tekli veya çoklu **RAIDz** dizilerinde yönetilmesine izin verir.

### ZFS'de Veri Yönetimi

ZFS geleneksel yöntemleri, dosya sistemi katmanlarına adapte eden yöntemlerle beraber kullanır. Bunu yaparken çeşitli mekanizmalar geliştirmiştir. ZFS'nin en büyük başarısından birisi [copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write) mekanizmasıdır ki bu mekanizma dosya yönetim sisteminde meydana gelebilecek her türlü bozulma ve kesintiye karşı dosyaların güvenli bir şekilde tutulabilmesini sağlar.

Aynı zamanda verilerin yerel olarak sıkıştırma ve tekilleştirme işini de veri iletimi sırasında yaparak diski daha optimize kullanmaktadır.

**Copy-on-Write**

ZFS, yazma işlemini nesne kopyalamasını esas alarak yapar. Aktif veri içeren bloklar asla yerinde yazılmaz; bunun yerine, yeni bir blok tahsis edilir, değiştirilen veriler üzerine yazılır, daha sonra buna referans veren herhangi bir meta veri bloğu benzer şekilde okunur, yeniden tahsis edilir ve yazılır.

Bu işlemin ek yükünü azaltmak için, birden çok güncelleme işlem grupları halinde gruplandırılır ve eşzamanlı yazma semantiği gerektiğinde ZIL \(amaç günlüğü\) yazma önbelleği kullanılır.

**Veri Tahsisi**

ZFS, bir veri havuzundaki tüm sanal disklerde havuzun performansını en üst düzeye çıkaracak şekilde veri depolamayı otomatik olarak tahsis eder. Ayrıca eklenen her bir sanal disk, havuzun veri depolama stratejisini en optimal hale getirmek için en optimal yolu seçerek kendini güncelleyecektir.

Genel bir kural olarak, **ZFS**, her sanal diskteki boş alanı temel alarak sanal diskler arasında yazma işlemlerini hesaplar. Bu, orantılı olarak daha az veriye sahip olan sanal disklere, yeni veri depolanacağı zaman daha fazla yazma yapılmasını sağlar. Bunun tek dezavantajı veri havuzuna veri biriktikçe havuzun veri yazılması için daha vasat bir hale gelmesine sebep olur. Bunu da yeni sanal diskler ekleyerek aşmak mümkündür.

**Veri Korunumu ve Tutulması**

ZFS içinde veri bütünlüğü, [Fletcher](https://en.wikipedia.org/wiki/Fletcher%27s_checksum) sağlama toplamı algoritmasını taban alan bir sağlama toplamı veya dosya sistemi ağacı boyunca bir [SHA-256](https://en.wikipedia.org/wiki/SHA-2) hash kullanılarak elde edilir.

Verilere ait sağlama toplamı, dosya sisteminin veri hiyerarşisinden kök düğüme kadar devam eder ve bu da sağlama toplamı alınır ve böylece bir [Merkle](https://en.wikipedia.org/wiki/Merkle_tree) ağacı oluşturur

Tüm verilerin ve meta verilerin hiyerarşisini, hiyerarşik sağlama toplamlarını \(checksum\) bloğun kendisinin üzerinde değil bir ana bloğun üzerinde tutarak, ana bloğu kullanarak alt bloklardaki verilerin kontrol edilmesini, kurtarılmasını ve geriye çevrilmesini sağlar.

Bütün bu özellikleri sayesinde verileri hiyerarşik olarak korumayı sağlar. Disk kullanımı sırasında veri bozulması veya hayali okumalar/yazmalar, sağlama toplamını verilerle birlikte sakladığı çoğu dosya sistemi tarafından tespit edilemez. Hatta bazı dosa sistemlerinde sağlama toplamı verileri hiç bulunmadığı için bütün veri çöpe bile çıkabilir. Ancak **ZFS**, her bloğun sağlama toplamını ana blok işaretçisinde saklar, böylece tüm havuz kendi kendini doğrular.

Ayrıca **hata ayıklama bayrakları** olarak adlandırılan mekanizması sayesinde bozunmuş verilerin duruma bağlı olarak kurtarılması ya da bozuk verilerin tespit ve temizlenmesi de otomatik sağlanır.

**Veri koruması ve Aynalama**

Bazı durumlarda, bir hata veya tutarsızlık durumunda dosya sisteminde ve verilerde yapılan son değişikliklerin otomatik olarak geri alınmasını sağlayan **anlık görüntü \(snapshot\)** özelliği bulunmaktadır. Kullanıcı elle veya işletim sistemlerine yazılan servisler yolu ile disk nloklarını yedekleyebilir, istediği bir andaki yedeğe geri dönebilir ya da geri yükleme yapabilir.

Yazılabilir anlık görüntüler **\(clone\)** da oluşturulabilir, bu da bir dizi bloğu paylaşan iki bağımsız dosya sistemi sağlar. Klon dosya sistemlerinden herhangi birinde değişiklik yapıldıkça, bu değişiklikleri yansıtmak için yeni veri blokları oluşturulur, ancak değiştirilmemiş bloklar, kaç klon var olursa olsun paylaşılmaya devam eder. Bu, **copy-on-write** ilkesinin bir diğer uygulamasıdır.

Ayrıca **ZFS**, verilerin birden çok kopyasını inode kopyaları halinde depolar. Özellikle meta veriler 4 veya 6'dan fazla kopyaya sahip olabilir. Bu sayede verilerin korunması ve diskte bir veri kaybı oluşması bu kopyaların en az birisi kullanılarak onarma sağlanılabilir

**ZFS**'nin bir diğer önemli özelliği de **vdev**'leri ayna sistem olarak kullanma özelliğidir. Aynalama yapılan disklerdeki verileri birbiri arasında eşzamanlı olarak korumaya yardımcı olur. Veriler istenilen yansıdan geri getirilebilir ve ek bir süreç kullanmadan yazma anında tüm diskin yedeği başka lokal kaynaklarda da korunabilir

### RAIDz

**ZFS**'nin veri bütünlüğünü garanti edebilmesi için kullandığı farklı yollardan sürekli bahsediyorum ancak bir diğer önemli özelliği yine veri korunumunu esas alan **RAIDz**'dir.

Genellikle birden çok diske yayılmış verilerde birden çok veri kopyasına ihtiyaç duyulur. Donanımsal olarak bu özellik, bir **RAID** denetleyicisi kullanılarak elde edilebilir.

**ZFS**, tüm depolama cihazlarına ham erişime sahipse ve diskler bir donanım, bellenim veya başka bir "yazılım" kullanılarak sisteme bağlı değilse, genellikle cihazın daha verimli çalışmasını ve veriyi en optimal olarak kullanıp veriyi koruması için donanım bazlı **RAID'den** kaçınır. Bunun sebebi ise ZFS'nin veri yolunu esas alarak çalışmasıdır. Bazı durumlarda **RAID** cihazının esas alınması verilerde kayba sebep verebilir. **RAID** denetleyicileri ayrıca, yazılım **RAID**'inin kullanıcı verilerine erişmesini önleyen sürücülere denetleyiciye bağlı veriler ekler. Bu durumda devam eden işlerde kayıplar yaşanabileceği gibi donanımsal hasarlar bile oluşabilir.

Bu nedenle, **RAID** kartlarının veya benzerlerinin kaynakları boşaltmak, işleme ve performansı ve güvenilirliği artırmak için kullanıldığı diğer birçok sistemden farklı olarak, **ZFS** ile bu yöntemlerin genellikle sistemin performansını ve güvenilirliğini düşürdükleri için kullanılmaz.

Donanımsal **RAID** yerine **ZFS**, kendi geliştirdiği **RAIDz** sistemini kullanır. RAIDz [RAID5](https://en.wikipedia.org/wiki/RAID_5) benzeri eşlik tabanlı yapı kullanırken ayrıca [RAID1](https://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_1)'e benzer disk aynalama sunan **"yumuşak" RAID** kullanır.

**RAIDz** dinamik şerit genişliği kullanır. Her blok, blok boyutuna bakılmaksızın kendi **RAID** şeridini ihtiva eder. ZFS, bunu mevcut veri semantiği ile birleştirerek, veri yazması esnasında yazma deliği hatasını ortadan kaldırır.

**RAIDz**'nin detaylarına ilerleyen sayfalarda tekrar değineceğim.

### ZFS'de Disk Kalaylama ve Fırçalama

**ZFS**, [fsck](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwiSrN3HgK3tAhXEjqQKHS6MDkwQFjACegQIARAC&url=https%3A%2F%2Flinux.die.net%2Fman%2F8%2Ffsck&usg=AOvVaw3PPzxybPATn2EwFj-w5PBr) olarak bilinen Unix disk yapılandırma aracına sahip değildir. Bunun yerine **resilvering \(kalaylama\)**, ve **scrub \(fırçalama\)** isminde iki araç getirir. Bu araçlar ile **ZFS**, tüm verileri düzenli olarak inceleyen; sessiz bozulma ve diğer sorunları onaran yerleşik bir temizleme işlevine sahiptir.

**ZFS**'nin bu araçlarının **fsck**'e göre pek çok artısı bulunmaktadır:

* **fsck**, çevrimdışı bir dosya sisteminde çalıştırılmalıdır. Bağlanmış diskler ile çalışmak mümkün değildir, bu da dosya sisteminin çıkarılmadan onarılamamasına ve onarılırken kullanılamamasına neden olur. Öteki yandan **scrub** bağlanmış, canlı bir dosya sisteminde kullanılmak üzere tasarlanmıştır ve **ZFS** dosya sisteminin bağının kaldırılmasına ya da çevrimdışı duruma getirilmesine gerek yoktur.
* **fsck** genellikle yalnızca meta verileri \(disk günlüğü gibi verileri\) kontrol eder, ancak verilerin kendisini asla kontrol etmez. Bu, bir **fsck**'den sonra verilerin depolandığı haliyle orijinal verilerle hala eşleşmeyebileceği anlamına gelir. Bunun da sebebi sağlama toplamlarının verilerle birlikte depolanmasına sebep olur. **fsck** bu verileri her zaman doğrulayamaz ve onaramaz. **ZFS** ise sağlama toplamlarını her zaman verilerden ayrı olarak saklar, bu da güvenilirliği ve birimi onarma becerisini artırır.
* **scrub**, meta veriler ve veriler dahil her şeyi kontrol eder bu sebeple yavaştır. Bazen büyük bir RAID üzerindeki **fsck** birkaç dakika içinde tamamlanır, bu da yalnızca meta verilerin kontrol edildiği anlamına gelir. Tüm meta verileri ve verileri büyük bir RAID'de incelemek uzun saatler sürer, bu da **scrub**'ın yaptığı tam olarak budur.Yavaş bile yapılsa tam manası ile veri düzeltilmesi sağlar.

### ZFS Disklerinde Şifreleme

**ZFS**'deki disklerin şifrelemesi I/O işlemine gömülü olarak yapılır. Veri işlemleri sırasında, bir blok sırayla sıkıştırılabilir, şifrelenebilir, sağlama toplamı alınabilir ve ardından tekilleştirilebilir.Bu sayede şifreleme üzerindeki yük anlık olarak bellek ve işlemci üzerine dağıtılır.

Şifreleme politikası, veri kümeleri oluşturulduğunda veri kümesi düzeyinde belirlenir. Kullanıcılar tarafından belirlenen şifreler, dosya sistemini çevrimdışı duruma getirmeden herhangi bir zamanda değiştirilebilir, aynı zamanda şifre anahtarlarının değiştirilmesi durumunda kısa sürede tüm işlemin tamamlanması kolaylaştıran sarmal şifreleme kullanılmaktadır.

### Yedek Parçalar ve Kotalar

ZFS veri kaybını şansa bırakmamak için onlarca özelliği içerisinde barındırır ve bunu yaparken bir yandan da hız faktörüne azımsanmayacak kadar çok dikkat eder.

Depolama havuzlarında mevcut dik hatalrını tamir etmek için **yedek parça \(spares\)** adı verilen kısımlar bulunur. Aynalama ile birlikte kullanıllanılan bu yedek parçalar fiziksel olarak gruplanır ve böylece tüm havuzun arızalanması durumunda yedek parçalar ile yola devam edilebilir.

Depolama havuzlarında, arızalı diskleri telafi etmek için çalışırken yedek parçalar bulunabilir. Aynalama sırasında, blok aygıtlar fiziksel kasaya göre gruplandırılabilir, böylece tüm kasanın arızalanması durumunda dosya sistemi devam edebilir.

Tüm **vdev**'lerin depolama kapasitesi, **zpool**'daki tüm dosya sistemi örneklerinde mevcuttur. Bu örnekler ile **ZFS** diskin dolumunu öngörebilir. Ayrıca dosya sistemi örneğinin kaplayabileceği alan miktarını sınırlamak için bir kota belirlenebilir ve bir dosya sistemi örneğinde yer olmasını garanti etmek için bir ayırma ayarlanabilir. Bunun sayesinde disk üzerinde karmaşık bir bölümlendirme tablosuna ihtiyaç duymadan mevcut cihaz veya **vdev**'leri kullanarak diskler sınırlandırılabilir.

### ZFS'nin Önbellekleme Mekanizması

ZFS veri aktarımının kesintisiz ve düzgün yapılabilmesi için önbellekleme yöntemlerinin bazılarını kullanmaktadır. Bütün disk tiplerinin kullandığı RAM üzerinden önbellekleme işlemi donanım için hayli masraflı bir işlemdir. Bu kayba karşın, performansı optimize etmek için veriler bir hiyerarşide otomatik olarak önbelleğe alınır; bu tip önbelleğe "karma depolama havuzları" denir. Bu yöntemle sık olarak kullanılan veriler RAM belleğe atılarak performans artışı sağlanırken, maliyetten kaçınmak için orta sıklıkla kullanılan veriler hızlı olan diske basılır \(bu bir katı hal diski olabilir, hızın önceliği vardır\).Sık erişilmeyen veriler önbelleğe alınmaz ve yavaş sabit disklerde \(örneğin HDD'lerde\) bırakılır. Eski veriler aniden çok okunursa, ZFS onu otomatik olarak hızlı olan disklere veya duruma göre RAM'e taşır. Sonuç olarak performans ile kaynak kullanımı optimal seviyede tutulur.

Bu önbellekleme mekanizması 4'e ayrılır:

1. **ARC \(adaptive replacement cache\):** Uyarlanabilir değiştirme önbelleği olarak bilinir. **ARC** boyutunun yeterince büyük olması koşuluyla, genellikle disklere erişilmesine gerek olmadyan verilerin depolandımasını sağlar. RAM çok küçükse, hemen hemen hiç **ARC** açılmayacaktır; bu durumda, ZFS'nin her zaman performansı ikinci plana atarak RAM belleği üzerinden önbellekleme yapmayı engelleyecektir. **ARC** önbelleği "işlem grupları" \(**transactional groups**\) aracılığıyla işlenir, duruma bağlı olarak her bir işlem grubu 5 saniye ile 30 saniye arasında işleme tabi tutularak diske basılır. Bir grup diske basılırken başka bir grup işlem grubu içerisine dahil edilerek eşzamanlı olarak işleme ve basma işlemleri gerçekleştirilir. Bu özellik, güç kesintisi veya donanım arızası durumunda en son işlemlerde küçük veri kaybı riski içerirken, temeldeki diskler için yazma işlemlerinin daha verimli bir şekilde organize edilmesini sağlar. Güç kaybı riski, **ZFS** yazma günlük kaydı ve **SLOG**/**ZIL** ikinci kademe yazma önbellek havuzu  ile önlenir, 
2. **SLOG\(Second Log\)**: ZFS Amaç Günlükleri olarak bilinir. Bir **SLOG** \(ikincil günlük cihazı\), bir sistem sorunu durumunda yazmaları kaydetmek için ayrı bir cihazda bulunan isteğe bağlı özel bir önbellektir. Bir **SLOG** cihazı mevcutsa, ikinci seviye günlük olarak **ZIL \(aşağıda değinilecek\)** için kullanılacak ve ayrı bir önbellek cihazı sağlanmadıysa, bunun yerine ana depolama cihazlarında **ZIL** oluşturulacaktır. Yani **SLOG**, teknik olarak havuzu hızlandırmak için **ZIL**'in yüklendiği ayrılmış bir ikinci diski ifade eder. Açıkçası, **ZFS**, birincil olarak yazılan verileri önbelleğe almak için **SLOG** aygıtını kullanmaz.Yazmaların kalıcı bir depolama ortamına olabildiğince hızlı bir şekilde yakalanmasını sağlamak için **SLOG**'u kullanır, böylece güç kaybı veya yazma hatası durumunda, yazıldığı kabul edilen hiçbir veri kaybolmaz.  
   **SLOG** cihazı, **ZFS**'nin, yerel depolama aygıtları için bile yazma işlemlerini hızlı bir şekilde depolamasına ve sorunları hızlı bir şekilde rapor etmesine olanak tanır. Normal faaliyet sürecinde, **SLOG**'a hiçbir zaman görevlendirmede bulunulmaz veya okunmaz ve önbellekleme yapmaz. Amacı, harmanlama ve yazmanın başarısız olması durumunda "yazma" için geçen birkaç saniye boyunca verileri aktarım esnasında korumaktır. Her şey yolunda giderse, depolama havuzu, mevcut işlem grubu diske yazıldığında 5 ila 60 saniye içinde bir noktada güncellenecek ve bu noktada **SLOG**'a kaydedilen veriler orada kalacaktır ve yeniden benzer bir durum yaşanırsa bu önbellek üzerine yeni veriler eklenerek eski veriler yoksayılacaktır. **SLOG** ancak ve ancak yazma sonunda başarısız olursa veya sistem, yazılmasını engelleyen bir çökme veya hatayla karşılaşırsa, **ZFS** tarafından _\*\*_tekrar okunarak yazın verilerin doğruluğunu kontrol için kullanılacaktır.

   Bu,eşzamanlı yazma özelliği bulunan \(ESXi, NFS ve bazı veri tabanlarında olduğu gibi\) yazılımlardaki gibi gerçekleşirse çok önemli hale gelir, çünkü, istemcinin faaliyetine devam etmeden önce başarılı bir yazma onayına ihtiyacı vardır; **SLOG**, **ZFS**'nin yazmanın başarılı olduğunu, istemciyi veri depolama durumu konusunda yanıltma riski olmaksızın, her seferinde ana depoya yazmak zorunda kaldığından çok daha hızlı doğrulamasına olanak tanır. **SLOG** cihazı yoksa, ana veri havuzunun bir kısmı aynı amaç için kullanılacaktır, ancak bu veri aktarım hızını yavaşlatacaktır.

   \*\*\*\*

3. **ZIL\(ZFS Intent Log\): ZIL** cihazı **SLOG**'dan farklı olarak bir ara katman görevi görür ve kendisi kaybolursa, en son yazılanları kaybetmek mümkündür, bu nedenle günlük cihazı başka **vdev**'lere aynalanmalıdır. **Ayrıca çok sık rastlanmasa dahi, büyük çaplı bir kayıpta ZIL cihazı sebebi ile veri havuzu tamamen yok olabilir.** Bu sebeple ZIL kullanan kullanıcılara aynalama yapması önerilir.
4. **L2ARC \(Level 2 ARC\):** İsteğe bağlı oluşturulur. **ZFS**, birçok durumda onlarca veya yüzlerce gigabayt olabilen **L2ARC**'de olabildiğince çok veriyi önbelleğe alır. Tüm tekilleştirme tablosu **L2ARC**'de önbelleğe alınabiliyorsa, **L2ARC** aynı zamanda tekilleştirmeyi önemli ölçüde hızlandıracaktır. **L2ARC** cihazı kaybolursa, tüm okumalar önbelleklenmeden disklere gidecek ve performans etkilenecektir, ancak başka hiçbir veri kaybı yaşanmayacaktır.

### Uyarlanabilir dayanıklılık \(Adaptive endianess\)

Havuzlar ve bunlarla ilişkili alt dosya sistemleri ve cihazlar; farklı bayt sıraları uygulayan sistemler dahil olmak üzere pek çok farklı platform mimarileri arasında taşınabilir. **ZFS** blok işaretçi biçimi, dosya sistemi meta verilerini endian uyarlamalı bir şekilde depolar; bireysel meta veri blokları, bloğu yazan sistemin yerel bayt sırası ile yazılır. Okurken, saklanan sonluk sistemin sonluluğuyla eşleşmiyorsa, meta veriler bellekte bayt değiştirilir. Bu da **ZFS**'ye uyarlanabilir bir dayanıklılık sağlar.

**POSIX** sistemlerinde olağan olduğu gibi, dosyalar uygulamalara basit bayt dizileri olarak görünür, bu nedenle veri oluşturan ve okuyan uygulamalar, temeldeki sistemin dayanıklılığından bağımsız bir şekilde bunu yapmaktan sorumlu kalır.

### Veri tekilleştirme

Sürekli bu durumu yazıp durdum. Veri tekelleştirme de veri tekelleştirme şeklinde. Pek çoğunuz az çok bir çıkarımda bulunmuş bile olsa bunu açmak isterim.

Veri tekelleştirme kavramı aslında sırasız olarak gelen verilerin temizlenerek düzenli bir şekilde, depolamaya hazır hale getirilmesini sağlayan bir yöntemdir. **ZFS** kendi içerisinde bir veri tekelleştirme yapısı içerir. Bu yapı tekrarlanan veri ve fuzzy verilerin ayıklanmasını, şifreleme kullanımı için şifrelenmesini, sıkıştırma kullanan **ZFS** yapıları için ise sıkıştırmanın yapılmasını sağlar. Tekelleştirme daha önbellekleme esnasında yapılır. Bu sayede hızlı ve daha temiz bir veri, disk kullanılmadan elde edilebilir.

Veri tekilleştirmenin ZFS'de etkili kullanımının dezavantajlarının bir tanesi ZFS'nin bazı işlemler için büyük RAM kapasitesi gerektirmesidir ki öneriler, her 1 TB depolama için 1 ile 5 GB RAM arasında RAM kullanımının esas alınarak sistem kurulumu yapılmasını tavsiye etmekte. Yetersiz fiziksel bellek veya ZFS önbelleğinin olmaması, veri tekilleştirme kullanılırken sanal belleğin çökmesine neden olabilir, bu da performansın düşmesine veya tam bellek açlığına neden olabilir ve sistem önemli ölçüde yavaşlayabilir.


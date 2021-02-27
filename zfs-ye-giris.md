# ZFS'ye Giriş

ZFS'de disk yönetimi iki temel üzerinde ele alınır. Üstte sanal disk havuzu **\(zpool\)** bulunmaktadır. Bu sanal disk boyutu altında farklı disk alanları ve dosya hiyerarşilerinden oluşan bir grup dosya sistemi bulunmaktadır **\(Z File System\)**.

ZFS'de disk alanları ve dosya sistemleri, sanal disk havuzunun altında oluşturulan katmandır. Farklı ZFS diskler ve farklı bölümleri aynı havuz içerisine kaydedilerek bu havuza ekleme ve çıkarma işlemi yapılabilir. ZFS dosya sistemleri, herhangi bir temel disk alanı ayırmanıza veya biçimlendirmenize gerek kalmadan dinamik olarak oluşturulabilir ve yok edilebilir. Dosya sistemleri çok hafif olduğundan ve ZFS'de merkezi yönetim noktası olduklarından, bunlar hem oldukça esnek hem de manüplasyonu tehlikeli yapılardır. Yani yaptığımız düzenlemeler kesinlikle çok dikkatlice yapılmalıdır. Çünkü dosya sistemleri üzerinde meydana gelen bir hasar kesinlikle bütün havuzu etkileyecektir ve hatta havuzun yok olmasına bile sebep olabilecek bir veri sorununa sebep olacaktır.

Başlamak için, ZFS bunları dahili olarak kapsamlı bir şekilde kullandığından, sanal cihazları veya **VDEV**'leri anlamamız gerekir. Basitçe açıklamak, bir veya daha fazla fiziksel cihazı temsil eden bir meta cihazımız var. Linux yazılım RAID'inde, 4 diskten oluşan RAID-5 dizisini temsil eden bir "/dev/md0" aygıtınız olabilir. Bu durumda, "/dev/md0" sizin "**VDEV**" aygıtınız olacaktır. Aynı şekilde bir dosya üzerine oluşturuduğunuz sanal disk "/tmp/disk1" yolunda olsun. Bu disk de bir dosya cihazımızdır ve bu da bizim için bir **VDEV** olacaktır.

ZFS'de yedi tür VDEV cihazı türü vardır:

* **disk \(varsayılan\):** Sisteminizdeki fiziksel sabit sürücüler. 
* **dosya \(file\):** Önceden ayrılmış dosyalardan/görüntülerden oluşan sanal sürücüler. 
* **ayna \(mirror\):** Standart yazılım RAID-1 aynası
* **raidz1/2/3:** Standart olmayan dağıtılmış eşlik tabanlı yazılım RAID seviyeleri. 
* **yedek \(spare\):** ZFS RAID için "etkin yedek" olarak işaretlenmiş sabit sürücüler. 
* **önbellek \(cache\):** Seviye 2 uyarlanabilir okuma önbelleği \(L2ARC\) için kullanılan cihaz. 
* **günlük \(log\):** "ZFS Amaç Günlüğü" veya ZIL adı verilen ayrı bir günlük \(SLOG\).

ZFS dosya sistemleri, `zfs` komutu kullanılarak yönetilir. ZFS komutu, dosya sistemlerinde belirli işlemleri gerçekleştiren bir dizi alt komut sağlar. Bu alt komutlar basit disk işlemleri \(disk alanı ekleme, çıkarma, kontrol etme\) haricinde daha öncesinde bahsettiğim aynalama \(`mirror`\) ve yedekleme \(`spare`\) işlerini de sağlar.

ZFS dosya sistemlerini oluşturmak ve yönetmek için ilk yapmamız gereken işlem bir ZFS disk havuzu oluşturmak. İlk adımda bu havuzları nasıl oluşturup nasıl manüple edeceğimizi göreceğiz.

Bu aşamada temel işlemleri ikiye ayıracağım. ZFS Disk yönetim sistemi için bir disk havuzu oluşturmak zorundayız. Bu bizim birinci kısmımızı oluşturacak. İkincisi ise bu havuzlar içerisinde ZFS disk bölümleri oluşturabilmeyi ve üzerinde işlemler yapmayı öğreneceğiz.


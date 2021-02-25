
# ZFS Disk Hiyerarşisi

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


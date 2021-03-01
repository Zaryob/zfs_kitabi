---
description: ZFS Havuz Yönetimine dair bazı ek özellikleri açıklamaktadır.
---

# ZFS Havuz Yönetimi - Bazı Ek Havuz Yönetimi Özellikleri

Bu kısımda önceki öğrendiğimiz komutların haricinde bazı ek ZFS komutlarına örnekler verecek ve bunlara ait örnek komutlar vereceğim. Burada çoğunlukla kullanmayacağınız ama yine de önemli olan bazı ZFS özelliklerinden bahsedeceğim.

## Yansıtılmış bir ZFS Depolama Havuzunu Bölerek Yeni Bir Havuz Oluşturma

Yansıtılmış bir ZFS depolama havuzu, zpool split komutu kullanılarak hızlı bir şekilde yedekleme havuzu olarak klonlanabilir.

Şu anda bu özellik, yansıtılmış bir kök havuzunu bölmek için kullanılamaz.

Ayrılmış disklerden biriyle yeni bir havuz oluşturmak için yansıtılmış bir ZFS depolama havuzundan diskleri ayırmak için `zpool split` komutunu kullanabilirsiniz. Yeni havuz, orijinal yansıtılmış ZFS depolama havuzuyla aynı içeriğe sahip olacaktır.

Varsayılan olarak, yansıtılmış bir havuzdaki bir zpool bölme işlemi, yeni oluşturulan havuz için son diski ayırır. Bölme işleminden sonra yeni havuzu içe aktarmamız gerekmekte.

Örneğin 4 adet dosyadan oluşan bir havuz oluşturalım. Bu havuzda ilk iki sanal disk ile son iki sanal disk iki `mirror`umuzu ihtiva etsin ve her bir dosya da ayrılmış 4 GiB alanı bulunsun diyelim:

```text
~# zpool create tank mirror /tmp/file{1,2} mirror /tmp/file{3,4}
~# zpool status
  pool: tank
 state: ONLINE
config:

    NAME            STATE     READ WRITE CKSUM
    tank            ONLINE       0     0     0
      mirror-0      ONLINE       0     0     0
        /tmp/file1  ONLINE       0     0     0
        /tmp/file2  ONLINE       0     0     0
      mirror-1      ONLINE       0     0     0
        /tmp/file3  ONLINE       0     0     0
        /tmp/file4  ONLINE       0     0     0

errors: No known data errors
```

Şimdi bu iki diski `zpool split` ile ikiye ayıralım.

```text
~# zpool split tank tank2
```

Bu bizim `tank`'ımızı ayna diskleri esas alarak ikiye bölmemizi sağlar. Şimdi `tank2`yi import edelim ve neler olmuş olduğunu görelim.

```text
~# zpool import -d $(pwd)/file2 tank2
~# zpool status
  pool: tank
  state: ONLINE
  config:

    NAME          STATE     READ WRITE CKSUM
    tank          ONLINE       0     0     0
      /tmp/file1  ONLINE       0     0     0
      /tmp/file3  ONLINE       0     0     0

errors: No known data errors

  pool: tank2
  state: ONLINE
  config:

    NAME          STATE     READ WRITE CKSUM
    tank2         ONLINE       0     0     0
      /tmp/file2  ONLINE       0     0     0
      /tmp/file4  ONLINE       0     0     0

errors: No known data errors
```

Şimdi de havuzların boyutlarını kontrol edelim.

```text
~# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank   7.50G   130K  7.50G        -         -     0%     0%  1.00x    ONLINE  -
tank2  7.50G   184K  7.50G        -         -     0%     0%  1.00x    ONLINE  -
```

Biraz kafamız karıştı değil mi. Teknik açıdan bizim şu anda havuzlarımızın boyutları 4'de birine inmeli idi. Yani sonuçta 2 adet havuzda, 2 adet mirror var ve her bir mirrorda da 2 disk var. Toplamda 16 GiB boyutunda sanal disk olsa da her bir mirror için yalnızca bir sanal diskin boyutunu kullanabiliyorduk. Sonuç olarak bizim her bir havuzda sadece ~ 4 Gib boyutumuz olmalı idi. Ancak burda farklı bir durum var ve bu durum da şu. Yeni havuz, orijinal yansıtılmış ZFS depolama havuzuyla aynı içeriğe sahiptir. Varsayılan olarak, yansıtılmış bir havuzdaki yapılan bölme işlemi, yeni oluşturulan havuz için son diski ayırır. Aslında teknik olarak biz bunu yaparken elimizdeki havuzu ikiye bölmüş oluruz ve iki adet aynı dosyamız olur. Yani bir çeşit yedek gibi düşünebiliriz. İsterseniz başa sardırıp bir örnek vereyim.

```text
~# zpool create tank mirror /tmp/file{1,2} mirror /tmp/file{3,4}
~# zpool status
  pool: tank
 state: ONLINE
config:

    NAME            STATE     READ WRITE CKSUM
    tank            ONLINE       0     0     0
      mirror-0      ONLINE       0     0     0
        /tmp/file1  ONLINE       0     0     0
        /tmp/file2  ONLINE       0     0     0
      mirror-1      ONLINE       0     0     0
        /tmp/file3  ONLINE       0     0     0
        /tmp/file4  ONLINE       0     0     0

errors: No known data errors
```

Şimdi kök dizine bağlanmış olan bu havuzun içerisine bir dosya yazdıralım.

```text
~# dd if=/dev/zero of=/tank/denemedosyasi bs=1G  count=1
~# ls -ali /tank
2 -rw-r--r--.  1 root root 2147483648 Şub 25 16:49 denemedosyasi
~# zpool list
NAME    SIZE  ALLOC   FREE   CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank   7.50G  2.26G   5.24G        -         -     0%     0%  1.00x    ONLINE  -
```

Şimdi ikiye ayıralım ve dosyaların boyutlarını bir daha kontrol edelim

```text
~# zpool import -d $(pwd)/file2 tank2
~# zpool status
  pool: tank
  state: ONLINE
  config:

    NAME          STATE     READ WRITE CKSUM
    tank          ONLINE       0     0     0
      /tmp/file1  ONLINE       0     0     0
      /tmp/file3  ONLINE       0     0     0

errors: No known data errors

  pool: tank2
  state: ONLINE
  config:

    NAME          STATE     READ WRITE CKSUM
    tank2         ONLINE       0     0     0
      /tmp/file2  ONLINE       0     0     0
      /tmp/file4  ONLINE       0     0     0

errors: No known data errors
```

Şimdi de havuzların boyutlarını kontrol edelim.

```text
~# zpool list
NAME    SIZE  ALLOC   FREE   CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
tank   7.50G  2.26G   5.24G        -         -     0%     0%  1.00x    ONLINE  -
tank2  7.50G  2.26G   5.24G        -         -     0%     0%  1.00x    ONLINE  -
```

Bingoo. Şimdi de import yapılan dizini kontrol edelim.

```text
~# ls -ali /tank2
2 -rw-r--r--.  1 root root 2147483648 Şub 25 16:49 denemedosyası
```

Sonuç olarak 2 farklı disk havuzunda birbirinin kopyası iki dosya var. Peki bunlar aynalanmış disklerde olduğu gibi bağlı mı? Hadi test edelim.

```text
~# dd if=/dev/zero of=/tank/denemedosyasi2 bs=1G  count=1
~# ls -ali /tank
2 -rw-r--r--.  1 root root 2147483648 Şub 25 16:49 denemedosyasi2
2 -rw-r--r--.  1 root root 2147483648 Şub 25 16:49 denemedosyasi2
~# zpool list
tank   7.50G  3.00G   4.50G        -         -     0%    40%  1.00x    ONLINE  -
tank2  7.50G  2.26G   5.24G        -         -     0%     0%  1.00x    ONLINE  -
```

Sonuç olarak ayrılma tamamlandığı andan itibaren iki farklı disk elimizde bulunmakta. Herhangi bir bağ da yok.

Zpool bölme özelliğini kullanmadan önce aşağıdaki noktalara dikkat etmemiz gerekmekte:

* Bu özellik bir RAIDZ yapılandırması veya birden çok diskten oluşan yedekli olmayan bir havuz bölme işlemi için kullanılamaz.
* Bir havuz bölme işlemi denemeden önce veri ve uygulama işlemlerinin tamamlamnış olması gerekmekte.
* Diskin, ayrılmamış / kapatılmış disklere sahip olması bölme esnasında sorun çıkarabilir bu sebeple, diskin önbelleği temizle komutu yapılmalıdır.
* Yeniden serme işlemi `(resilvering)` devam ediyorsa bir havuz bölünemez.
* Bölünmüş bir havuz yeniden bölünemez. Örneğin:

```text
~# zpool split tank2 tank3
Unable to split tank2: Source pool must be composed only of mirrors
```

* Mevcut havuz üç yönlü bir aynaysa, yani 3 farklı disk içeren bir ayna ise yeni havuz bölünme işleminden sonra bir disk içerecektir. Var olan havuz iki diskin iki yönlü bir aynasıysa, oluşacak havuz iki diskten oluşan yedeksiz iki havuzdur. Yedekli olmayan havuzları aynalı havuzlara dönüştürmek için iki ek disk eklemeniz gerekecektir. Örneğin aşağıdaki disk için:

  \`\`\` ~\# zpool status pool: tank state: ONLINE config:

  NAME STATE READ WRITE CKSUM tank ONLINE 0 0 0 mirror-0 ONLINE 0 0 0 /tmp/file1 ONLINE 0 0 0 /tmp/file2 ONLINE 0 0 0 mirror-1 ONLINE 0 0 0 /tmp/file3 ONLINE 0 0 0

errors: No known data errors

```text
ikiye ayırma işlemi yapıldığında şöyle bir tablo oluşacaktır.
```

pool: tank state: ONLINE config:

```text
NAME            STATE     READ WRITE CKSUM
tank            ONLINE       0     0     0
  mirror-0      ONLINE       0     0     0
    /tmp/file1  ONLINE       0     0     0
    /tmp/file3  ONLINE       0     0     0
```

errors: No known data errors

pool: tank2 state: ONLINE config:

```text
NAME          STATE     READ WRITE CKSUM
tank2         ONLINE       0     0     0
  /tmp/file2  ONLINE       0     0     0
```

errors: No known data errors

\`\`\`

* Bölme işlemi sırasında verilerinizi yedekli tutmanın iyi bir yolu, üç diskten oluşan yansıtılmış bir depolama havuzunu bölmemiz durumunda, bölünme işleminden sonra orijinal havuzun iki yansıtılmış diskten oluşması sağlanmış olur böylece veri kaybı yaşanmaz.
* Bu işlemi yaptıktan sonra geri alamazsınız. Bu ZFS'nin bir dezavantajı olarak sayılabilir.


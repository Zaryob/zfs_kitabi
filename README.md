---
description: ZFS ve OpenZFS üzerine Türkçe bir kaynak oluşturma çabamın meyvesidir.
---

# ZFS 101: Giriş, Yanış ve Sonuç

## ZFS nedir?

OpenZFS üzerine türkçe kaynak

**ZFS** ya da eski adı ile **Zettabyte File System**, ilk olarak [Sun Microsystems](https://www.oracle.com/sun/) şirketi tarafından [Solaris işletim sistemi](https://www.oracle.com/solaris/solaris11/) için özelleştirilmiş bir dosya sistemi oluşturmak amacı ile ortaya çıktı. Sun firmasında çalıştıkları sırada Jeff Bonwick, Bill Moore ve Matthew Ahrens tarafından 2001 yılında başlanılan ZFS dosya sistemi ilk olarak 14 Eylül 2004 yılında duyruldu ve Solaris'e 31 Ekim 2005 yılında dahil edildi.

2005 yılı Haziran ayında Sun Microsystems'in kaynak kod havuzunu CDDL lisansı ile paylaşmasının ardından OpenSolaris bileşenleri ile beraber kaynak kodu açık kaynak olarak paylaşıldı. Bu süreçte BSD, Linux ve Darwin çekirdeklerine portlanarak kullanılmaya başlandı.

2010 yılında Sun Microsystems'in Oracle tarafından alt şirket olarak satın alınmasının ardından ZFS ve Solaris kaynak havuzunda yer alan kodlar kapatıldı. İlerleyen dönemde ZFS'nin yaratıcılarının da aralarında bulunduğu bir grup geliştiricinin Oracle/Sun dan ayrılmasının ardından ZFS disk tipinin açık kaynak kodlu devamı niteliğinde OpenZFS projesi çıkarılmıştır. Şu an ZFS iki ayrı koldan geliştirilmeye devam edilmektedir.

## ZFS'nin Özellikleri

2001 yılında ZFS sadece bir disk yönetim sistemi ihtiyacından dolayı değil mevcut mimarinin dezavantajlarını kapatacak yeni bir format geliştirmek idi.

ZFS disk yapısının getirdiği sistem mevcut disk yapıları ve RAID dizilerinden farklılık gösterir. Birden fazla diski ve sanal aygıtları \(**virtual devices - vdev**\) aynı anda bir disk havuzu **\(zpool\)** üzerinden birleştirir.

ZFS, "128 bit" bir dosya sistemidir, yani 128 bit, içindeki herhangi bir birim için en büyük boyutlu adresdir. Bu boyut, öngörülebilir gelecekte herhangi bir zamanda sınırlandırılması muhtemel olmayan kapasitelere ve boyutlara izin verir. Örneğin, uyguladığı teorik sınırlar dizin başına 2^48 giriş, maksimum dosya boyutu 16 EB \(2 ^ 64 veya ~ 16 \* 2 ^ 18 bayt\) ve "**zpool**" başına maksimum 2^64 sanal aygıtı içerebilir. 

ZFS 128-bit adresleme şeması sayesinde 256 katrilyon zettabayt depolayabilir, bu da **1000 PB \(petabayt\)** depolama kapasitesini aşan ölçeklenebilir bir dosya sistemine dönüşürken, tekli veya çoklu ZFS'nin Z-RAID dizilerinde yönetilmesine izin verir.

ZFS geleneksel yöntemleri, dosya sistemi katmanlarına adapte eden yöntemlerle beraber kullanır. Bunu yaparken çeşitli mekanizmalar geliştirmiştir. ZFS'nin en büyük başarısından birisi [copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write) mekanizmasıdır ki bu mekanizma dosya yönetim sisteminde meydana gelebilecek her türlü bozulma ve kesintiye karşı dosyaların güvenli bir şekilde tutulabilmesini sağlar.

Tüm verilerin ve meta verilerin hiyerarşisini, hiyerarşik sağlama toplamlarını \(checksum\) bloğun kendisinin üzerinde değil bir ana bloğun üzerinde tutarak, ana bloğu kullanarak alt bloklardakş verilerin kontrol edilmesini, kurtarılmasını ve geriye çevrilmesini sağlar.

Bazı durumlarda, bir hata veya tutarsızlık durumunda dosya sisteminde ve verilerde yapılan son değişikliklerin otomatik olarak geri alınmasını sağlayan **snapshot** özelliği bulunmaktadır.

Aynı zamanda verilerin yerel olarak sıkıştırma ve tekilleştirme işini de veri iletimi sırasında yaparak diski daha optimize kullanmaktadır. 

Bir diğer önemli özelliğine **vdev**'leri ayna sistem olarak kullanma özelliğidir. Aynalama yapılan disklerdeki verileri birbiri arasında eşzamanlı olarak korumaya yardımcı olur.






---
description: ZFS ve OpenZFS üzerine Türkçe bir kaynak oluşturma çabamın meyvesidir.
---

# Giriş

## Lisans

```text
Copyright (C) 2020 Süleyman POYRAZ <zaryob.dev@gmail.com>.

İşbu belgeyi kopyalama, dağıtma ve/veya değiştirme hakkı, Özgür Yazılım Vakfı tarafından yayımlanan GNU Özgür Belgelendirme Lisansı Sürüm 1.3 ya da daha sonraki sürümler kapsamında verilmiş olup; Değişmeyen Kısımlar, Ön Kapak Metinleri ve Arka Kapak Metinleri’ni içermez. İşbu lisansın bir kopyası “GNU Özgür Belgelendirme Lisansı” bölümünde mevcuttur.
```

## Bu Belgeleme Hakkında

Bu belgeleme bir harmanlama projesidir. **ZFS** hakkında yazılmış bütün kaynakları inceleyerek, Türkçe bir kaynak oluşturma çabasına girmemin bir ürünüdür.

Bu belgelendirmede yararlanılan kaynaklar Kaynakça bölümünde belirtilecektir.

Ayrıca belgelendirmede sıkça kullanılan **ZFS** ibaresi disk yönetim sistemi olan ve NetApp tarafından [Write Anywhere File Layout](https://en.wikipedia.org/wiki/Write_Anywhere_File_Layout) ile patentlenen dosya sistemini açıklamaktadır. Kullanılan araçlar, açık kaynak kodlu olarak devam eden **OpenZFS**'yi işaret etmektedir.

## ZFS nedir?

**ZFS** ya da eski adı ile **Zettabyte File System**, ilk olarak [Sun Microsystems](https://www.oracle.com/sun/) şirketi tarafından [Solaris işletim sistemi](https://www.oracle.com/solaris/solaris11/) için özelleştirilmiş bir dosya sistemi oluşturmak amacı ile ortaya çıktı. Sun firmasında çalıştıkları sırada Jeff Bonwick, Bill Moore ve Matthew Ahrens tarafından 2001 yılında başlanılan ZFS dosya sistemi ilk olarak 14 Eylül 2004 yılında duyruldu ve Solaris'e 31 Ekim 2005 yılında dahil edildi.

2005 yılı Haziran ayında Sun Microsystems'in kaynak kod havuzunu **CDDL 1.0** lisansı ile paylaşmasının ardından OpenSolaris bileşenleri ile beraber kaynak kodu açık kaynak olarak paylaşıldı. Bu süreçte BSD, Linux ve Darwin çekirdeklerine portlanarak kullanılmaya başlandı.

2010 yılında Sun Microsystems'in Oracle tarafından alt şirket olarak satın alınmasının ardından ZFS ve Solaris kaynak havuzunda yer alan kodlar kapatıldı. İlerleyen dönemde ZFS'nin yaratıcılarının da aralarında bulunduğu bir grup geliştiricinin Oracle/Sun dan ayrılmasının ardından ZFS disk tipinin açık kaynak kodlu devamı niteliğinde OpenZFS projesi çıkarılmıştır. Şu an ZFS iki ayrı koldan geliştirilmeye devam edilmektedir.


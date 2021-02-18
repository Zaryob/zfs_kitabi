# ZFS'yi Gerçek Ortamda Kullanmak

## FreeBSD üzerinde ZFS dosya sistemini kullanmak

FreeBSD öntanımlı olarak iki dosya sistemi kullanmaktadır. Bunlardan birincisi **UFS** diğeri ise **ZFS'**dir. Benim kişisel olarak kullandığım FreeBSD sürümlerinden 11 ve sonrası sürümleri ZFS dosya sistemini direk kernel kodu ile getirmekte.



FreeBSD ve türevlerinde tek yapmamız gerek `/etc/rc.conf` dosyasını düzenlemek. Bunun için aşağıdaki komutu kullanmanız yeterli:

```text
$ echo "zfs_enable=\"YES\"" >>/etc/rc.conf
```

## Linux distrolarında ZFS kullanmak



#### ZFS on Root \(Kök sisteminde ZFS Kullanmak\)




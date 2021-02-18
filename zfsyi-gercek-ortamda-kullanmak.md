# ZFS'yi Gerçek Ortamda Kullanmak

## ZFS'yi Gerçek Ortamda Kullanmak

 ZFS'nin Kurulumu ZFS daha öncesinde belirttiğim gibi Oracle tarafın yürütülen Solaris ZFS ve açık kaynak kodlu olarak devam ettirilen OpenZFS projesi ile devam etmekte. Bu kitapta esas olarak OpenZFS kurulumu anlatılmıştır.

### FreeBSD Üzerinde ZFS Kullanmak 

FreeBSD öntanımlı olarak iki dosya sistemi kullanmaktadır. Bunlardan birincisi UFS diğeri ise ZFS'dir. ZFS dosya sistemi kurulum aşamasında hedef dosya yöneticisi olarak kullanılabilir. Benim kişisel olarak kullandığım FreeBSD sürümlerinden 11 ve sonrası sürümleri ZFS dosya sistemini direk kernel kodu ile getirmekte. FreeBSD ve türevlerinde tek yapmamız gerek `/etc/rc.conf` dosyasını düzenlemek. Bunun için aşağıdaki komutu kullanmanız yeterli: 

```text
# echo "zfs_enable=\"YES\"" >>/etc/rc.conf 
```

### Linux Distrolarında ZFS Kullanmak

 OpenZFS FreeBSD haricinde Linux çekirdeği için de modul destei getirmekte ancak Linus Torvalds bu modulu açıkça linux çekirdeğinin içerisine eklemedi. Bunun da bazı sebepleri var. 

[ItFoss](https://itsfoss.com/linus-torvalds-zfs) haber sitesinde de paylaşılan Linus Torvalds'ın açıklamasında, başta anlattığım yazılım lisansının Linux kernelinin lisansı olan GPL ile uygun olmaması sebebiyle bu modül maalesef resmi olarak eklenmedi ve Torvalds bunu eklemeye de bir miktar imtina ediyor. Bunun da bazı sebepleri elbette var.

Bu sebepleri bir yana bırakırsak ZFS paketi pek çok Linux dağıtımında da paketlenmiş ve ZFSonLinux organizasyonun sağladığı depolarla veya resmi depolarla dağıtılmakta.ZFSonLinux web sitesinden kendi sisteminiz için sağlanmış dağıtım paketlerini elde edebilirsiniz. Ancak bu kılavuzda Debian-Ubuntu ve Fedora için nasıl kurulacağı aktarılmıştır. 

#### Debian/Ubuntu Dağıtımlarında OpenZFS Kurulumu

 Debian ve Ubuntu dağıtımları için aşağıdaki adımları kullanarak kurulum yapabilirsiniz. Bu işlemleri yaparken root kullanıcısı olmayı unutmayın.

 Paketleri debian ve ubuntunun depolarından aynı isimle bulabilirsiniz ve dağıtımınızın ana deposundan kurabilirsiniz.

```text
~# apt install --yes --no-install-recommends zfs-dkms 
~# apt install --yes zfsutils-linux 
```

 Bu aşamaların bitmesinin ardından zfs kernel modülünü etkinleştirmeniz gerekmekte: 

```text
~# modprobe zfs 
```

Bu aşamanın ardından zfs komutunu kullanabilirsiniz. 

#### Fedora için OpenZFS Kurulumu 

Başlangıç için ZFS deposunu Fedora sistemimize ekleyelim. 

```text
~# dnf install https://zfsonlinux.org/fedora/zfs-release$(rpm -E %dist).noarch.rpm
~# gpg --import --import-options show-only /etc/pki/rpm-gpg/RPM-GPG-KEY-zfsonlinux
pub   rsa2048 2013-03-21 [SC]
      C93AFFFD9F3F7B03C310CEB6A9D5A1C0F14AB620
uid                      ZFS on Linux <zfs@zfsonlinux.org>
sub   rsa2048 2013-03-21 [E]
```

ZFS modülünü kuralım ve modülü aktive edelim. 

Eğer ki sisteminizde öncesinde zfs-fuse paketi kurulu ise zfs paketini kurmak için zfs-fuse yerine zfs paketini seçelim. 

```text
~# dnf swap zfs-fuse zfs 
```

Şimdi ZFS deposundan zfs paketini kuralım ve modülü aktive edelim. 

```text
~# dnf install zfs 
~# modprobe zfs 
```

### MacOS X Üzerinde ZFS Kullanmak

[ ZFS on OS X](https://openzfsonosx.org/wiki/Downloads) indirme sayfası üzerinden işletim sisteminize uygun paketi seçerek kurmanız yeterlidir.

### Windows Üzerinde ZFS Kullanmak

Windows üzerinde ZFS için bir test yapmadım ancak [ZFS On Windows](https://openzfsonwindows.org/) projesi ile kurulumun mümkün olduğu belirtilmiş. [ZFS on Windows github sayfasından](https://github.com/openzfsonwindows/ZFSin/releases) `OpenZFSOnWindows-release-*.exe` kurucusunu kullanarak kurabilirsiniz.


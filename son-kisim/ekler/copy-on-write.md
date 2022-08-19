---
description: ZFS Disk Hiyerarşisini anlamadan önce Copy-on-Write uygulamasının ve ZFS'de bu uygulamanın nasıl vücut bulduğundan bahsedelim
---


# Copy-On-Write Nedir?

Yazma üzerine kopyalama, başka bir ilginç (ve harika) özelliktir. Çoğu dosya sisteminde, verilerin üzerine yazıldığında sonsuza kadar kaybolur. ZFS'de yeni bilgiler farklı bir bloğa yazılır. Yazma tamamlandığında, dosya sistemi meta verileri yeni bilgileri gösterecek şekilde güncellenir. Bu, yazma sırasında sistem çökerse (veya başka bir şey olursa), eski verilerin korunmasını sağlar. Bu aynı zamanda bir sistem çökmesinden sonra sistemin fsck çalıştırmasına gerek olmadığı anlamına gelir.

Copy-on-write özelliğinin ana kullanımını, işletim sistemlerinin işlemler için sanal bellek yönetimi yaparken, sistem çağrılarının ve bellek rezervasyon işlemlerinin kullandığını görürüz. Fork ile oluşturulan işlemin kendi adres alanı vardır. Bu yeni işlem için bir adres alanı oluşturulur. Normalde bu rezervasyon, kök işlemin düşmesi ile rezerve alan silinirek kullanıma açılır. Copy-on-write özelliği burada silme işlemi ile vakit kaybetmemek için kullanırken görürüz.

Kök işlemin, ve child processin kullandıkları kaynak ortaktır taki bu kaynak üzerinden bir değişiklik veya yazma işlemi yapılıncaya kadar. Yazma işlemi yapıldığı anda bir kopya oluşturulur ve önceden oluşturulan adres alanına ebeveyn processin hafızası kopyalanır. Böylece yazma işlemi gerçekleşene kadar kopyalama işlemi yapılmaz. Ortak bir hafıza alanı kullanılır.

Ortak hafıza alanı bütün işlemlerin tamamlanması ile sona erdirilir.

Copy on Write stratejisinin disk alanlandırılmasında kullanılması diski büyük ölçüde parçalar. Bunun büyük performans etkileri olabilir. Bu nedenle, parçalanmayı en aza indirmek için blokları önceden tahsis etmek için bazı çalışmalar yapılması gerekir. İki temel yaklaşım vardır: uzantıları önceden tahsis etmek için bir b-ağacı kullanmak veya bir döşeme yaklaşımı kullanmak, kopya için disk plakalarını işaretlemek. ZFS, Btrfs'nin de kullandığı b-tree ağaç yaklaşımını kullanarak veri döşemesi yapmaktadır..
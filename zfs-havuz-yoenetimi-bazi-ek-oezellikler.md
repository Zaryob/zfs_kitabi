---
description: ZFS Havuz Yönetimi Komutları
---


# ZFS Havuz Yönetimi - Bazı Ek Havuz Yönetimi Özellikleri

Bu kısımda önceki öğrendiğimiz komutların haricinde bazı ek ZFS komutlarına örnekler verecek ve bunlara ait örnek komutlar vereceğim.
Burada çoğunlukla kullanmayacağınız ama yine de önemli olan bazı ZFS özelliklerinden bahsedeceğim.

## Yansıtılmış bir ZFS Depolama Havuzunu Bölerek Yeni Bir Havuz Oluşturma

Yansıtılmış bir ZFS depolama havuzu, zpool split komutu kullanılarak hızlı bir şekilde yedekleme havuzu olarak klonlanabilir.

Şu anda bu özellik, yansıtılmış bir kök havuzunu bölmek için kullanılamaz.

Ayrılmış disklerden biriyle yeni bir havuz oluşturmak için yansıtılmış bir ZFS depolama havuzundan diskleri ayırmak için `zpool split` komutunu kullanabilirsiniz. Yeni havuz, orijinal yansıtılmış ZFS depolama havuzuyla aynı içeriğe sahip olacaktır.

Varsayılan olarak, yansıtılmış bir havuzdaki bir zpool bölme işlemi, yeni oluşturulan havuz için son diski ayırır. Bölme işleminden sonra yeni havuzu içe aktarın. Örneğin:
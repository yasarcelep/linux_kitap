# traceroute

traceroute komutu bir sunucuya erişene kadar gidilen yolu tespit etmeye yarar. Bunun için bu yol boyunca uğradığı bütün sunuculara ICMP paketleri gönderir. Van Jacobson traceroute programını 1987'de geliştirmiştir ve aslında ping'in yaptığından farklı neredeyse hiçbir şey yoktur. Sadece ping'i doğru şekilde kullanarak çok faydalı bir araç geliştirmiştir. İlerleyen yıllarda Mike Muss \(ping'in gelişricisi\) bu kullanım şeklini atladığını, bu fikri geliştirdiği için de Van Jacobson'ı resmen kıskandığını ifade etmiştir.

> I was insanely jealous when Van Jacobson of LBL used my kernel ICMP support to write TRACEROUTE, by realizing that he could get ICMP Time-to-Live Exceeded messages when pinging by modulating the IP time to life \(TTL\) field. I wish I had thought of that! :-\) Of course, the real traceroute uses UDP datagrams because routers aren't supposed to generate ICMP error messages for ICMP messages.

Nasıl çalıştığını anlamak için ping komutunda karşımıza çıkan \(ancak üstünde durmadığımız\) TTL \(Time To Live\) mekanizmasını anlamak gerekir.

## TTL

TTL basitçe, gönderdiğimiz paketin kaç noktadan geçmesine izin verdiğimizi ifade eder. Ping programını incelerken kullandığımız örneklerde genellikle bu değerin 54 olduğunu gördük. Bu şu anlama geliyor, "bu paket en fazla 54 noktadan geçebilir, daha fazlasına iletilmemeli."

Ağ'daki \(internet veya yerel ağ, fark etmez\) her nokta \(sunucu, router vb.\) kendisine gelen paketin TTL'ine bakar, eğer bu değer 1'den büyükse, değeri 1 azaltıp bir sonraki  noktaya iletir, eğer değer 1'se, o zaman paketi göz ardı, eder, ICMP hata pakedi gönderir ve bu bilgi içerisinde kendi IP adresi de yer alır.

Öyleyse uzaktaki bir sunucuya ping atarken, eğer TTL değerimizi aradaki router sayısından daha küçük tutarsak, bu paket hedef sunucuya asla ulaşmaz, ancak arada bir noktadan cevap alırız.

Örneğin google.com sunucusuna ping atmak istersek, ancak TTL değerini 1 tutarsak,

```bash
eaydin@dixon ~ $ ping -t 1 google.com
PING google.com (216.58.208.110) 56(84) bytes of data.
From 192.168.100.1 icmp_seq=1 Time to live exceeded
```

TTL değerimizi 2'ye çıkardığımızda,

```bash
eaydin@dixon ~ $ ping -t 2 google.com
PING google.com (216.58.208.110) 56(84) bytes of data.
From 81.212.171.130.static.turktelekom.com.tr (81.212.171.130) icmp_seq=1 Time to live exceeded
```

Giderek artırırsak, Google'a ulaşana kadar geçtiğimiz yolu deşifre edebiliriz...

```bash
eaydin@dixon ~ $ ping -t 3 google.com
PING google.com (216.58.208.110) 56(84) bytes of data.
From 93.155.0.184 icmp_seq=1 Time to live exceeded
```

```bash
eaydin@dixon ~ $ ping -t 4 google.com
PING google.com (216.58.208.110) 56(84) bytes of data.
From 81.212.106.225.static.turktelekom.com.tr (81.212.106.225) icmp_seq=4 Time to live exceeded
```

```bash
eaydin@dixon ~ $ ping -t 5 google.com
PING google.com (216.58.208.110) 56(84) bytes of data.
From 195.175.173.172.06-ulus-xrs-t2-2.06-ulus-t3-7.statik.turktelekom.com.tr (195.175.173.172) icmp_seq=1 Time to live exceeded
```

Yukarıdaki örneklerde, Google'a ping pakedi \(ICMP echo\) gönderdik, ancak TTL'ini sırayla 3, 4, 5 yaptık. Yani "paket yola çıksın ama 3 router/hop noktası geçebilsin en fazla" dedik. 4. hata noktasında da bize cevap gönderdi. Aynı işlemi birer birer artırarak her hata aldığımız router/hop noktasından cevap toplamış olduk.

Traceroute programı da aslında arka planda bu mantığı uygular. Sırasıyla ICMP paketlerimizin TTL'ini artırarak ilgili noktaya ulaşana kadar bu işi tekrar eder.

## Kullanımı

Kullanımı gayet basittir ve çıktısı aşağıdaki gibidir.

```bash
eaydin@dixon ~ $ traceroute google.com
traceroute to google.com (216.58.209.14), 30 hops max, 60 byte packets
 1  192.168.100.1 (192.168.100.1)  8.191 ms  8.391 ms  8.403 ms
 2  81.212.171.130.static.turktelekom.com.tr (81.212.171.130)  42.977 ms  45.323 ms  45.343 ms
 3  93.155.0.184 (93.155.0.184)  45.342 ms  45.360 ms  45.368 ms
 4  81.212.106.225.static.turktelekom.com.tr (81.212.106.225)  45.373 ms  45.377 ms  45.401 ms
 5  195.175.174.42.06-ulus-xrs-t2-1.06-ulus-t3-7.statik.turktelekom.com.tr (195.175.174.42)  45.404 ms  45.414 ms  45.420 ms
 6  195.175.166.204.00-gayrettepe-xrs-t2-1.06-ulus-xrs-t2-1.statik.turktelekom.com.tr (195.175.166.204)  5123.341 ms * *
 7  * * *
 8  * * *
 9  * 209.85.250.69 (209.85.250.69)  74.476 ms  76.287 ms
10  209.85.142.189 (209.85.142.189)  76.328 ms  76.329 ms  76.330 ms
11  sof01s12-in-f14.1e100.net (216.58.209.14)  76.291 ms  52.555 ms  52.308 ms
```

Yukarıdaki çıktıyı incelediğimizde, 11 noktadan sonra Google'a erişebildiğimiz görüyoruz. Gerçekten de TTL 11 ile ping atarsak, Google'un cevap vereceğini görebiliriz.

```bash
eaydin@dixon ~ $ ping -t 11 google.com
PING google.com (216.58.209.14) 56(84) bytes of data.
64 bytes from sof01s12-in-f14.1e100.net (216.58.209.14): icmp_seq=1 ttl=54 time=51.5 ms
```

Ancak arada bazı noktalarda "\*" işaretleri mevcut. Bu işaretler, ilgili sunucuya giderken pinglenen noktaların cevap vermediği, veya DNS çözümlemesinde hata yaşandığı gibi pek çok şeyi ifade edebilir.

Farkındaysanız 9. satırda başta bir "\*" işareti var, sonra satır normal devam ediyor. Bunun sebebi, traceroute her noktayı pinglerken, 3 defa deniyor. Belli ki denemelerinden birinde başarısız olmuş. Her satır için 3 farklı ms değeri olması, ancak 9. satırda sadece 2 farklı ms değeri olması da bundan kaynaklanmaktadır.

Her nokta \(hop\) için yapılacak pingleme \(query\) miktarını `-q` parametresiyle değiştirebilirsiniz. Ayrıca hostname çözümlemesi yapmayıp, doğrudan çıktıda IP adreslerinin görünmesini de `-n` ile sağlayabilirsiniz.

```bash
eaydin@dixon ~ $ traceroute google.com -n -q 1
traceroute to google.com (216.58.211.14), 30 hops max, 60 byte packets
 1  192.168.100.1  5.363 ms
 2  81.212.171.130  33.504 ms
 3  93.155.0.184  35.415 ms
 4  81.212.106.225  35.994 ms
 5  195.175.174.42  37.083 ms
 6  *
 7  195.175.174.59  45.340 ms
 8  72.14.197.192  53.157 ms
 9  209.85.248.54  60.998 ms
10  64.233.175.34  76.644 ms
11  216.239.47.189  82.119 ms
12  216.58.211.14  77.069 ms
```

Traceroute algoritmasında TTL değerinin 1'den başladığını ifade etmiştik. Dilerseniz bu değeri 1'den başlatmak yerine farklı değerle taramaya başlayabilirsiniz.

```bash
eaydin@dixon ~ $ traceroute google.com -n -f 5
traceroute to google.com (216.58.211.14), 30 hops max, 60 byte packets
 5  195.175.174.42  34.707 ms  35.731 ms  37.701 ms
 6  195.175.166.204  44.291 ms  44.878 ms  44.895 ms
 7  * * *
 8  72.14.197.194  55.306 ms 72.14.197.192  55.963 ms  55.967 ms
 9  209.85.248.54  65.153 ms  67.279 ms  69.913 ms
10  64.233.175.34  78.079 ms  68.314 ms  76.712 ms
11  216.239.47.189  79.059 ms  88.210 ms  89.261 ms
12  216.58.211.14  69.281 ms  78.807 ms  78.197 ms
```

## ICMP, UDP ve TCP Kullanımı

traceroute'un çalışma prensibini anlamak için ping komutunu farklı TTL'ler kullanarak denedik. Aslında traceroute komutu, parametre belirtilmediğinde ping atarak bu işlemi gerçekleştirmez, ancak mekanizmanın rahat anlaşılması için bu yöntemi tercih ettik.

traceroute programı, parametre kullanılmadığında UDP ile yol çıkarmaya çalışır. Bunu nasıl yaptığını ve sebebini aşağıda inceleyeceğiz, ancak traceroute ile istediğimiz metodu kullanarak yol çıkarmak mümkün, yeter ki ağ yapısı buna müsaade etsin.

Bağlantı tipini belirlemek için `-M` \(method\) parametresi kullanılır. Örneğin TCP yöntemini seçmek için

```bash
traceroute google.com -M tcp
```

**NOT:** MS Windows sistemlerde traceroute programı `tracert` ismiyle bulunur. Bunun sebebi, eski DOS sistemlerinde dosya isimlerine getirilen kısıtlamadır. Eski DOS sistemlerinde dosya adları en fazla 8 karakter olabilir, dosya uzantıları ise 3 karakter olabilirdi. Bunun için programı `tracert.exe` olarak isimlendirmişlerdir. Bu programın bir diğer farklılığı, standart tarama mekanizması olarak UDP değil, ICMP paketleri kullanmasıdır.

### UDP ile Kullanımı

Daha önce belirttiğimiz gibi, traceroute programı GNU/Linux üzerindeki dağıtımında parametre belirtilmeyince bu tekniği kullanır. UDP taramasıyla program, karşı tarafın UDP üzerinde bir servis çalıştığı "düşünülmeyen" portlarına datagram gönderir. Eğer karşı tarafta gerçekten bu portlarda bir servis çalışmıyorsa, cevap olarak "Destination Port Unreachable" içerikli bir ICMP paketi gönderir.

traceroute UDP üzerinden tarama yaparken, 33434 portundan başlar, her hop'ta değeri bir artırır.

### ICMP ile Kullanımı

Bu yöntemi kullanmak için `-M icmp` veya `-I` parametreleri tanımlanır. traceroute programının algoritmasını anlatırken belirttiğimiz gibi ICMP paketleri kullanır, özetle karşı tarafı ping'leyebiliyorsanız, bu yöntemi kullanabilirsiniz.

### TCP ile Kullanımı

Aslında çok sık kullanılması gereken bu yöntem, biraz tecrübe gerektirdiğinden standart olarak sunulmaz, bu durum bir çok kişinin yol çıkarma işlemini doğru yapamamasına sebep olur. Pek çok firewall UDP paketlerini belirli port aralığı haricinde \(DNS vb.\) engeller, ICMP paketlerinin de engellenmesi çok sık karşılaşılan bir durumdur. Ancak ağ yöneticilerinin neredeyse her zaman izin verdiği bir takım servisler TCP üzerinden çalışır. Örneğin tarayacağımız ağ üzerinde çok büyük ihtimalle HTTP protokolünün standart portu olan TCP 80 izin veriliyordur. Bu durumdan faydalanarak firewall engellerini aşabiliriz. 80 en yaygın kullanılan olduğu için, standart port olarak belirlenmiş, ancak farklı port belirtmek de mümkün.

```bash
traceroute google.com -T -p 453
```

TCP taraması yapılırken, karşı tarafta bir bağlantı açmış olmayız. Normal şartlar altında bir TCP bağlantısı şu şekilde gerçekleştirilir:

1. Karşı tarafa SYN paketi gönderilir
2. Karşı taraftan SYN+ACK paketi alınır
3. Karşı tarafa ACK paketi gönderilir

Yukarıda tanımlanan "three-way handshake" ile iki sunucu arasında bir TCP bağlantısı kurulmuş olur. Son aşamada, ACK paketi geldikten sonra bağlantı başlayacağından, sunucu üzerindeki bir yazılım ancak bu aşamada bir bağlantı geldiğinden haberdar olur.

Ancak traceroute aşağıdaki yöntemi izler.

1. Karşı tarafa SYN paketi gönderilir
2. Karşı taraftan SYN+ACK paketi alınır
3. Karşı tarafa RST paketi gönderilir

Son aşamada RST paketi gönderdiğimiz için karşı taraftaki hiçbir yazılım bizim SYN paketi veya RST paketi gönderdiğimizi görmez. Ancak network trafiğinin detaylı analiziyle \(veya firewall yardımıyla\) bu mümkün olur. Traceroute programının kullandığı bu bağlantı biçmine "half-open" \(yarı-açık\) teknik denilir.

"Half-open" tekniği yerine programın gerçek bir TCP bağlantısı sunan, yani SYN -&gt; SYN+ACK -&gt; ACK şeklinde bağlantı kuran parametresi de mevcuttur. `-M tcpconn` ile bu sağlanabilir. Ancak bu durum tavsiye edilmez, hem karşı tarafta dinleyen servis gelen rastgele pakete ne cevap vereceğini bilmeyebilir, hem de application seviyesinde bağlantı kurduğunuz karşı tarafın loglarında yer alır.


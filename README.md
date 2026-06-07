# BİL 304 - ELF Analizi ve Araştırma Raporu

## 1. Dosyanın ELF Sınıfı, Mimarisi ve Giriş Adresi
Dosyanın başlık (header) bilgilerini okumak için terminalde `msp430-readelf -h udp-client.z1` komutunu çalıştırdım. Çıkan tabloyu incelediğimde temel yapısal özellikler şu şekilde:

* **ELF Sınıfı (Class):** Çıktıda `ELF32` değerini görüyoruz. Z1 donanımındaki MSP430 mikrodenetleyicisi aslında 16-bit bir mimariye sahip ancak araç zinciri (toolchain) derleme yaparken standart bir bellek haritalaması sunmak için dosyayı 32-bit ELF formatında oluşturmuş.
* **Mimari (Machine):** Cihaz hedefi `Texas Instruments msp430 microcontroller` olarak belirtilmiş. Bu da derlenen imajın doğrudan MSP430'un komut setiyle (instruction set) uyumlu olduğunu ve başka bir mimaride koşamayacağını doğruluyor.
* **Giriş Adresi (Entry Point):** Çıktıda giriş adresi `0x3100` olarak verilmiş. Bu adres oldukça kritik; çünkü cihaza enerji verildiğinde veya boot/reset işlemi yapıldığında işlemci doğrudan Flash bellekteki bu adrese atlayıp kodu oradan çalıştırmaya başlayacak.



## 2. Temel Bölümlerin (.text, .data, .bss, .rodata, .vectors) Varlığı ve Anlamı
Dosyadaki bölümleri incelemek için `msp430-readelf -S udp-client.z1` komutunu çalıştırdım. Çıkan section tablosu, yazdığımız kodun ve değişkenlerin donanım belleğine nasıl dağıtılacağını çok net gösteriyor:

* **.text Bölümü:** Tabloda `0x3100` adresinden başladığını ve `AX` (Alloc, Execute) bayraklarına sahip olduğunu görüyoruz. Burası aslında bizim yazdığımız programın derlenmiş makine dili komutlarını barındırıyor. Doğrudan Flash (ROM) belleğe yazılıyor ve işlemci kodları buradan okuyup çalıştırıyor.
* **.rodata Bölümü:** Tablodaki `A` (Alloc) bayrağından anlıyoruz ki burası sadece okunabilir (Read-Only) verilerin tutulduğu kısım. Kod içindeki sabitler (const) veya ekrana bastığımız string mesajlar burada duruyor. Değiştirilmeyeceği için RAM'de boşuna yer kaplamaması adına Flash bellekte muhafaza ediliyor.
* **.data Bölümü:** `WA` (Write, Alloc) bayraklarına sahip. Kod yazarken başlangıç değeri atadığımız global veya statik değişkenler (örneğin `int sayac = 5;`) bu bölümde tutuluyor. Çalışma zamanında (runtime) değerleri değişebileceği için RAM'e yükleniyorlar.
* **.bss Bölümü:** Tabloda tipinin `NOBITS` olduğunu görmek çok mantıklı. Çünkü burası başlangıç değeri atanmamış global/statik değişkenler (örneğin `int toplam;`) için ayrılan yer. Firmware dosyasının içinde (disk/flash üzerinde) boşuna yer kaplamıyor; sadece bir nevi "yer tutucu" görevi görüyor. Cihaz boot edildiğinde işletim sistemi RAM'de bu değişkenler için yer ayırıp içlerini sıfırla dolduruyor.
* **.vectors Bölümü:** `0xffc0` adresine yerleşen bu özel kısım, donanım kesme (interrupt) vektörlerini tutuyor. Örneğin bir timer dolduğunda veya ağdan paket geldiğinde, işlemcinin o anki işini bırakıp kodda hangi adrese sıçraması gerektiğini gösteren yönlendirme tablosu burada saklanıyor.




## 3. Kod ve Veri Boyutları
Programın cihaz hafızasında ne kadar yer kaplayacağını görmek için `msp430-size udp-client.z1` komutunu kullandım. Çıktıdaki sayılar bize RAM ve Flash (ROM) kullanımının net bir özetini veriyor:

* **text (42542 bayt):** Derlenen kodların ve sabit değerlerin kapladığı alan. Bu kısım doğrudan cihazın Flash belleğine yazılıyor.
* **data (336 bayt):** Başlangıç değeri atadığımız değişkenlerin boyutu. Bu veriler cihaz kapalıyken Flash'ta saklanıyor, cihaz boot edildiğinde ise RAM'e kopyalanıyor.
* **bss (5888 bayt):** Başlangıç değeri olmayan değişkenler için çalışma zamanında (runtime) RAM'de ayrılan boş alan.
* **dec (48766 bayt):** Programın bellekteki toplam ayak izi (text + data + bss).

Bu sayılara bakarak cihaz üzerindeki asıl yükümüzü hesaplarsak; bu program Z1 cihazının **Flash belleğinde 42.878 bayt** (text + data, yani yaklaşık 41.8 KB), **RAM üzerinde ise 6.224 bayt** (data + bss, yaklaşık 6 KB) yer kaplayacak. Z1'in 92 KB Flash kapasitesi olduğunu düşünürsek, hafızanın yarısından azını kullanmışız; yani donanım sınırları açısından oldukça güvenli bir noktadayız diyebilirim.




## 4. Sembol Tablosu ve Anlamlı Semboller
Derlenmiş dosyanın içindeki fonksiyon ve değişkenlerin bellek adreslerini haritalamak için `msp430-nm -n udp-client.z1` komutunu çalıştırdım. Çıktı çok uzun bir liste olduğu için özellikle projemizin yapısını ele veren bazı kritik sembolleri inceledim:

* **'T' ve 't' (Text) Sembolleri:** Çıktıda en çok yer kaplayan bu semboller, Flash bellekteki fonksiyonlarımızı gösteriyor. Listede `uip_udp_packet_send`, `uip_process` ve `uip_icmp6_input` gibi fonksiyonları açıkça görebiliyoruz. Bu da bize cihazın içinde tam teşekküllü bir IPv6/UDP ağ yığınının (network stack) gömülü olduğunu kanıtlıyor. Ayrıca `printf` ve `memcpy` gibi standart C kütüphanesi fonksiyonlarının da kodumuza dahil edildiğini görüyoruz.
* **'R' ve 'r' (Read-Only) Sembolleri:** Bunlar değiştirilemez sabit verileri temsil ediyor. Listede `cc2420_driver`, `csma_driver` ve `button_sensor` gibi semboller var. Bunlar cihazın telsiz çipini (CC2420) ve fiziksel butonlarını yöneten sürücülerin sabit yapı (struct) tanımlamaları.
* **Linker (Bağlayıcı) İşaretçileri:** Listenin sonlarına doğru yer alan `_etext`, `__data_load_start` ve `__vectors_start` gibi 'A' (Absolute) işaretli semboller benim yazmadığım, derleyicinin arka planda otomatik eklediği özel adresler. RAM'in nerede başladığını ve kesme vektörlerinin (interrupt vectors) nereye yerleştiğini işletim sistemine göstermek için kullanılıyorlar. 

Özetle bu tablo, Contiki-NG işletim sisteminin devasa altyapısının bizim o küçük udp-client kodumuzla nasıl birleşip makine adreslerine döküldüğünü çok güzel özetliyor.





## 5. Kesme Vektörleri veya Başlangıç Adresi ile İlişkili Bilgiler
`msp430-objdump -d` komutuyla makine kodunun (assembly) içine daldığımda çok tanıdık bir adresle karşılaştım: `0x3100`. Birinci maddede `readelf` ile bulduğumuz "Entry Point" (giriş) adresi tam olarak burasıydı! 

Çıktıyı incelediğimde, cihaz boot olduğunda doğrudan bizim C kodumuzdaki `main()` veya süreç (process) fonksiyonlarına gitmediğini görüyorum. İşlemci önce `0x3100` adresine zıplayıp donanımı hazırlamak için şu adımları işletiyor:
* **__watchdog_support (0x3100):** Cihaz boot sırasında takılıp kendine reset atmasın diye ilk iş olarak donanımsal watchdog timer ayarlanıyor.
* **__init_stack (0x310c):** Fonksiyonların çağrılabilmesi ve yerel değişkenlerin tutulabilmesi için Stack (yığın) bellek alanının başlangıç noktası (`0x3100` değeriyle) register'a yükleniyor.
* **__do_copy_data (0x3110) & __do_clear_bss (0x3128):** Hatırlarsan 2. ve 3. maddelerde `.data` ve `.bss` bölümlerinden bahsetmiştik. İşte cihaz tam bu adımda `.data` içindeki değişkenleri ROM'dan RAM'e kopyalıyor ve `.bss` alanındaki değişkenlerin içini sıfırlarla (`0x00`) temizliyor. 

Yani bu başlangıç adresi (reset vektörünün işaret ettiği ilk yer), işletim sisteminin çalışabilmesi için sahneyi hazırlayan o kritik "C Runtime (CRT)" kurulum kodlarını barındırıyor.





## 6. Dosyanın Neden "Ham Binary" Değil de "ELF Executable" Olarak Değerlendirildiği
Aslında bu sorunun cevabını yukarıdaki 5 maddeyi terminalde analiz ederken yaşayarak gördüm. Eğer elimizdeki dosya "ham binary" (raw binary) olsaydı, içinde sadece 0 ve 1'lerden oluşan kuru bir makine kodu yığını olurdu. `readelf`, `nm` veya `objdump` gibi araçları çalıştırdığımda o bölümleri veya fonksiyon isimlerini göremezdim.

Ancak bu dosya bir **ELF (Executable and Linkable Format)**. Yani içinde sadece çalışacak kodlar değil; aynı zamanda işletim sistemine ve linker'a (bağlayıcıya) rehberlik eden zengin bir "üst veri" (metadata) barındırıyor. RAM'in neresine hangi verinin yazılacağı (Section Headers), bizim yazdığımız fonksiyonların isimleri ve adresleri (Symbol Table), hata ayıklama bilgileri (Debug Info) ve cihazın boot olacağı ilk adres (Entry Point) bu ELF yapısının içinde saklı. Kısacası ELF, donanıma sadece "ne yapacağını" değil, "hafızayı nasıl organize edeceğini" de anlatan bir harita içerdiği için ham binary'den ayrılıyor.
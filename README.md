# MSP430 `.z1` / `.sky` / `ARM M4F(CC1352R)` / `cooja-native` Platformları için Üretilmiş Firmware’ler Üzerinde Yapılabilecek Analiz Türleri Kontrol Listesi

---
##### (* ARM Mimarisinde derlenmiş firmware analizi yapmak isteyen gruplar MSP430 Toolchain yanında ARM-Toolchain araçlarını da indirip, kullanmalıdırlar.)
``` bash
  $ wget https://armkeil.blob.core.windows.net/developer/Files/downloads/gnu-rm/9-2020q2/gcc-arm-none-eabi-9-2020-q2-update-x86_64-linux.tar.bz2
  $ tar -xjf gcc-arm-none-eabi-9-2020-q2-update-x86_64-linux.tar.bz2
```
---
##### ** Analiz etmeniz için farklı platformlarda oluşturulmuş örnek firmware arşivi bil.omu drive linki için [tıklayınız](https://drive.google.com/file/d/1oLrZWPmDyuznWe5qS7zOsfSyyyPcQbBG/view?usp=sharing) .


---

# 1. Binary Kimlik Analizi

![alt text](<images/Ekran görüntüsü 2026-05-21 171809.png>)
`new-firmware.z1` bellenim imajı üzerinde `msp430-readelf -h` komutu çalıştırılarak elde edilen parametreler ve bu parametrelerin fonksiyonel analizleri aşağıda sunulmuştur:

* **Hedef Platform Analizi:** Bellenim dosyasının `.z1` uzantısına sahip olması, bu imajın MSP430 tabanlı ultra düşük güçlü kablosuz duyarga ağı platformları (Z1 Motes) için üretildiğini göstermektedir.
* **MSP430 Mimari Tipi:** Çıktıda yer alan *Machine: Texas Instruments msp430 microcontroller* ifadesi, bellenimin 16-bit RISC mimarisine sahip düşük güçlü TI MSP430 çekirdeği için derlendiğini doğrulamaktadır.
* **ELF Format Bilgisi:** Dosya *Class: ELF32* (32-bit nesne formatı) yapısındadır. Bu durum dosyanın ham bir binary olmadığını, işletim sistemi veya bootloader tarafından anlamlandırılabilecek başlık ve kesit tabloları barındırdığını gösterir.
* **Endianness Nedir ve Endianness Bilgisi:** Endianness, çoklu bayt verilerinin bellekte hangi sırayla depolanacağını belirleyen mimari kuraldır. Çıktıda yer alan *2's complement, little endian* bilgisi, verinin en önemsiz baytının (LSB) en düşük bellek adresine yerleştirileceğini ifade eder.
* **Entry Point Adresi:** *0x3100*. Mikrodenetleyici resetlendikten sonra, işlemcinin program sayacının (PC) dallanacağı ve bellenimin ilk makine komutunu yürütmeye başlayacağı başlangıç adresidir.
* **ABI Nedir ve ABI Bilgisi:** ABI, derlenmiş makine kodunun çalışma zamanında donanımla nasıl etkileşime gireceğini belirleyen standartlar bütünüdür. Çıktıda *OS/ABI: Standalone App* ve *ABI Version: 0* olarak tespit edilmiştir. Bu durum, bellenimin harici bir kütüphaneye bağımlı olmadan donanım üzerinde doğrudan çalışacağını gösterir.
* **Compiler İzi:** ELF başlığındaki bayrak yerleşimleri incelendiğinde, bellenimin GNU GCC ekosistemine ait `msp430-gcc` derleyicisi tarafından üretildiğine dair yapısal izler taşımaktadır.
* **Toolchain Versiyonu:** *Version: 0x1 (current)* olarak raporlanmıştır. Analiz sürecinde `msp430` araç zinciri bileşenleri aktif olarak kullanılmıştır.
* **Optimization Level Tahmini:** Kesit başlıklarının dosya boyutunun oldukça ilerisinde olması ve çok sayıda debug bölüm tablosu barındırması, derleyicinin kodu aşırı derecede agresif optimize etmediğini (büyük olasılıkla `-O0` veya kod boyutu optimizasyonu için `-Os` seviyesinde tutulduğunu) göstermektedir.
* **Debug Symbol Var/Yok Analizi:** *Number of section headers: 21* olarak tespit edilmiştir. 21 adet kesit başlığının varlığı, dosya içeriğinde sembol tablosu ve hata ayıklama izlerinin korunduğunu, yani dosyanın sembollerden arındırılmadığını (*not stripped*) gösterir.
---

# 2. Bellek Kullanım Analizi

![alt text](<images/Ekran görüntüsü 2026-05-21 175957.png>)
`new-firmware.z1` imajı üzerinde çalıştırılan `msp430-readelf -S` komutunun çıktısı doğrultusunda sistemin bellek mimarisi ve kesit dağılımı şu şekildedir:

* **Flash, RAM, Stack, Heap Anlamları:**
  * *Flash (Kalıcı Hafıza):* Bellenim kodlarının, sabitlerin ve kesme vektörlerinin kalıcı olarak saklandığı salt okunur (read-only) alandır.
  * *RAM (Geçici Hafıza):* Çalışma zamanında değişkenlerin, yığın yapılarının ve dinamik verilerin tutulduğu hızlı okunup yazılabilir alandır.
  * *Stack (Yığın Çerçevesi):* Fonksiyon çağrıları, yerel değişkenler ve geri dönüş adreslerinin donanımsal olarak yönetildiği geçici bellek bölgesidir.
  * *Heap (Dinamik Bellek):* Çalışma zamanında `malloc` benzeri fonksiyonlarla dinamik olarak ayrılan bellek havuzudur.

* **Flash Kullanım Miktarı:** İmajın kalıcı hafızada kapladığı alan `.text`, `.far.text`, `.rodata`, `.data` ve `.vectors` kesitlerinin toplamıdır. Hesaplanan toplam değer yaklaşık 72 KB düzeyindedir.
* **RAM Kullanım Miktarı:** Bellenim ayağa kalktığında RAM üzerinde kilitlenen statik alan miktarı `.data` ve `.bss` kesitlerinin toplamıdır. Bu değer yaklaşık 5.9 KB seviyesindedir.
* **.text Boyutu:** Tablodaki `[ 2] .text` kesiti `0x00976e` bayt (yaklaşık 37.8 KB) boyutundadır. Programın asıl yürütülebilir makine kodlarını barındırır. Ek olarak `[ 1] .far.text` kesiti de `0x004a78` bayt (yaklaşık 18.6 KB) boyutunda bir uzak kod alanına sahiptir.
* **.data Boyutu:** Tablodaki `[ 4] .data` kesiti `0x000150` bayt (tam olarak 336 Bayt) boyutundadır. Başlangıç değeri atanmış küresel ve statik değişkenleri barındırır.
* **.bss Boyutu:** Tablodaki `[ 5] .bss` kesiti `0x001648` bayt (yaklaşık 5.5 KB) boyutundadır. Başlangıç değeri atanmamış küresel değişkenler için ayrılmıştır. `Type` alanında `NOBITS` yazması, bu alanın diskteki bellenim dosyasında yer kaplamadığını, sadece RAM'e yüklendiğinde rezerve edileceğini gösterir.
* **Stack Kullanım Tahmini:** MSP430 mimarisinde yığın (stack) RAM'in en üst adresinden başlayarak aşağıya doğru büyür. Statik RAM kullanımının (~5.9 KB) ardından geriye kalan boş RAM alanı, çalışma zamanındaki derin fonksiyon çağrıları ve kesme servis rutinleri (ISR) için stack alanı olarak kullanılacaktır.
* **Heap Var/Yok Analizi:** Çıktıda dinamik bellek tahsisine işaret eden herhangi bir özel `.heap` kesiti veya dinamik bağlama tablosu yer almamaktadır. Contiki-NG gibi gömülü işletim sistemleri, determinizm ve bellek sızıntılarını önlemek adına genellikle dinamik heap yönetimini devre dışı bırakır veya oldukça kısıtlı kullanır.
* **Section Dağılımı:** Dosyada toplam 21 adet kesit başlığı (section header) bulunmaktadır. Bunların bir kısmı donanıma yüklenecek aktif kod ve verileri barındırırken (`.text`, `.data`, `.bss`), büyük bir kısmı ise (`.debug_info`, `.debug_line` vb.) sadece hata ayıklama süreçleri için dosyada tutulan veri dışı alanlardır.
* **Memory Map Analizi:** Kod ve salt okunur veriler Flash üzerinde `0x3100` (.text) ve `0x10000` (.far.text) taban adreslerinden itibaren eşlenmiştir. Yazılabilir statik veriler ise RAM üzerinde `0x1100` (.data) adresinden başlayarak yerleştirilmiştir.
* **Büyük Veri Yapılarının Tespiti:** `.bss` kesitinin 5.5 KB gibi mikrodenetleyici ölçeğinde büyük bir boyuta sahip olması, bellenim kaynak kodunda geniş boyutlu statik dizilerin, ağ paket tamponlarının (packet buffers) veya komşu yönlendirme tablolarının tanımlı olduğunu göstermektedir.
---

# 3. Symbol / Function Analizi

![alt text](<images/Ekran görüntüsü 2026-05-21 182103.png>)
`new-firmware.z1` bellenimi üzerinde `msp430-nm -n` komutu çalıştırılarak sembol tablosu adres sırasına göre çözümlenmiş ve cihazın fonksiyonel haritası doğrudan çıktı parametreleri üzerinden şu şekilde doğrulanmıştır:
* **Fonksiyon İsimleri:** Bellenim içerisindeki yürütülebilir fonksiyonlar tablodaki `T` (küresel kod) ve `t` (statik/local kod) bayraklarından tespit edilmiştir. Örneğin; donanım seviyesinde `accm_init` (0x396a), `cc2420_init` (0x4436) ve sistem ana fonksiyonu olan `main` (0x313e) bellenimin çekirdek fonksiyon isimleridir.
* **Global Değişkenler:** Program genelinde erişilebilir olan ve başlangıç değeri atanmış küresel değişkenler, RAM adresi olan `0x1100` bölgesinden itibaren `D` harfi ile işaretlenmiştir. Örneğin; `hello_world_process` (0x114a) ve `sensors` (0x122c) birer başlatılmış küresel yapıdır. Ayrıca `node_id` (0x14e6) değişkeni de `B` bayrağı ile RAM üzerinde global olarak rezerve edilmiştir.
* **Static Değişkenler:** Sadece tanımlandığı kaynak dosya içerisinden erişilebilen lokal statik değişkenler, küçük harfli `d` ve `b` bayrakları ile ayrıştırılmıştır. Örneğin; `mac_pan_id` (0x1148) statik başlatılmış bir veri iken, `channel` (0x1272) ve `suppressTimer1` (0x1254) RAM üzerindeki başlatılmamış statik değişkenlerdir.
* **ISR (Interrupt Service Routine) Fonksiyonları:** Çıktının son kısımlarında `0xFFC0` adresinde yer alan `__ivtbl_32` (Kesme Vektör Tablosu) sembolü ve buna bağlı olarak `0x353e` adresindeki `port1_isr` (Buton/GPIO kesmesi), `0x35c2` adresindeki `irq_p2` (Radyo kesmesi), `0x37ae` adresindeki `uart0_rx_interrupt` gibi `_isr` uzantılı semboller, donanımsal kesme servis rutinlerinin haritasını oluşturmaktadır.
* **Contiki Process Entry’leri:** Sembol tablosunda yer alan `hello_world_process` (0x114a), `cc2420_process` (0x110c), `etimer_process` (0x113c) ve bunların iş parçacığı karşılıkları olan `process_thread_hello_world_process` (0x5ba6) sembolleri, Contiki-NG'nin protothread tabanlı çoklu görev (multi-tasking) yönetim girişlerini kanıtlamaktadır.
* **Radio Driver Fonksiyonları:** IEEE 802.15.4 kablosuz ağ standardını kullanan radyo donanım sürücüsüne ait `cc2420_driver` (0xc910) sembolü, `cc2420_on` (0x3e5c), `cc2420_off` (0x3ef6) ve `cc2420_set_channel` (0x408a) fonksiyonları doğrudan telsiz haberleşme sürücü katmanını doğrular.
* **Timer Callback’leri:** Gömülü sistem periyodik zamanlayıcıları ve geri çağırımları için kullanılan `ctimer_init` (0x5034), `etimer_set` (0x52ac) ve rtimer mimarisine ait `rtimer_arch_schedule` (0xaad4) sembolleri sistem zamanlama alt yapısını oluşturmaktadır.
* **Networking Callback’leri:** Çıktıda yer alan `uip_process` (0x130f2), `tcpip_input` (0xc1b4) ve IPv6 yönlendirme protokolüne ait olan `rpl_process_dio` (0x79a0), `rpl_process_dis` (0x7ce4) gibi semboller ağ katmanı callback ve paket işleme mekanizmalarıdır.
* **Sensor Handler’ları:** Çevresel donanımların kontrolünü sağlayan `sensors_process` (0x11d6) yapısı, ivmeölçer için `accm_read_axis` (0x38f2) ve sıcaklık sensörü için `tmp102_read_temp_x100` (0xc806) sembolleri donanım işleyicileridir.
* **Kullanılan Kütüphaneler:** Sembol listesinin sonundaki `memcpy` (0x1484c), `memset` (0x14a1c), `printf` (0x13e88) ve `rand` (0x147f0) sembolleri, bellenimin ihtiyaç duyduğu standart C kütüphane fonksiyonlarının derleme anında bellenim içerisine statik olarak bağlandığını göstermektedir.
* **Kullanılmayan (Dead) Fonksiyonlar:** Çıktının en başında yer alan ve başında adres bilgisi bulunmayan `U` (Undefined) bayraklı `button_hal_buttons` veya `gpio_hal_arch_init` gibi semboller, kod içerisinde referans verilmiş fakat bu firmware bileşeninde alt kısımları boş bırakılmış veya o an çağrılmayan ölü fonksiyon yapılarına işaret etmektedir.
* **Function Address Mapping:** Tüm fonksiyonların MSP430 Flash bellek uzayındaki fiziksel yerleşim haritası `main` fonksiyonunun `0x313e` adresinden başlayıp, kütüphane fonksiyonlarının bittiği `0x14a78` adresine kadar (`_efartext`) kesintisiz olarak haritalanmıştır.
---

# 4. String ve Metadata Analizi
![alt text](<images/Ekran görüntüsü 2026-05-21 183442.png>)
![alt text](<images/Ekran görüntüsü 2026-05-21 183456.png>)
`new-firmware.z1` bellenimi üzerinde `msp430-strings` komutu koşturularak bellenimin içine statik olarak gömülmüş tüm log mesajları, protokol tanımları ve meta veriler gerçek çıktı eşleşmeleriyle şu şekilde çözümlenmiştir:
* **Hata Ayıklama / printf Logları:** Bellenim içerisinde sistem durumunu raporlayan standart ve biçimlendirilmiş log kalıpları tespit edilmiştir. Çıktıda yer alan `[%-4s: %-10s]`, `packet sent: no metadata`, `could not allocate queuebuf, dropping packet` ve `Check in inconsistent state` dizgileri, sistemin çalışma anında terminale basacağı aktif hata ayıklama mesajlarıdır.
* **IPv6 Adresleri:** Ağ katmanında adres durum tespiti yapan `Tentative link-local IPv6 address:`, `IPv6 addresses:`, `Adding global IP address` ve `Removing global IP address` metin dizgileri, bellenimin IPv6 adres yönetim yeteneğini doğrulamaktadır.
* **MAC Adresleri:** Bağlantı katmanı (Link-layer) takibi için gömülen `Link-layer address:`, `drop duplicate link layer packet from` ve `%02x:%02x...` formatına işaret eden ardışık `%02x` dizgileri, cihazın MAC adresi çözümleme fonksiyonlarını gösterir.
* **Ağ Düğüm Kimlikleri:** Cihazların simülasyondaki benzersiz numaralarını gösteren `Node ID: %u` dizgisi, bellenimin düğüm kimlik atama (Node ID) meta verisini doğrudan barındırdığını kanıtlar.
* **Sensör İsimleri:** Donanım katmanında sürücüleri olan fiziksel sensör adları çıktıdan doğrudan okunmaktadır: `ADXL345 sensor` (Dijital ivmeölçer) ve `TMP102 sensor` (Dijital sıcaklık sensörü).
* **Process İsimleri:** Contiki-NG üzerinde koşan bağımsız süreçlerin (processes) metinsel isimleri log çıktılarında yakalanmıştır: `Accelerometer process`, `CC2420 driver`, `Ctimer process`, `Event timer`, `Hello world process` ve `TCP/IP stack`.
* **Yönlendirme Protokolleri:** Kablosuz duyarga ağlarında düşük güçlü yönlendirme standardı olan RPL mimarisine ait `RPL Lite` dizgisi bellenimin ağ protokol kimliğini tanımlar.
* **TSCH / 6LoWPAN / RPL Stringleri:** Çıktıda üst katman protokol yığınlarına (network stacks) ait çok yoğun meta veri dizgileri bulunmuştur:
  * *6LoWPAN için:* `reassembly: failed to store new fragment`, `uncompression: IPHC dispatch`, `compression: after (%d)` dizgileri.
  * *RPL için:* `created a new RPL DAG`, `ignoring DIO with an unsupported OF`, `sending a DIS to`, `sending a %sDAO seqno %u` yönlendirme dizgileri.
* **Gizli Teşhis Mesajları (Hidden Diagnostics):** Sistem açılışında basılan ve bellenimin hangi açık kaynak ağacından derlendiğini tam revizyon numarasıyla gösteren `Starting Contiki-NG-release/v4.8-625-g8518cbaff-dirty` metni, bellenimin geliştirme mimarisine ait en kritik gizli teşhis/metadata bilgisidir.
* **Sabit Kodlanmış Yapılandırma Değerleri (Hardcoded Configs):** Ağın kimlik ve frekans parametrelerini belirleyen `- 802.15.4 PANID: 0x%04x`, `- 802.15.4 Default channel: %u` ve `- MAC: %s` dizgileri, bellenim içerisindeki statik konfigürasyon yapısını gösterir.
* **Geliştirici Notları:** Hata ve uyarı durumları için geliştiriciler tarafından kod bloklarına yerleştirilen ayırt edici `WARN`, `INFO` ve `! radio does not support getting RADIO_CONST_MAX_PAYLOAD_LEN. Abort init.` gibi kritik geliştirici uyarı notları ve mesaj kalıpları ayıklanmıştır.
---

# 5. Assembly / Instruction Analizi

![alt text](<Ekran görüntüsü 2026-05-21 184232.png>)
`new-firmware.z1` bellenimi içerisindeki `<input>` fonksiyonuna ait makine kodları `msp430-objdump -d` komutu ile disassemble edilmiş ve derleyicinin ürettiği komut mimarisi şu şekilde analiz edilmiştir:
* **Instruction Sequence Analizi:** Fonksiyon genel olarak MSP430X genişletilmiş komut kümesine ait veri taşıma (`mov`, `mov.b`), aritmetik (`add`, `sub`), bit manipülasyonu (`bis`, `and`, `swpb`), karşılaştırma (`cmp`, `tst`) ve dallanma (`jz`, `jnz`, `jmp`, `bra`) komut dizilimlerinden oluşmaktadır. 
* **Function Prologue/Epilogue:** Fonksiyonun giriş (prologue) kısmında `10000: pushm.a #8, r11` komutu görülmektedir. Bu komut, fonksiyon içinde ezilecek olan r11 ve altındaki 8 adet yazmacı tek seferde yığına (stack) yedekler. Hemen ardından `10002: add #-14, r1` komutu ile yığın işaretçisi (`r1` / SP) yukarı kaydırılarak yerel değişkenler için yığında 14 baytlık güvenli bir alan açılır.
* **Register Kullanımı:** Fonksiyon parametre aktarımlarında ve ara hesaplamalarda yoğun bir şekilde genel amaçlı yazmaçları kullanmaktadır. MSP430 çağrı protokolüne uygun olarak, kütüphane fonksiyonlarına (örneğin `0x13be8`) gönderilecek parametreler ve dönen sonuçlar `r15`, `r14` ve `r13` yazmaçları üzerinden taşınmaktadır.
* **Stack Frame Yapısı:** `add #-14, r1` ile açılan stack çerçevesine, fonksiyonun orta kısımlarında yer alan `mov #148, 8(r1)` ve `mov.b #1, 7(r1)` komutları ile doğrudan erişim sağlandığı, yerel değişkenlerin ve tamponların `r1` tabanlı ofset adreslerinde (indisli adresleme modu ile) saklandığı tespit edilmiştir.
* **ISR Akışı:** İncelenen kod bloğu donanımsal bir kesme (ISR) değil, kesme servis rutinlerinden veya ağ katmanından gelen paketleri yakalayan üst seviye bir yazılımsal giriş alt rutinidir. Ancak donanım saklayıcılarına (`&0x249a` gibi mutlak adreslere) erişim izleri barındırır.
* **Loop Yapıları:** İncelenen kesitte düzenli bir geriye doğru dallanma (loop) olmamasına rağmen, fonksiyonun ileri kısımlarında koşulların kontrol edilip belirli blokların atlanması veya döngüsel süreçlerin işletilmesi için `jmp`, `jz` ve `jnz` komut grupları kombine edilmiştir.
* **Branch Analizi:** Kod içerisinde koşullu ve koşulsuz dallanmalar çok yoğundur. `10026: tst r15` (r15 sıfır mı testi) komutunun hemen ardından gelen `10028: jnz $+38` koşullu dallanması, eğer veri sıfır değilse hata ayıklama bloğunu atlayarak doğrudan paket işleme alanına (`0x1004e`) geçişi koordine eder.
* **Jump Table Analizi:** Çıktıda `swpb` (bayt yer değiştirme) ve `and` maskeleme işlemleriyle verinin tipine bakıldığı (`and #248, r14` ve `cmp #192, r14`) görülmektedir. Bu kalıp, switch-case bloklarının derleyici tarafından ardışık karşılaştırma ve atlama (jump) yapılarına dönüştürüldüğünü gösterir.
* **Function Call Graph:** `<input>` fonksiyonunun içinden harici kütüphane ve alt bileşen fonksiyonlarına yoğun çağrılar (`calla` - extended call) yapılmaktadır. Çağrılan kritik adresler: `0x06ac8` (packetbuf_addr), `0x05ed8` (link_stats_input_callback), `0x13e88` (printf loglama) ve `0x0ae48` (store_fragment) fonksiyonlarıdır.
* **Inline Function Tespiti:** Kaynak kodda fonksiyon olarak tanımlanan bazı küçük yardımcı rutinlerin (örneğin bit kaydırma işlemlerinin), ayrı bir fonksiyon çağrısı (`calla`) yapmak yerine `rlam #2, r14` (yazmacı sola 2 bit kaydır/4 ile çarp) komutuyla doğrudan kodun içine gömüldüğü (inline edildiği) tespit edilmiştir.
* **Compiler Optimization Davranışı:** Derleyicinin tek bir komutla birden fazla yazmacı itmesini sağlayan `pushm.a` komutunu tercih etmesi ve sabit üreteç yazmacını (`r3 As==01`) kullanarak hızlı atamalar yapması, kod boyutunu küçültmeye yönelik etkin bir optimizasyon (`-Os`) uygulandığını göstermektedir.
* **Delay Loop / Busy-Wait Yapıları:** Kod kesitinde donanımın hazır olmasını bekleyen statik bir gecikme döngüsü yerine, doğrudan verinin durumunu (`tst r15`) test edip, olumsuz durumda `bra #0x11182` ile fonksiyondan güvenli çıkış yapan olay odaklı asenkron bir akış tercih edilmiştir.
* **Protothread Expansion / Scheduler Davranışı:** Kodun `bra #0x10e5e` komutu ile sonlanması ve işlenen paketlerin durumuna göre Contiki-NG ana çekirdek fonksiyonlarına (`calla #0x13c0e`) yönlendirme yapması, makine kodu seviyesinde protothread yapılarının yielding (teslim etme) ve zamanlayıcı (scheduler) etkileşim noktalarını doğrulamaktadır.
---

# 6. Source-Level Mapping Analizi

(Debug build varsa)

* Address → source line eşleme
* Function → source file eşleme
* ISR → source mapping
* Crash address çözümleme
* Optimization sonrası source mapping
* Inline edilmiş kodların tespiti

Araçlar:

* `msp430-addr2line`
* `msp430-objdump -S`
* `Ve üstteki araçların ARM versiyonları...`

---

# 7. ELF Yapısı Analizi

* ELF header
* Section header
* Program header
* Symbol table
* Relocation entries
* Debug sections
* DWARF info
* Linker-generated metadata
* Startup section
* Vector table
* Initialization routines

Araçlar:

* `msp430-readelf`
* `msp430-elfedit`
* `Ve üstteki araçların ARM versiyonları...`

---

# 8. Interrupt ve Donanım Analizi

* Interrupt vector table
* GPIO access pattern
* Timer interrupt kullanımı
* UART ISR
* Radio interrupt handler
* ADC access
* Sensor polling
* Low-power mode geçişleri
* Clock configuration
* MSP430 register erişimleri

Araçlar:

* `msp430-objdump`
* `msp430-readelf`
* `Ve üstteki araçların ARM versiyonları...`

---

# 9. Networking Analizi

* Unicast kullanım tespiti
* Broadcast kullanım tespiti
* Multicast tespiti
* IPv6 stack kullanımı
* RPL routing analizi
* TSCH scheduler çağrıları
* MAC layer interaction
* Packet buffer kullanımı
* Neighbor table erişimi
* Radio transmission akışı
* Retransmission logic
* ACK mekanizmaları
* CSMA/TSCH farkları
* Contiki network API kullanımı

Araçlar:

* `msp430-nm`
* `msp430-objdump`
* `msp430-strings`
* `Ve üstteki araçların ARM versiyonları...`

---

# 10. Wireless / TSCH Analizi

* TSCH slot operation
* Channel hopping logic
* ASN handling
* Radio timing loops
* Synchronization routines
* Schedule management
* Packet timing
* MAC timing critical path
* Drift compensation
* Low-power radio behavior

Araçlar:

* `msp430-objdump`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

---

# 11. Sensor ve Peripheral Analizi

* Button handler
* LED driver
* UART usage
* SPI access
* I2C access
* ADC routines
* Sensor polling interval
* Interrupt-driven sensor logic
* GPIO toggle behavior
* Peripheral initialization sequence

Araçlar:

* `msp430-objdump`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

---

# 12. Algoritma Koşma / DSP / Matematiksel Analiz

* Floating-point kullanımı
* Fixed-point kullanımı
* Trigonometric computation
* Multiply/divide routines
* Software floating-point emulation
* DSP benzeri loop’lar
* Matrix operation izleri
* Signal processing pattern’leri
* Computational hotspot’lar
* Numerical optimization

Araçlar:

* `msp430-objdump`
* `msp430-gprof`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

---

# 13. Güç ve Performans Analizi

* Low-power mode geçişleri
* CPU-intensive function’lar
* Busy-wait detection
* Sleep/wakeup flow
* Timer usage intensity
* Radio duty cycle tahmini
* ISR yoğunluğu
* Function execution cost
* Flash/RAM efficiency
* Energy-heavy computation bölgeleri

Araçlar:

* `msp430-gprof`
* `msp430-objdump`
* `msp430-size`
* `Ve üstteki araçların ARM versiyonları...`

---

# 14. Coverage ve Profiling Analizi

* Function call frequency
* Execution hotspot
* Unused branch’ler
* Rarely executed path’ler
* Test coverage
* Critical execution path
* Runtime bottleneck’ler

Araçlar:

* `msp430-gcov`
* `msp430-gprof`
* `Ve üstteki araçların ARM versiyonları...`

---

# 15. Reverse Engineering Analizi

* Firmware behavior recovery
* Unknown firmware classification
* Feature inference
* Protocol inference
* ISR purpose discovery
* Hardware interaction recovery
* State machine extraction
* Scheduler reconstruction
* Event-flow reconstruction
* Network role inference

Araçlar:

* `msp430-objdump`
* `msp430-nm`
* `msp430-readelf`
* `msp430-strings`
* `Ve üstteki araçların ARM versiyonları...`

---

# 16. Compiler ve Optimization Analizi

* `-O0/-O2/-Os` farkları
* Inlining behavior
* Dead code elimination
* Constant folding
* Loop optimization
* Register allocation
* Tail-call optimization
* Branch optimization
* Macro expansion
* Preprocessor etkileri

Araçlar:

* `msp430-gcc`
* `msp430-cpp`
* `msp430-objdump`
* `Ve üstteki araçların ARM versiyonları...`

---

# 17. Linker ve Build Sistemi Analizi

* Section placement
* Link order
* Static library linkage
* Startup code
* Linker script behavior
* Vector placement
* Symbol resolution
* Relocation behavior

Araçlar:

* `msp430-ld`
* `msp430-ar`
* `msp430-ranlib`
* `msp430-readelf`
* `Ve üstteki araçların ARM versiyonları...`

---

# 18. Binary Transformation Analizi

* ELF → HEX conversion
* ELF → binary conversion
* Section extraction
* Symbol stripping
* Debug removal
* Firmware minimization
* Binary patch preparation

Araçlar:

* `msp430-objcopy`
* `msp430-strip`
* `Ve üstteki araçların ARM versiyonları...`

---

# 19. Library ve Archive Analizi

* Static library içeriği
* Object file extraction
* Archive symbol table
* Linked module analizi

Araçlar:

* `msp430-ar`
* `msp430-gcc-ar`
* `msp430-ranlib`
* `Ve üstteki araçların ARM versiyonları...`

---

# 20. Contiki-NG Özel Analizler

* PROCESS_THREAD recovery
* Protothread expansion
* Event-driven scheduler analizi
* etimer/ctimer usage
* PROCESS_BEGIN/END expansion
* PROCESS_YIELD flow
* NETSTACK interaction
* Packetbuf lifecycle
* uIP callback chain
* Rime stack usage

Araçlar:

* `msp430-cpp`
* `msp430-objdump`
* `msp430-nm`
* `Ve üstteki araçların ARM versiyonları...`

---

# 21. Güvenlik ve Robustness Analizi

* Hardcoded credential arama
* Debug backdoor izleri
* Buffer handling
* Unsafe memory access
* Stack-heavy routines
* Potential overflow bölgeleri
* Assert/debug remnants
* Information leakage string’leri

Araçlar:

* `msp430-strings`
* `msp430-objdump`
* `msp430-readelf`
* `Ve üstteki araçların ARM versiyonları...`

---

# 22. Karşılaştırmalı Firmware Analizi

İki firmware arasında:

* Code size farkı
* RAM farkı
* Function count farkı
* ISR yoğunluğu
* Networking complexity
* Radio stack farkı
* Symbol farkı
* Optimization farkı
* Assembly complexity farkı



---

# 23. Eğitimsel Reverse Engineering Görevleri

* Bir firmware’in ne yaptığını bulma
* hangi protokolü kullandığını çıkarma
* button/LED mapping bulma
* ISR’leri tanıma
* network role çıkarımı
* Kullandığı algoritmik blok tespiti
* energy-heavy bölgeleri bulma
* stripped firmware çözümleme


---

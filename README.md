## Akıllı Ev Enerji Tüketimi Tahmin Projesi
Bu projede, bir akıllı evden alınan sensör verilerini ve hava durumu bilgilerini kullanarak, evin saatlik toplam elektrik tüketimini (House overall [kW]) makine öğrenmesi yöntemleriyle tahmin etmeye çalıştım.

Amacım sadece var olan veriyi modele vermek değil, geçmiş tüketim alışkanlıklarını da modele öğreterek daha gerçekçi tahminler elde etmek.

---
### 1. Veri Setinin Yüklenmesi ve Temizlik
İlk olarak Smart Home Dataset.csv dosyasını yükledim. Veriyi incelediğimde yaklaşık 246 bin satır veri olduğunu gördüm.
<br>
<img src="/images/resim1.png" width="700" >
<br>
Eksik Veriler (NaN): Veri setinde çok az sayıda (sadece 1-2 satır) eksik veri olduğunu fark ettim. Bu kadar az veri için ortalama ile doldurma (imputation) yapmak yerine, veri bütünlüğünü bozmamak adına bu satırları sildim (dropna).
<br>
<img src="/images/resim1.1.png" width="300" >
<br>
<img src="/images/resim1.2.png" width="300" >

---
### 2. Zaman Verisini İşlenebilir Hale Getirme
Makine öğrenmesi modelleri 2016-01-01 14:00 gibi tarih formatlarını doğrudan anlayamaz. Bu yüzden time sütununu parçaladım:

Saat (hour): Elektrik tüketimi günün saatine göre çok değişir (Gece az, akşam çok). Bu yüzden saati ayırdım.
Gün ve Ay (day_of_week, month): Hafta içi/hafta sonu ayrımı ve mevsimsel etkileri yakalamak için bu özellikleri türettim.
<br>
<img src="/images/resim2.png" width="700" >

---
### 3. Özellik Mühendisliği (Feature Engineering) 
Modelin başarısını artırmak için veriye iki kritik ekleme yaptım:

* #### A. Geçmişe Bakış:
  Evin şu anki tüketimi, büyük ihtimalle 1 saat önceki veya dün aynı saatteki tüketimiyle ilişkilidir. Bu yüzden shift() fonksiyonunu kullanarak: Veriyi aşağı kaydırmış oldum ve geçmiş değerler bugünkü satıra geldi. Yani geçmişte olan artık bugünkü satırın özelliği oldu diyebiliriz.  Mesela burada shift(1) dediğimizde 1 satır aşağı kaydırmış oluyorz yani her satıra bir önceki saatin tüketimini yazıyoruz.

Last_1_Hour_Consumption: 1 saat önceki tüketim, 

Last_24_Hour_Consumption: Tam 24 saat önceki tüketim verilerini sütun olarak ekledim. 
<br>
<img src="/images/resim3.png" width="700" >
<br>
* #### B. Referans Noktası:
  Modelin "Bu saatte ve bu hava durumunda ev normalde ne kadar yakar?" sorusuna cevap verebilmesi için bir referans tablosu oluşturdum. Yani burada amacımız aynı saat (hour) ve aynı hava durumu (icon) koşullarında evin “ortalama” elektrik tüketimini referans (baseline) olarak hesaplayıp ve bunu her satıra eklemek.

Önce eğitim verisindeki her Saat ve Hava Durumu (Icon) ikilisi için ortalama tüketimi hesapladım (groupby ile).
Bu ortalamayı Baseline_Consumption adıyla ana tabloya ekledim. Bu sayede model, tahmine sıfırdan başlamak yerine bu ortalamayı baz alıp ince ayar yapabildi.
<br>
<img src="/images/resim3.1.png" width="700" >

---
### 4. Hangi Sütunlar Neden Çıkarıldı? 
Projenin en önemli kısmı burasıydı. İlk başta bu sütunları silmeyi unutmuştum ve r2 skorum 0.99 olarak geldi yani bu modelin verrideki değişkenliği tamamı ile doğru bir şekilde açıklayabilmesi demek yani neredeyse yanlış tahmin yapmıyordu.Ama bu hile oluryordu çünkü bu özelliklerin aslında veri sızıntısına sebep olduğunu anladım.
<br>
<img src="/images/resim4.png" width="700" >
<br>
Çıkardığım sütunlar ve sebepleri:

Ev Aletleri (Dishwasher, Furnace, Fridge, Solar, Living room vb.): * Sebep: Benim hedefim evin Toplam Tüketimini tahmin etmek. Eğer bulaşık makinesinin, fırının ve buzdolabının ne kadar yaktığını modele verirsem, model bunları toplayıp sonucu bulur. Bu bir tahmin değil, hesap makinesi işlemi olur. Geleceği tahmin ederken "Yarın bulaşık makinesi çalışacak mı?" sorusunun cevabını bilemeyeceğim için bu sütunları sildim.

* House overall [kW]: Sebep: Hedef değişkenim olduğu için X (girdi) verisinden çıkardım.
* Metin ve Ham Veriler (summary, cloudCover, time, date_obj):

Sebep: summary sütunu icon ile aynı bilgiyi taşıyordu, gereksiz tekrarı önlemek ve metin verisiyle uğraşmamak için çıkardım. cloudCover sütununda ise sayısal olmayan hatalı veriler vardı. Tarihleri zaten parçaladığım için ham hallerini sildim.


---
### 5. Encoding ve Ölçeklendirme
One-Hot Encoding: Hava durumu (icon) "Yağmurlu", "Güneşli" gibi kategorik veriler içeriyordu. Modelin bunları matematiksel işleyebilmesi için 0 ve 1'lerden oluşan sütunlara çevirdim (get_dummies). BUrada one-hot encoding her kategori için ayrı ayrı sütunlar oluşturuyor. Ama bu sorun değil çünkü icon özelliğimiz de zaten çok fazla farklı değişken olmadığı için aşırı bir sütun oluşumuna sebep olmuyor.Burada label encoding kullanmak saçma olurdu çünkü icon içindeki değerler anlam bakımından sıralanabilecek ve kıyaslanabilecek değerler değil. 
<br>
<img src="/images/resim5.png" width="700" >
<br>
StandardScaler: Kullandığım KNN algoritması, sayılar arasındaki mesafeye duyarlı olduğu için (büyük sayılar küçükleri ezmesin diye) tüm verileri standart bir aralığa ölçekledim.Bu şekilde farklı büyüklükteki veriler karşılaştırılabilir hale geldi.Özellikle KNN için scale etmek önemli diyebiliriz.Çünkü her tahminini mesafe ve komşuluğa bakarak yapıyor ve eğer bu durumda ölçekler yanlış olursa komşular yanlış seçilir ve düşük tahmin değerleri verir diyebiliriz.
<br>
<img src="/images/resim5.1.png" width="700" >

---
### 6. Modelleme ve Sonuçlar 
İki farklı algoritma denedim:
<br>
<img src="/images/resim6.png" width="700" >
<br>
Random Forest Regressor: Karmaşık ilişkileri yakalamakta başarılı olduğu için.

KNN (K-Nearest Neighbors): Benzer koşullardaki geçmiş verileri baz aldığı için.


Sonuç: Her iki model de R² = ~0.76 civarında bir başarı skoru verdi.
<br>
<img src="/images/resim6.1.png" width="700" >
<br>
Bu skor, modelin evin elektrik tüketimindeki değişimin %76'sını doğru açıklayabildiğini gösteriyor.

Geriye kalan %24'lük kısım ise insan davranışındaki rastgelelikten (aniden misafir gelmesi, fırının beklenmedik anda çalıştırılması vb.) kaynaklanıyor ki bu gayet beklenen, gerçekçi bir sonuç.

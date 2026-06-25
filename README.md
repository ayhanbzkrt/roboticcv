# AI’Han Computer Vision Robotics Lab

**MediaPipe tabanlı gerçek zamanlı hareket takibi, Three.js 3B robot kontrolü ve eklem senkronizasyonu deneyimi**

[English version](#english) · [Türkçe sürüm](#türkçe)

---

# Türkçe

## Proje Hakkında

**AI’Han Computer Vision Robotics Lab**, bilgisayarlı görü teknolojilerinden elde edilen gerçek zamanlı insan hareketi verilerinin üç boyutlu bir robot modeline nasıl aktarılabileceğini deneyimlemek amacıyla geliştirilmiş etkileşimli bir eğitim uygulamasıdır.

Uygulama;

* MediaPipe el takibi,
* yüz landmark ve mimik analizi,
* vücut pozu tahmini,
* Three.js tabanlı 3B render,
* quaternion dönüşümleri,
* iskelet ve eklem hiyerarşisi,
* hareket filtreleme,
* gerçek zamanlı robot kontrolü

gibi bilgisayarlı görü ve 3B grafik konularını tek bir çalışma ortamında bir araya getirir.

Projenin merkezinde yer alan **AI’Han**, kullanıcının kamera üzerinden algılanan hareketlerini farklı çalışma modlarına göre takip eden ve bunları robotik eklem hareketlerine dönüştüren üç boyutlu bir eğitim robotudur.

---

## Temel Özellikler

### Hand Pose — El ve Kol Takibi

Hand Pose modunda yalnızca el ve kol zincirleri aktif olarak kontrol edilir.

* Sağ ve sol el bağımsız takip edilir.
* Omuz, dirsek, bilek, avuç ve parmak eklemleri senkronize edilir.
* El landmarkları parmak kıvrımlarına dönüştürülür.
* Pose Assist desteğiyle omuz–dirsek–bilek zinciri hesaplanır.
* Algılanmayan kol kontrollü biçimde nötr pozda tutulur.
* Gövde, yüz ve bacaklar standby konumunda kalır.

### Face Mesh — Yüz ve Mimik Takibi

Face Mesh modunda robotun yüzü otomatik olarak yakın kadraja alınır.

* Başın sağa, sola, yukarı ve aşağı yönelimi
* Göz kırpma
* Kaş hareketleri
* Ağız ve çene açıklığı
* Gülümseme yoğunluğu
* Yüz hareketlerinin yumuşatılması

gerçek zamanlı olarak AI’Han’ın yüz sistemine aktarılır.

Bu modda robotun kolları, gövdesi ve bacakları sabit standby pozunda kalır.

### Body Pose — Tam Vücut Takibi

Body Pose modu, MediaPipe Pose Landmarker tarafından üretilen vücut bağlantı noktalarını robotun iskelet sistemine aktarır.

Kontrol edilen başlıca bölgeler:

* Baş ve boyun
* Omuzlar
* Üst ve alt kollar
* Bilekler
* Gövde
* Kalça
* Üst ve alt bacaklar
* Dizler
* Ayak bilekleri
* Ayak yönleri

Düşük güven veya görünürlük değerine sahip bağlantı noktaları robot hareketine doğrudan uygulanmaz.

### Fusion — Birleşik Takip

Fusion modunda üç farklı computer vision sistemi birlikte çalışır:

* **Pose Landmarker:** Gövde, omuzlar, kollar ve bacaklar
* **Hand Landmarker:** Avuç, bilek yönelimi ve parmaklar
* **Face Landmarker:** Baş hareketleri ve yüz mimikleri

Her model yalnızca sorumlu olduğu eklem grubunu kontrol eder. Böylece farklı modellerin aynı eklemi eş zamanlı olarak farklı yönlere çekmesi engellenir.

---

## Senkronizasyon Motoru

AI’Han hareket sistemi, MediaPipe landmark koordinatlarını doğrudan robot eklemlerine kopyalamaz.

İşlem hattı aşağıdaki aşamalardan oluşur:

```text
Kamera karesi
    ↓
MediaPipe landmark tahmini
    ↓
Landmark güven ve görünürlük kontrolü
    ↓
Koordinat normalizasyonu
    ↓
Yön ve kemik vektörlerinin oluşturulması
    ↓
Yerel eklem uzayına dönüşüm
    ↓
Quaternion hedeflerinin hesaplanması
    ↓
Filtreleme ve hareket tahmini
    ↓
Frame-rate bağımsız quaternion damping
    ↓
Three.js robot eklem hiyerarşisi
```

### Quaternion Dönüşümü

Bir robot kemiğinin başlangıç yönü ile MediaPipe tarafından tahmin edilen hedef yön arasındaki dönüş quaternion ile hesaplanır.

Temel yaklaşım:

```javascript
const sourceDirection = new THREE.Vector3(0, -1, 0).normalize();
const targetDirection = new THREE.Vector3(x, y, z).normalize();

const targetQuaternion = new THREE.Quaternion()
  .setFromUnitVectors(sourceDirection, targetDirection);
```

Eklemin dünya koordinatlarında değil, ebeveyn ekleme bağlı yerel koordinat sisteminde hareket etmesi gerekir.

```javascript
const parentInverse = parentWorldQuaternion.clone().invert();

const localTargetQuaternion = parentInverse
  .multiply(targetWorldQuaternion);
```

Hareketin ani sıçramalar üretmemesi için quaternion hedefleri doğrudan uygulanmaz.

```javascript
joint.quaternion.slerp(
  targetQuaternion,
  smoothingFactor
);
```

Projede eklem açısına, kamera kare hızına ve hareket büyüklüğüne göre değişen adaptif bir yumuşatma yaklaşımı kullanılmaktadır.

---

## Hareket İzolasyonu

Her çalışma modu yalnızca ihtiyaç duyduğu robot parçalarını kontrol eder.

| Mod       | Aktif bölgeler                         | Standby bölgeleri                            |
| --------- | -------------------------------------- | -------------------------------------------- |
| Hand Pose | Omuz, dirsek, bilek, avuç ve parmaklar | Baş, gövde ve bacaklar                       |
| Face Mesh | Baş, gözler, kaşlar, ağız ve çene      | Kollar, gövde ve bacaklar                    |
| Body Pose | Baş, gövde, kollar ve bacaklar         | Ayrıntılı yüz mimikleri ve parmaklar         |
| Fusion    | Tüm desteklenen eklem grupları         | Algılanmayan veya güvenilir olmayan bölgeler |

Mod değiştirilirken önceki moddan kalan eklem hedefleri temizlenir. Böylece robotun ilgisiz uzuvlarının rastgele veya bozuk pozlarda kalması önlenir.

---

## Teknolojiler

* JavaScript
* HTML5
* CSS3
* Three.js
* WebGL
* MediaPipe Tasks Vision
* Hand Landmarker
* Face Landmarker
* Pose Landmarker
* Quaternion tabanlı eklem kontrolü
* One Euro benzeri hareket filtreleme
* `requestAnimationFrame`
* `requestVideoFrameCallback`
* Web Camera API
* Canvas API

---

## Gizlilik

Kamera görüntüleri mümkün olan durumlarda doğrudan kullanıcının tarayıcısında işlenir.

* Kamera görüntüleri uygulama tarafından kaydedilmez.
* Kamera görüntüleri proje sahibi tarafından depolanmaz.
* Kullanıcının açık izni olmadan kamera başlatılmaz.
* Kamera erişimi tarayıcı izin sistemi üzerinden yönetilir.
* Uygulama kapatıldığında kamera akışı durdurulur.

> Kamera ve landmark verilerinin işlenme biçimi, kullanılan MediaPipe dağıtımına ve tarayıcı ortamına göre değişebilir.

---

## Çalıştırma

Modern bir Chromium tabanlı tarayıcı önerilir:

* Google Chrome
* Microsoft Edge
* Brave
* Chromium

Safari desteği, kullanılan MediaPipe ve WebGL özelliklerine göre sınırlı olabilir.

### 1. Depoyu klonlayın

```bash
git clone <REPOSITORY_URL>
cd <REPOSITORY_FOLDER>
```

### 2. Yerel sunucuyu başlatın

Python 3 kullanarak:

```bash
python3 -m http.server 8080
```

Windows ortamında:

```bash
python -m http.server 8080
```

### 3. Tarayıcıda açın

```text
http://localhost:8080
```

Kamera erişiminin çalışabilmesi için uygulamanın `localhost` veya HTTPS üzerinden açılması gerekir. HTML dosyasını doğrudan `file://` protokolüyle açmak kamera ve JavaScript modülü hatalarına neden olabilir.

---

## Kullanım

1. Uygulamayı yerel sunucu üzerinden açın.
2. Kamera erişimine izin verin.
3. Hand Pose, Face Mesh, Body Pose veya Fusion modlarından birini seçin.
4. Kamera karşısında ilgili vücut bölgesini görünür tutun.
5. Gerekirse kalibrasyon seçeneğini kullanın.
6. Robot hareketlerini ve landmark bağlantılarını gerçek zamanlı inceleyin.
7. Kamera kullanımınız bittiğinde kamera akışını durdurun.

---

## En İyi Sonuç İçin

* Ortamın yeterince aydınlık olmasını sağlayın.
* Kamera karşısında tek kişinin bulunması önerilir.
* Eller ile gövde arasında yeterli mesafe bırakın.
* Parmakların birbirini kapatmasını önleyin.
* Body Pose modunda tüm vücudu kamera kadrajında tutun.
* Face Mesh modunda yüzün doğrudan kameraya bakması doğruluğu artırır.
* Arka plan ile kıyafet arasında belirgin renk farkı bulunması algılamayı kolaylaştırabilir.
* Çok düşük donanımlı cihazlarda diğer tarayıcı sekmelerini kapatın.

---

## Projenin Eğitim Amaçları

Bu proje aşağıdaki konuların uygulamalı olarak incelenmesini destekler:

* Bilgisayarlı görü
* İnsan pozu tahmini
* El ve parmak takibi
* Yüz landmark analizi
* 3B koordinat sistemleri
* Vektör matematiği
* Quaternion dönüşümleri
* İleri ve ters kinematik
* Eklem hiyerarşileri
* Gerçek zamanlı animasyon
* Hareket filtreleme
* İnsan–robot etkileşimi
* Tarayıcı tabanlı yapay zekâ uygulamaları

---

## Sınırlılıklar

Bu proje tıbbi, biyometrik, güvenlik amaçlı veya endüstriyel robot kontrol sistemi değildir.

Algılama performansı aşağıdaki etkenlere bağlı olarak değişebilir:

* Kamera çözünürlüğü
* Işık koşulları
* Kullanıcının kamera içindeki konumu
* Vücut parçalarının birbirini kapatması
* Tarayıcı performansı
* Grafik işlemci kapasitesi
* MediaPipe model performansı
* Cihazın kare işleme hızı

Robot hareketleri eğitimsel bir görselleştirmedir ve fiziksel robot güvenlik sistemi olarak kullanılamaz.

---

## Telif Hakkı ve Kullanım Koşulları

```text
Copyright © 2026 AI’Han Academy — Ayhan Bozkurt
Tüm hakları saklıdır.
```

Bu proje, kaynak kodu, arayüzü, robot tasarımı, görsel kimliği, metinleri, animasyon sistemi ve proje kapsamında sunulan diğer tüm içerikler yalnızca eğitim, öğretim, akademik inceleme ve kişisel öğrenme amaçlarıyla kullanılabilir.

Aşağıdaki kullanımlara yazılı izin verilmemektedir:

* Ticari kullanım
* Satış veya yeniden satış
* Ücretli ürün ya da hizmetlere entegrasyon
* Ücretli eğitimlerde izinsiz kullanım
* Yeniden lisanslama
* Kaynak kodun ticari bir projede kullanılması
* Projenin değiştirilerek ticari ürün olarak yayımlanması
* Robot karakterinin veya görsel kimliğinin izinsiz kullanılması
* Kaynak gösterilmeden kopyalanması veya dağıtılması
* Projenin tamamının ya da önemli bölümlerinin başka bir adla yayımlanması

Eğitim amaçlı kullanım sırasında aşağıdaki atıf korunmalıdır:

```text
AI’Han Computer Vision Robotics Lab
AI’Han Academy — Ayhan Bozkurt
```

Bu depo açık kaynak lisansıyla yayımlanmamaktadır. Kaynak kodun görüntülenebilir veya erişilebilir olması; kodun ticari olarak kullanılabileceği, yeniden dağıtılabileceği, yeniden lisanslanabileceği ya da sahiplenilebileceği anlamına gelmez.

Ticari kullanım, kurumsal lisanslama, eğitim ortaklığı veya özel kullanım izni için proje sahibinden önceden yazılı izin alınmalıdır.

---

## Sorumluluk Reddi

Proje eğitim ve deneyimleme amacıyla sunulmaktadır.

Yazılımın belirli bir amaca uygunluğu, kesintisiz çalışması, tüm cihazlarda aynı performansı göstermesi veya algılama sonuçlarının mutlak doğruluğu garanti edilmez.

Kullanıcılar projeyi kendi sorumlulukları altında kullanır. Proje sahibi; doğrudan veya dolaylı veri kaybı, donanım sorunu, tarayıcı uyumsuzluğu, yanlış algılama ya da uygulamanın kullanımından doğabilecek diğer sonuçlardan sorumlu tutulamaz.

---

## Geliştirici ve Hak Sahibi

**Ayhan Bozkurt**
**AI’Han Academy**

---

# English

## About the Project

**AI’Han Computer Vision Robotics Lab** is an interactive educational application developed to demonstrate how real-time human motion data generated by computer vision systems can be transferred to a three-dimensional robot.

The application combines:

* MediaPipe hand tracking,
* facial landmark and expression analysis,
* body pose estimation,
* Three.js-based 3D rendering,
* quaternion transformations,
* skeletal joint hierarchies,
* motion filtering,
* real-time robot control.

At the center of the project is **AI’Han**, a three-dimensional educational robot that follows movements detected through the user’s camera and converts them into robotic joint rotations.

---

## Core Features

### Hand Pose Tracking

Hand Pose mode controls only the hand and arm chains.

* Independent right and left hand tracking
* Shoulder, elbow, wrist, palm and finger synchronization
* Finger curl estimation from hand landmarks
* Pose-assisted shoulder–elbow–wrist tracking
* Controlled neutral return when a hand is not detected
* Standby state for the face, torso and legs

### Face Mesh Tracking

Face Mesh mode automatically moves the camera to a closer view of AI’Han’s face.

Supported facial controls include:

* Head orientation
* Eye blinking
* Eyebrow movement
* Mouth and jaw opening
* Smile intensity
* Smoothed facial animation

The arms, torso and legs remain in a stable standby pose while Face Mesh mode is active.

### Body Pose Tracking

Body Pose mode transfers MediaPipe Pose Landmarker data to the robot skeleton.

Main controlled regions include:

* Head and neck
* Shoulders
* Upper and lower arms
* Wrists
* Torso
* Hips
* Upper and lower legs
* Knees
* Ankles
* Foot directions

Landmarks with insufficient confidence or visibility are not directly applied to robot joints.

### Fusion Mode

Fusion mode combines three computer vision systems:

* **Pose Landmarker:** Body, shoulders, arms and legs
* **Hand Landmarker:** Palms, wrist orientation and fingers
* **Face Landmarker:** Head orientation and facial expressions

Each model is assigned ownership of specific joint groups. This prevents multiple models from attempting to control the same joint simultaneously.

---

## Synchronization Engine

AI’Han does not directly copy MediaPipe coordinates to robot joints.

The processing pipeline includes:

```text
Camera frame
    ↓
MediaPipe landmark inference
    ↓
Confidence and visibility validation
    ↓
Coordinate normalization
    ↓
Direction and bone vector calculation
    ↓
Local joint-space transformation
    ↓
Quaternion target calculation
    ↓
Motion filtering and prediction
    ↓
Frame-rate-independent quaternion damping
    ↓
Three.js robot joint hierarchy
```

### Quaternion Conversion

The rotational difference between the default direction of a robot bone and the direction detected by MediaPipe is represented as a quaternion.

```javascript
const sourceDirection = new THREE.Vector3(0, -1, 0).normalize();
const targetDirection = new THREE.Vector3(x, y, z).normalize();

const targetQuaternion = new THREE.Quaternion()
  .setFromUnitVectors(sourceDirection, targetDirection);
```

Joint rotations must be converted into the local coordinate system of their parent joint.

```javascript
const parentInverse = parentWorldQuaternion.clone().invert();

const localTargetQuaternion = parentInverse
  .multiply(targetWorldQuaternion);
```

Quaternion targets are interpolated instead of being applied immediately.

```javascript
joint.quaternion.slerp(
  targetQuaternion,
  smoothingFactor
);
```

The project uses adaptive smoothing based on joint angle, motion magnitude and rendering frame rate.

---

## Mode Isolation

Each operating mode controls only the robot regions required for that task.

| Mode      | Active regions                               | Standby regions                         |
| --------- | -------------------------------------------- | --------------------------------------- |
| Hand Pose | Shoulders, elbows, wrists, palms and fingers | Head, torso and legs                    |
| Face Mesh | Head, eyes, eyebrows, mouth and jaw          | Arms, torso and legs                    |
| Body Pose | Head, torso, arms and legs                   | Detailed facial expressions and fingers |
| Fusion    | All supported joint groups                   | Missing or unreliable regions           |

When the operating mode changes, previous joint targets are cleared. This prevents unrelated robot parts from remaining in invalid or distorted poses.

---

## Technologies

* JavaScript
* HTML5
* CSS3
* Three.js
* WebGL
* MediaPipe Tasks Vision
* Hand Landmarker
* Face Landmarker
* Pose Landmarker
* Quaternion-based joint control
* One Euro-style motion filtering
* `requestAnimationFrame`
* `requestVideoFrameCallback`
* Web Camera API
* Canvas API

---

## Privacy

Camera frames are intended to be processed locally within the user’s browser whenever supported by the selected MediaPipe implementation.

* Camera frames are not intentionally recorded by the application.
* Camera frames are not stored by the project owner.
* The camera does not start without user permission.
* Camera access is controlled through the browser permission system.
* The camera stream is stopped when the application is closed or camera tracking is disabled.

> The exact processing behavior may vary depending on the MediaPipe distribution and browser environment used.

---

## Running the Project

A modern Chromium-based browser is recommended:

* Google Chrome
* Microsoft Edge
* Brave
* Chromium

Safari support may be limited depending on the MediaPipe and WebGL features being used.

### 1. Clone the repository

```bash
git clone <REPOSITORY_URL>
cd <REPOSITORY_FOLDER>
```

### 2. Start a local server

Using Python 3:

```bash
python3 -m http.server 8080
```

On Windows:

```bash
python -m http.server 8080
```

### 3. Open the application

```text
http://localhost:8080
```

The application should be served through `localhost` or HTTPS for camera access. Opening the HTML document directly through the `file://` protocol may cause camera permission and JavaScript module errors.

---

## Usage

1. Open the application through a local server.
2. Allow camera access.
3. Select Hand Pose, Face Mesh, Body Pose or Fusion mode.
4. Keep the relevant body region visible in the camera frame.
5. Use calibration when required.
6. Observe robot joints and landmark connections in real time.
7. Stop the camera stream when tracking is no longer required.

---

## Recommended Conditions

* Use the application in a well-lit environment.
* Only one person should remain in the camera frame where possible.
* Keep sufficient distance between the hands and torso.
* Avoid overlapping fingers.
* Keep the full body visible during Body Pose mode.
* Look toward the camera during Face Mesh mode.
* Use clothing that contrasts with the background.
* Close unnecessary browser tabs on lower-performance devices.

---

## Educational Objectives

This project supports the practical exploration of:

* Computer vision
* Human pose estimation
* Hand and finger tracking
* Facial landmark analysis
* 3D coordinate systems
* Vector mathematics
* Quaternion transformations
* Forward and inverse kinematics
* Skeletal joint hierarchies
* Real-time animation
* Motion filtering
* Human–robot interaction
* Browser-based artificial intelligence

---

## Limitations

This project is not a medical, biometric, security or industrial robot control system.

Tracking performance may vary depending on:

* Camera resolution
* Lighting conditions
* User position
* Occlusion
* Browser performance
* Graphics processor capacity
* MediaPipe model performance
* Device inference speed

Robot movement is an educational visualization and must not be used as a physical robot safety or control system.

---

## Copyright and Terms of Use

```text
Copyright © 2026 AI’Han Academy — Ayhan Bozkurt
All rights reserved.
```

This project, including its source code, interface, robot design, visual identity, text, animation system and all other included materials, may only be used for education, teaching, academic review and personal learning.

The following uses are not permitted without prior written authorization:

* Commercial use
* Sale or resale
* Integration into paid products or services
* Unauthorized use in paid training programs
* Relicensing
* Use of the source code in commercial projects
* Publication of a modified version as a commercial product
* Unauthorized use of the AI’Han character or visual identity
* Copying or distribution without attribution
* Republishing the project or substantial parts of it under another name

The following attribution must be retained in educational use:

```text
AI’Han Computer Vision Robotics Lab
AI’Han Academy — Ayhan Bozkurt
```

This repository is not distributed under an open-source license. The fact that the source code may be visible or accessible does not grant permission to commercially use, redistribute, relicense or claim ownership of the project.

Commercial licensing, institutional use, training partnerships and other special permissions require prior written authorization from the copyright holder.

---

## Disclaimer

The project is provided for educational and experimental purposes.

No guarantee is made regarding uninterrupted operation, compatibility with every device, fitness for a particular purpose or absolute accuracy of tracking results.

Users access and operate the project at their own risk. The project owner shall not be held responsible for direct or indirect data loss, hardware issues, browser incompatibilities, incorrect tracking results or other consequences resulting from the use of the application.

---

## Developer and Copyright Holder

**Ayhan Bozkurt**
**AI’Han Academy**

---

```text
AI’Han Computer Vision Robotics Lab
AI’Han Academy — Ayhan Bozkurt
Copyright © 2026
All rights reserved.
Educational use only.
Commercial use is prohibited without prior written permission.
```

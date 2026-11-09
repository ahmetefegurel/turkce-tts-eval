# Türkçe TTS Değerlendirme Harness

Türkçe metin okuma (TTS) sistemlerini insana bağımlı olmadan ölçen, tamamen yerelde çalışan bir değerlendirme akışı. Hiçbir veri dışarıdaki bir API'ye gitmez (KVKK uyumlu çalışma için tasarlandı).

Bir banka stajında (20 gün, 2026 yazı) geliştirildi. Repodaki test cümleleri, ses dosyaları ve sonuçlar örnek amaçlıdır, gerçek müşteri verisi içermez.

> **In English:** A self-hosted evaluation harness for Turkish text-to-speech. It round-trips TTS audio through two ASR models, scores it with character error rate (CER) and a field-level metric for numbers, dates and currency (CFER), and uses UTMOSv2 as a relative naturalness signal. Main finding: critical-field errors drop from 54% to 0% when a Turkish-specific normalization layer converts digits to spoken form before synthesis.

## Problem

Bir TTS sistemi "Kartınızdan 49,90 TL tutarında alışveriş yapıldı" cümlesini yanlış okursa müşteri yanlış tutarı duyar. Cümlenin geri kalanı kusursuz olsa bile bu ciddi bir hatadır. Ama bunu her sürümde insanlara dinleterek ölçmek yavaş ve pahalı.

Bu proje şu soruyu cevaplıyor: TTS çıktısının doğru okunup okunmadığını otomatik olarak nasıl ölçeriz?

## Nasıl çalışıyor

Üç bacak var:

1. **Okuma doğruluğu.** TTS'in ürettiği ses bir konuşma tanıma (ASR) modeline geri verilir, çıkan metin orijinal metinle karşılaştırılır. Karşılaştırmadan önce iki taraf da aynı sözlü forma çevrilir ("1.500 TL" → "bin beş yüz lira").
2. **Doğallık.** UTMOSv2 ile tahmini MOS skoru. Model Türkçe ile eğitilmediği için mutlak skora güvenilmez, sadece iki sistemi ya da iki sürümü kıyaslamak için kullanılır.
3. **İnsan kontrolü.** Hiçbir otomatik metriğin yakalayamadığı durumlar insana gider. Örnek: "kâr" ve "kar" aynı yazılır, ASR ikisini de "kar" yazar, hata görünmez.

```
metin ──► normalizasyon ──► TTS ──► .wav ──┬──► Whisper ──┐
                                            ├──► wav2vec2 ─┼──► CER + CFER ──► HTML rapor
                                            └──► UTMOSv2 ──┘
```

### Neden iki ASR?

Whisper duyduğunu değil, düzelttiğini yazar. TTS "yirmi dokuz nokta sıfır altı" dese bile Whisper "29.06" yazabilir. Bu durumda hata oranı sıfır çıkar, gerçek okuma hatası görünmez. wav2vec2 düzeltme yapmaz, duyduğu sesi yazar. İki modelin sonucu belirgin şekilde ayrışırsa sistem "kör nokta" bayrağı kaldırır.

### Neden WER değil CER?

Türkçe sondan eklemeli bir dil. "Bankasında" ile "bankası nda" arasındaki küçük fark WER'de bütün kelimeyi yanlış sayar. CER karakter seviyesinde baktığı için gerçek hatayı daha adil ölçer.

### CFER nedir?

Critical Field Error Rate. CER "sesin yüzde kaçı bozuk" der. CFER "kritik değerlerin (tarih, saat, para, sayı, kısaltma) kaçı yanlış okundu" der. Kritik alanlar referans metinden regex ile otomatik bulunur, elle etiketleme gerekmez.

## Ana bulgu

| Koşu | CFER |
|---|---|
| Piper, ham metin | %54 |
| VoxCPM, ham metin | %54 |
| VoxCPM, modelin kendi normalizasyonu | %77 |
| Piper + Türkçe normalizasyon katmanı | %0 |
| VoxCPM + Türkçe normalizasyon katmanı | %0 |

- Daha büyük model almak sorunu çözmüyor. VoxCPM (2B parametre) Piper ile aynı hatayı yapıyor.
- Modelin kendi normalizasyonunu açmak işi kötüleştiriyor. Türkçe rakamları İngilizce okuyor ("49,90 TL" → "forty nine ninety").
- Rakamları TTS'e vermeden önce Türkçe sözlü forma çeviren ayrı bir katman, iki sistemde de hatayı sıfırlıyor.

Sonuç: sorun model kalitesinde değil, TTS'e rakam gösterilmesinde.

**Sınır:** Bu deney sözlü formun doğru okunduğunu kanıtlıyor, sözlü formun kendisinin doğru olduğunu kanıtlamıyor ("09:00" için "dokuz" mu, "saat dokuz" mu?). Ayrıntılar için [DEGISIKLIKLER.md](DEGISIKLIKLER.md).

## Raporlar

HTML raporları indirip tarayıcıda açın. Her cümle için iki sistemin sesi, ASR çıktıları ve fark görünümü yan yana gelir.

| Dosya | İçerik |
|---|---|
| `rapor_ab.html` | Piper vs VoxCPM |
| `rapor_katman.html` | Piper ham vs Piper + katman |
| `rapor_katman_voxcpm.html` | VoxCPM ham vs VoxCPM + katman |
| `rapor_norm.html` | VoxCPM ham vs VoxCPM kendi normalizasyonu (İngilizce okuma) |

Farkı en net gösteren yer `cikti_11` kartı.

## Kurulum

Python 3.10+ gerekir.

```bash
pip install -r requirements.txt
```

Sadece normalizasyon ve metrik motorunu denemek için ağır paketlere gerek yok:

```bash
python3 turkish_tts_eval.py --demo
python3 alan_cfer.py piper.json piper_norm.json --ad-a "Piper ham" --ad-b "Piper + katman"
```

TTS modelleri repoda yok. Piper için `tr_TR-dfki-medium.onnx` ve `.onnx.json` dosyalarını proje klasörüne indirin. VoxCPM için bir referans ses dosyası (`--ref-wav`) gerekir.

## Uçtan uca çalıştırma

```bash
# 1. Test setini seslendir (--on-normalize: Türkçe katmanı aç)
python3 synth_piper.py --out sesler
python3 synth_piper.py --out sesler_piper_norm --on-normalize

# 2. Sesleri puanla
python3 score_wav.py --batch sesler --refs sesler/refs.tsv --json piper.json
python3 score_wav.py --batch sesler_piper_norm --refs sesler_piper_norm/refs.tsv --json piper_norm.json

# 3. CFER karşılaştır
python3 alan_cfer.py piper.json piper_norm.json --ad-a "Piper ham" --ad-b "Piper + katman"

# 4. HTML rapor
python3 rapor.py piper.json piper_norm.json \
  --ad-a "Piper ham" --ad-b "Piper + katman" \
  --ses-a sesler --ses-b sesler_piper_norm -o rapor.html
```

## Dosyalar

| Dosya | Görev |
|---|---|
| `turkish_tts_eval.py` | Türkçe normalizasyon katmanı, CER/WER, 22 cümlelik test seti |
| `score_wav.py` | Skorlayıcı (Whisper + wav2vec2 + UTMOSv2) |
| `alan_cfer.py` | Alan bazlı CFER ve ön-normalizasyon |
| `gerekce.py` | Hatalar için deterministik sebep kodları |
| `rapor.py` | HTML skor kartı |
| `karsilastir.py` | İki koşuyu terminalde karşılaştırır |
| `synth_piper.py`, `synth_voxcpm.py` | Test setini seslendiren üreticiler |
| `dinleme_seti.py` | Kör dinleme (MOS) seti hazırlar, cevap anahtarını ayrı tutar |
| `deney_oku.py` | VoxCPM normalizasyon deneyini okur (baseline, kontrol, deney) |
| `test_w2v2.py` | wav2vec2 ile tek dosya hızlı test |
| `*.json` | Koşu sonuçları (her biri bir TTS + ayar kombinasyonu) |
| `gerekce_*.txt`, `deney_normalizasyon.txt` | Kural motoru ve deney çıktıları (terminal dökümü) |
| `mos_capa_ozet.json` | Çapalı UTMOSv2 ölçümü özeti |
| `BULGU_normalizasyon.md`, `DEGISIKLIKLER.md` | Deney notları ve sürüm değişiklikleri |
| `sesler*/` | Üretilen sesler ve referans metinler (`refs.tsv`) |

## Lisanslar ve araçlar

Sadece ticari kullanıma uygun lisanslı ve self-hosted çalışabilen araçlar seçildi.

| Araç | Rol | Lisans |
|---|---|---|
| faster-whisper | Ana ASR | MIT |
| mpoyraz/wav2vec2-xls-r-300m-cv8-turkish | İkinci ASR | Apache 2.0 |
| VoxCPM | TTS | Apache 2.0 |
| Piper | TTS | Ses modelinin lisansı model kartından teyit edilmeli |
| UTMOSv2 | Doğallık | Ürün kullanımı öncesi teyit edilmeli |

NISQA kullanılmadı, ağırlıkları ticari kullanıma açık değil.

## Açık kalemler

- Kör dinleme testi henüz insanlara dağıtılmadı.
- Normalizasyon hedef formları (saat, kısaltma okunuşu) ürün ekibiyle netleştirilmeli.
- Kısaltma okunuşu: sözlük "ka ve ka ka" diyor, Piper "ke ve ke ke" okuyor.

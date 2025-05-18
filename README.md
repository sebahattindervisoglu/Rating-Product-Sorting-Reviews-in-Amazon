# Rating Product & Sorting Reviews in Amazon

## İş Problemi

E-ticaret sitelerinde ürünlere verilen puanların doğru hesaplanması, müşteri memnuniyeti, satıcı başarısı ve satışların artması için kritik öneme sahiptir.  
Ayrıca, ürün yorumlarının doğru ve faydalı şekilde sıralanması, kullanıcıların güvenli ve hızlı alışveriş deneyimi yaşamalarını sağlar.  
Yanıltıcı veya alakasız yorumların öne çıkması hem maddi kayıplara hem de müşteri güveninin azalmasına yol açar.

## Veri Seti

Elektronik kategorisindeki Amazon ürünlerine ait kullanıcı puanları ve yorumları içermektedir.  
Başlıca değişkenler:

- `reviewerID`: Kullanıcı ID'si  
- `asin`: Ürün ID'si  
- `reviewerName`: Kullanıcı adı  
- `helpful`: Yorumun faydalı bulunma sayısı  
- `reviewText`: Yorum metni  
- `overall`: Ürün puanı (rating)  
- `summary`: Yorum özeti  
- `unixReviewTime`: Yorumun zamanı (Unix formatında)  
- `reviewTime`: Yorumun zamanı (okunabilir formatta)  
- `day_diff`: Yorumun yapıldığı tarihten bugüne geçen gün sayısı  
- `helpful_yes`: Faydalı oy sayısı  
- `total_vote`: Toplam oy sayısı  

## Çözüm Yaklaşımı

1. **Güncel Yorumlara Göre Ortalama Puan Hesaplama:**  
   Tarihe göre ağırlıklı puan hesaplanarak ürünlerin ortalama puanları güncel yorumlara daha fazla önem verilerek yeniden hesaplanmıştır.

2. **En Faydalı 20 Yorumu Belirleme:**  
   Yorumların faydalılıklarına göre sıralanması için Wilson Lower Bound skoru kullanılmıştır. Böylece hem faydalı oy sayısı hem de güven aralığı dikkate alınarak yorumların sıralaması optimize edilmiştir.

## Örnek Kod

```python
import pandas as pd
import math
import scipy.stats as st

# Veri seti okunuyor
df = pd.read_csv("Datasets/amazon_review.csv")

# Tarihe göre ağırlıklı ortalama puan hesaplama
weighted_avg = df.loc[df["day_diff"] <= 30, "overall"].mean() * 0.28 + \
               df.loc[(df["day_diff"] > 30) & (df["day_diff"] <= 90), "overall"].mean() * 0.26 + \
               df.loc[(df["day_diff"] > 90) & (df["day_diff"] <= 180), "overall"].mean() * 0.24 + \
               df.loc[df["day_diff"] > 180, "overall"].mean() * 0.22

print(f"Tarihe göre ağırlıklı ortalama puan: {weighted_avg:.4f}")

# Faydalı ve faydasız oy sayısı hesaplanıyor
df["helpful_no"] = df["total_vote"] - df["helpful_yes"]

def wilson_lower_bound(up, down, confidence=0.95):
    n = up + down
    if n == 0:
        return 0
    z = st.norm.ppf(1 - (1 - confidence) / 2)
    phat = up / n
    return (phat + z*z/(2*n) - z * math.sqrt((phat*(1-phat) + z*z/(4*n))/n)) / (1 + z*z/n)

df["wilson_lower_bound"] = df.apply(lambda x: wilson_lower_bound(x["helpful_yes"], x["helpful_no"]), axis=1)

# En faydalı 20 yorumu seçme
top_reviews = df.sort_values("wilson_lower_bound", ascending=False).head(20)
print(top_reviews[["reviewText", "wilson_lower_bound"]])

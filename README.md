STM32F4 CMSIS, Timer (TIM2) Tabanlı LED Kontrolü

Bu proje, STM32F4 mikrodenetleyicisinde **timer interrupt** mekanizmasını kullanarak LED kontrolü yapmayı öğretir. TIM2 timer'ının nasıl konfigüre edileceği, kesme işleyicilerinin nasıl yazılacağı ve timer tabanlı periyodik işlemlerin nasıl gerçekleştirileceği detaylı olarak açıklanmıştır.

Timer Odaklı Öğrenme Hedefleri
  - STM32 timer yapısını anlamak
  - Timer saat hesaplamalarını öğrenmek  
  - Prescaler ve ARR kavramlarını kavramak
  - Timer interrupt konfigürasyonu
  - NVIC (Nested Vectored Interrupt Controller) kullanımı
  - Kesme tabanlı programlama yaklaşımı

STM32F4 Timer Hiyerarşisi
  ```
  System Clock (168 MHz)
      ↓
  AHB Bus (168 MHz)
      ↓
  APB1 Bus (42 MHz) → Timer Clock = APB1 × 2 = 84 MHz (APB1 prescaler ≠ 1 olduğunda)
      ↓
  TIM2/3/4/5 (General Purpose Timers)
  ```

Timer Clock Hesaplama Kuralları
  APB1 Prescaler = 1 ise:
    ```
    Timer Clock = APB1 Clock
    ```
  
  APB1 Prescaler ≠ 1 ise:
    ```
    Timer Clock = APB1 Clock × 2
    ```
  
  Bizim durumumuzda:
    ```
    System Clock = 168 MHz
    APB1 Prescaler = 4
    APB1 Clock = 168 MHz ÷ 4 = 42 MHz
    Timer Clock = 42 MHz × 2 = 84 MHz
    ```

Timer Parametrelerinin Hesaplanması
  Temel Formül
    ```
    Output Frequency = Timer Clock ÷ (Prescaler + 1) ÷ (ARR + 1)
    ```
  
  1 Hz Elde Etmek İçin Hesaplama
  
  Veriler:
    - Timer Clock = 84 MHz
    - İstenen Frekans = 1 Hz
  
  Adım 1: Prescaler Seçimi
    ```
    Prescaler = 8400 - 1 = 8399 (register değeri)
    Gerçek Bölme = 8400
    Prescaled Clock = 84,000,000 ÷ 8400 = 10,000 Hz
    ```
  
  Adım 2: ARR Hesaplama
    ```
    ARR = (Prescaled Clock ÷ İstenen Frekans) - 1
    ARR = (10,000 ÷ 1) - 1 = 9999
    ```
  
  Doğrulama:
    ```
    Çıkış Frekansı = 84,000,000 ÷ 8400 ÷ 10000 = 1 Hz ✓
    ```

TIM2 Konfigürasyon Adımları
  1. Timer Clock Enable
    ```c
    RCC->APB1ENR |= RCC_APB1ENR_TIM2EN;  // TIM2 saatini etkinleştir
    (void)RCC->APB1ENR;                  // Clock sync için readback
    ```
  
  2. Prescaler ve Period Ayarları
    ```c
    TIM2->PSC = 8399;    // Prescaler: 84MHz → 10kHz
    TIM2->ARR = 9999;    // Auto-Reload: 10kHz → 1Hz
    ```
  
  3. Interrupt Konfigürasyonu
    ```c
    TIM2->DIER |= TIM_DIER_UIE;          // Update interrupt enable
    ```
  
  4. NVIC Ayarları
    ```c
    NVIC_SetPriority(TIM2_IRQn, 2U);     // Öncelik seviyesi
    NVIC_EnableIRQ(TIM2_IRQn);           // Interrupt enable
    ```
  
  5. Timer Başlatma
    ```c
    TIM2->CR1 |= TIM_CR1_CEN;            // Counter enable
    ```

Timer Register'ları Detayı
  PSC (Prescaler Register)
    - Boyut: 16-bit (0-65535)
    - Fonksiyon: Timer saatini böler
    - Hesaplama: Gerçek bölme = PSC + 1
    - Örnek: PSC = 8399 → 8400 ile bölme
  
  ARR (Auto-Reload Register)
    - Boyut: TIM2 için 32-bit, diğerleri 16-bit
    - Fonksiyon: Sayacın maximum değeri
    - Hesaplama: Period = ARR + 1
    - Örnek: ARR = 9999 → 10000 count'ta overflow
  
  CNT (Counter Register)
    - Boyut: 32-bit (TIM2 için)
    - Fonksiyon: Anlık sayaç değeri
    - Davranış: 0'dan ARR'a kadar saydırır, sonra sıfırlanır
  
  CR1 (Control Register 1)
    ```c
    TIM_CR1_CEN    // Counter Enable
    TIM_CR1_UDIS   // Update Disable  
    TIM_CR1_URS    // Update Request Source
    TIM_CR1_OPM    // One Pulse Mode
    TIM_CR1_DIR    // Direction (0=up, 1=down)
    TIM_CR1_ARPE   // Auto-Reload Preload Enable
    ```
  
  DIER (DMA/Interrupt Enable Register)
    ```c
    TIM_DIER_UIE   // Update Interrupt Enable
    TIM_DIER_CC1IE // Capture/Compare 1 Interrupt Enable
    // ... diğer kanal kesmeleri
    ```
  
  SR (Status Register)
    ```c
    TIM_SR_UIF     // Update Interrupt Flag
    TIM_SR_CC1IF   // Capture/Compare 1 Interrupt Flag  
    // ... diğer durum bayrakları
    ```

Timer Interrupt İşleyici (ISR)
  Interrupt Akışı
    ```
    1. Timer overflow (CNT = ARR)
       ↓
    2. Hardware UIF bit'ini set eder
       ↓  
    3. NVIC TIM2_IRQHandler()'ı çağırır
       ↓
    4. Software UIF bit'ini temizler
       ↓
    5. İş yapılır (LED toggle)
       ↓
    6. ISR'dan çıkılır
    ```
  
  ISR Yazım Kuralları
    ```
    
    void TIM2_IRQHandler(void)
    {
        // 1. Flag kontrol et
        if (TIM2->SR & TIM_SR_UIF) {
            
            // 2. Flag'i hemen temizle
            TIM2->SR &= ~TIM_SR_UIF;
            
            // 3. Kısa işlem yap
            GPIOD->ODR ^= (1 << 12);
            
            // 4. Uzun işlemler için flag set et
            // volatile uint8_t process_flag = 1;
        }
    }
    ```

Farklı Frekanslar İçin Hesaplama Örnekleri
  10 Hz İçin:
    ```c
    uint32_t prescaler = 8400 - 1;      // 84MHz → 10kHz
    uint32_t arr = 1000 - 1;            // 10kHz → 10Hz
    ```
  
  100 Hz İçin:
    ```c
    uint32_t prescaler = 840 - 1;       // 84MHz → 100kHz  
    uint32_t arr = 1000 - 1;            // 100kHz → 100Hz
    ```
  
  1 kHz İçin:
    ```c
    uint32_t prescaler = 84 - 1;        // 84MHz → 1MHz
    uint32_t arr = 1000 - 1;            // 1MHz → 1kHz
    ```

Timer Konfigürasyon Alternatifleri
  Preload Enable ile Güvenli Güncelleme
    ```c
    TIM2->CR1 |= TIM_CR1_ARPE;          // Auto-reload preload enable
    TIM2->PSC = new_prescaler;
    TIM2->ARR = new_arr;
    TIM2->EGR = TIM_EGR_UG;             // Generate update event
    ```

  One-Shot Timer
    ```c
    TIM2->CR1 |= TIM_CR1_OPM;           // One pulse mode
    // Timer bir kez overflow olur ve durur
    ```
  
  Down Counting
    ```c
    TIM2->CR1 |= TIM_CR1_DIR;           // Down counting
    TIM2->CNT = TIM2->ARR;              // Start from top
    ```

Yaygın Hatalar ve Çözümleri
  1. Clock Enable Unutma
    ```

    // YANLIŞ:
    TIM2->PSC = 8399;  // Clock açılmadan register yazma
    
    // DOĞRU:
    RCC->APB1ENR |= RCC_APB1ENR_TIM2EN;
    TIM2->PSC = 8399;
    ```
  
  2. Flag Temizleme Unutma
    ```

    // YANLIŞ:
    void TIM2_IRQHandler(void) {
        GPIOD->ODR ^= (1 << 12);  // Flag temizlenmedi!
    }
    
    // DOĞRU:
    void TIM2_IRQHandler(void) {
        if (TIM2->SR & TIM_SR_UIF) {
            TIM2->SR &= ~TIM_SR_UIF;  // Flag temizlendi
            GPIOD->ODR ^= (1 << 12);
        }
    }
    ```
  
  3. NVIC Konfigürasyon Sırası
    ```

    // YANLIŞ SIRA:
    NVIC_EnableIRQ(TIM2_IRQn);
    TIM2->DIER |= TIM_DIER_UIE;
    
    // DOĞRU SIRA:  
    TIM2->DIER |= TIM_DIER_UIE;
    NVIC_SetPriority(TIM2_IRQn, 2U);
    NVIC_EnableIRQ(TIM2_IRQn);
    ```

Performans Optimizasyonu
  ISR Süresi Minimizasyonu
    ```
    
    // Yavaş yaklaşım:
    void TIM2_IRQHandler(void) {
        if (TIM2->SR & TIM_SR_UIF) {
            TIM2->SR &= ~TIM_SR_UIF;
            
            // Uzun işlem - KÖTÜ!
            for(int i = 0; i < 1000; i++) {
                process_data();
            }
        }
    }
    
    // Hızlı yaklaşım:
    volatile uint8_t timer_flag = 0;
    
    void TIM2_IRQHandler(void) {
        if (TIM2->SR & TIM_SR_UIF) {
            TIM2->SR &= ~TIM_SR_UIF;
            timer_flag = 1;  // Sadece flag set et
        }
    }
    
    int main(void) {
        while(1) {
            if(timer_flag) {
                timer_flag = 0;
                // Uzun işlemleri burada yap
                process_data();
            }
            __WFI();
        }
    }
    ```

Debug ve Test Teknikleri
  Timer Durumunu Kontrol Etme
    ```
    
    // Debug bilgileri
    uint32_t current_count = TIM2->CNT;
    uint32_t prescaler_val = TIM2->PSC;  
    uint32_t reload_val = TIM2->ARR;
    uint16_t status_reg = TIM2->SR;
    ```
  
  Frekans Ölçümü
    ```
    
    // GPIO pin ile frekans çıkışı (osiloskop için)
    void TIM2_IRQHandler(void) {
        if (TIM2->SR & TIM_SR_UIF) {
            TIM2->SR &= ~TIM_SR_UIF;
            
            GPIOD->ODR ^= (1 << 12);  // LED
            GPIOD->ODR ^= (1 << 13);  // Test pin
        }
    }
    ```

İleri Seviye Timer Konuları
  Timer Chaining (Zincirleme)
  - TIM2'yi master, TIM3'ü slave yapma
  - Daha uzun periyotlar için 64-bit timer oluşturma
  
  PWM Generation
  - Timer channel'ları ile PWM üretimi
  - Duty cycle kontrolü
  
  Input Capture
  - Harici sinyal frekansı ölçümü
  - Pulse width measurement
  
  Output Compare
  - Belirli zamanlarda çıkış üretimi
  - Multi-channel timing

Timer Odaklı Öğrenme: Bu README, STM32 timer sistemini derinlemesine anlamanız için hazırlanmıştır. Her bölüm timer kavramlarını pekiştirmek üzere tasarlanmıştır.

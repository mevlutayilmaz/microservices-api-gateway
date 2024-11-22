# Ocelot API Gateway Kullanım Kılavuzu

Bu kılavuz, .NET tabanlı bir API Gateway olan Ocelot'un nasıl kullanılacağını adım adım açıklamaktadır. Ocelot, çeşitli mikro servisleri tek bir giriş noktası altında birleştirerek, istekleri yönlendirerek, yetkilendirme ve güvenlik gibi işlevleri sağlayarak uygulamalarınızın ölçeklenebilirliğini ve yönetilebilirliğini artırmanıza yardımcı olur.

### 1. Kurulum ve Yapılandırma:

* **Proje Oluşturma:**  Ocelot'un kurulumu için öncelikle API Gateway görevi görecek bir Asp.NET Core uygulaması oluşturulmalı ve ardından bu uygulamaya Ocelot kütüphanesi yüklenmelidir.

* **NuGet Paketinin Eklenmesi:**  `Ocelot` paketini NuGet Paket Yöneticisi aracılığıyla ekleyin.

* **Ocelot Yapılandırma Dosyası (`ocelot.json`):**  Ardından API Gateway görevi görecek bu uygulama içerisinde Ocelot'un yapılandırılması gerekmektedir. Bunun için de `ocelot.json` adında bir dosya oluşturulmalı ve içeriği doldurulmalıdır.  Aşağıda basit bir örnek gösterilmiştir:

  ```json
  {
    "Routes": [
      {
        "DownstreamPathTemplate": "/",
        "DownstreamScheme": "https",
        "DownstreamHostAndPorts": [
          {
            "Host": "localhost",
            "Port": 7205
          }
        ],
        "UpstreamPathTemplate": "/api1",
        "UpstreamHttpMethod": [ "GET", "POST" ]
      }
    ],
    "GlobalConfiguration": {
      "BaseUrl": "https://localhost:7130"
    }
  }
  ```

  * **Açıklama:**
      * `Routes`: Ocelot'un yönlendireceği istekleri tanımlar.
      * `DownstreamPathTemplate`:  Yönlendirme yapılacak mikroservis'in route'unu tutmaktadır.
      * `DownstreamScheme`:  İlgili mikroservis'e yapılacak isteğin hangi protokol üzerinden gerçekleştirileceğini bildirmektedir (http veya https).
      * `DownstreamHostAndPorts`:  Mikroservis'in Host ve Port bilgilerini tutmaktadır.
      * `UpstreamPathTemplate`:  API Gateway üzerinden mikroservis'e yapılacak yönlendirmenin template'ini tutmaktadır. Bu template ile API Gateway'e gelen istekler DownstreamPathTemplate'teki route'a yönlendirilecektir.
      * `UpstreamHttpMethod`:  Hangi isteklerin yapılabileceği bildirilmektedir (GET, POST, PUT, DELETE, vb.).
      * `GlobalConfiguration`:  Bu alanda ise Ocelot'un BaseUrl'i belirlenmektedir. Yani bu API Gateway uygulamasının temel adresidir. Client, bu url üzerinden alt servislere istekte bulunacaktır.


### 2. Ocelot'un Entegre Edilmesi:

Ocelot yapılandırıldıktan sonra, oluşturmuş olduğumuz yapılandırma dosyasının uygulamaya dahil edilmesi gerekmektedir. `Program.cs` dosyanızı aşağıdaki gibi düzenleyin:

```csharp
using Ocelot.DependencyInjection;
using Ocelot.Middleware;

var builder = WebApplication.CreateBuilder(args);

builder.Configuration.AddJsonFile("ocelot.json");
builder.Services.AddOcelot(builder.Configuration);

var app = builder.Build();

await app.UseOcelot();
app.UseHttpsRedirection();

app.Run();
```

Bu kod, Ocelot'un yapılandırılmasını ve middleware olarak eklenmesini sağlar. `ocelot.json` dosyasındaki ayarları okur ve istekleri yönlendirir.

Bunların dışında istekleri yönlendirme sürecinde parametreli yönlendirme ve tüm isteklerin yakalanması durumları için aşağıdaki şekilde yapılandırmalarda bulunmak yeterlidir.

**Paremetreli Yönlendirme:**
  ```json
  {
    "Routes": [
      {
        "DownstreamPathTemplate": "/{id}",
        "DownstreamScheme": "https",
        "DownstreamHostAndPorts": [
          {
            "Host": "localhost",
            "Port": 7205
          }
        ],
        "UpstreamPathTemplate": "/api1/{id}",
        "UpstreamHttpMethod": [ "GET", "POST" ]
      }
    ],
    "GlobalConfiguration": {
      "BaseUrl": "https://localhost:7130"
    }
  }
  ```

**Tüm İstekleri Yakalama:**
  ```json
  {
    "Routes": [
      {
        "DownstreamPathTemplate": "/{everything}",
        "DownstreamScheme": "https",
        "DownstreamHostAndPorts": [
          {
            "Host": "localhost",
            "Port": 7205
          }
        ],
        "UpstreamPathTemplate": "/api1/{everything}",
        "UpstreamHttpMethod": [ "GET", "POST" ]
      }
    ],
    "GlobalConfiguration": {
      "BaseUrl": "https://localhost:7130"
    }
  }
  ```

### 3. Mikro Servislerin Çalıştırılması:

Ocelot'un yönlendirme yapması için, `ocelot.json` dosyasında tanımlanan mikro servislerin çalışır durumda olması gerekir.  Bu servisler ayrı projeler olabilir ve bağımsız olarak çalıştırılabilirler.


### 4. Gelişmiş Özellikler:

* **Yetkilendirme:** Ocelot, JWT (JSON Web Token) ve diğer yetkilendirme mekanizmaları ile entegre edilebilir.  `ocelot.json` dosyasında ilgili ayarlar yapılandırılmalıdır.

* **Güvenlik:**  HTTPS kullanımı, rate limiting, ve diğer güvenlik önlemleri Ocelot ile yapılandırılabilir.

* **QoS (Quality of Service):**  Ocelot, isteklerin önceliklendirilmesi, zaman aşımı ayarları gibi QoS özelliklerini destekler.

* **Loglama ve İzleme:**  Ocelot'un loglarını izleyerek uygulamanızın performansını ve hatalarını takip edebilirsiniz.

* **Health Checks:**  Mikro servislerin durumunu kontrol etmek için health check mekanizmaları entegre edilebilir.

* **Cache:**  Ocelot, yanıtları önbelleğe alarak performansı artırabilir.


**5. Docker Kullanımı:**

Docker kullanarak Ocelot ve mikro servislerinizi containerize edebilirsiniz.  Dockerfile oluşturarak, uygulamalarınızı ve bağımlılıklarını image'lere paketleyebilirsiniz.  Docker Compose ile tüm containerları yönetebilirsiniz.


**Örnek `docker-compose.yml`:**

```yaml
version: "3.9"
services:
  ocelot:
    image: <ocelot-image> # Ocelot docker image
    ports:
      - "5000:5000"
    volumes:
      - ./ocelot.json:/app/ocelot.json
    depends_on:
      - products-service
      - users-service

  products-service:
    image: <products-service-image> # Products service docker image
    ports:
      - "5001:5001"

  users-service:
    image: <users-service-image> # Users service docker image
    ports:
      - "5002:5002"
```

**Önemli Notlar:**

* Ocelot'un performansını optimize etmek için, doğru yapılandırma ve uygun kaynak kullanımı önemlidir.
*  Güvenlik açısından, üretim ortamlarında HTTPS kullanımı zorunludur.
*  Mikro servislerinizin adreslerini ve portlarını `ocelot.json` dosyasında doğru bir şekilde belirtmeniz gerekir. Docker kullanıyorsanız, container isimlerini kullanmanız daha pratik olacaktır.
*  Her zaman en son Ocelot sürümünü kullanmaya çalışın ve güncelleme notlarını takip edin.


Bu kılavuz, Ocelot'un temel kullanımını açıklamaktadır. Daha detaylı bilgi için Ocelot'un resmi dokümanlarını inceleyebilirsiniz.  Ayrıca, özel durumlar için gerekli olan konfigürasyonlar ve gelişmiş özellikler için Ocelot'un geniş dokümantasyonuna ve örneklerine bakmanız tavsiye edilir.

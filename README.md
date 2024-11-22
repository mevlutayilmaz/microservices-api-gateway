# Ocelot API Gateway

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

### 4. Authentication & Authentication:

Ocelot'ta authentication ve authorization işlemlerini gerçekleştirmek için genellikle JWT kullanılmaktadır. Bunun için `Microsoft.AspNetCore.Authentication.JwtBearer` kütüphanesinin uygulamaya yüklenmesi gerekmektedir.

Authentication için `ocelot.json`'da aşağıdaki gibi bir yapılandırmada bulunulması gerekmektedir.

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
        "UpstreamHttpMethod": [ "GET", "POST" ],
        "AuthenticationOptions": {
          "AllowedScopes": [],
          "AuthenticationProviderKey": "Bearer"
        }
      }
    ],
    "GlobalConfiguration": {
      "BaseUrl": "https://localhost:7130"
    }
  }
  ```

Örnekte görüldüğü üzere hangi route'da bir kontrol yapılacaksa onun nesnesinde AuthenticationOptions alanının tanımlanması gerekmektedir. Burada AuthenticationProviderKey alanına `Bearer` vererek, kimlik doğrulama opsiyonlarından JWT'nin kullanılacağı bildirilmektedir.

Bu yapılandırmadan sonra API Gateway uygulamasının `Program.cs` dosyasında aşağıdaki yapılandırmanın yapılması gerekmektedir.

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Configuration.AddJsonFile("ocelot.json");
builder.Services.AddOcelot(builder.Configuration);

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(JwtBearerDefaults.AuthenticationScheme, options =>
    {
        options.TokenValidationParameters = new()
        {
            ValidateAudience = true,
            ValidateIssuer = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Token:Issuer"],
            ValidAudience = builder.Configuration["Token:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Token:SecurityKey"])),
            ClockSkew = TimeSpan.Zero
        };
    });

var app = builder.Build();

app.UseAuthentication();
app.UseAuthentication();

await app.UseOcelot();
app.UseHttpsRedirection();

app.Run();
```

Audience, Issuer ve SecurityKey değerlerinin `appsettings.json` içerisinde saklamak daha doğru olacaktır. Bu değerleri kendinize göre özelleştirebilirsiniz.

Böylece API Gateway uygulamasında authentication'ı yapılandırmış bulunuyoruz. Bu yapılandırmalara uygun bir şekilde servislere erişim göstermek isteniyorsa eğer alt servislerde de benzer şekilde  yapılandırmada bulunulması gerekmektedir.

Authorization için ise claim tabanlı aşağıdaki gibi bir yapılandırmanın olduğunu varsayarsak:

```charp
builder.Services.AddAuthorization(options => options.AddPolicy("AdminPolicy", policy => policy.RequireClaim("Role", "Admin")));

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/", () => "API 1")
    .RequireAuthorization("AdminPolicy");

app.Run();
```

Görüldüğü üzere Role claim'ine karşılık Admin değerini politika olarak tanımlamış olduk. 

`ocelot.json`'da da aşağıdaki gibi RouteClaimsRequirement yapılandırmasında bulunulmalıdır.

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
        "UpstreamHttpMethod": [ "GET", "POST" ],
        "AuthenticationOptions": {
          "AllowedScopes": [],
          "AuthenticationProviderKey": "Bearer"
        },
        "RouteClaimsRequirement": {
          "Role": "Admin"
        }
      }
    ],
    "GlobalConfiguration": {
      "BaseUrl": "https://localhost:7130"
    }
  }
  ```
Böylece artık bu yapılandırmanın gerçekleştirildiği route'a 'Admin' claim'i eşliğinde istek gönderileceği ifade edilmiş olunmaktadır.

### 5. Gelişmiş Özellikler:

* **Yetkilendirme:** Ocelot, JWT (JSON Web Token) ve diğer yetkilendirme mekanizmaları ile entegre edilebilir.  `ocelot.json` dosyasında ilgili ayarlar yapılandırılmalıdır.

* **Güvenlik:**  HTTPS kullanımı, rate limiting, ve diğer güvenlik önlemleri Ocelot ile yapılandırılabilir.

* **QoS (Quality of Service):**  Ocelot, isteklerin önceliklendirilmesi, zaman aşımı ayarları gibi QoS özelliklerini destekler.

* **Loglama ve İzleme:**  Ocelot'un loglarını izleyerek uygulamanızın performansını ve hatalarını takip edebilirsiniz.

* **Health Checks:**  Mikro servislerin durumunu kontrol etmek için health check mekanizmaları entegre edilebilir.

* **Cache:**  Ocelot, yanıtları önbelleğe alarak performansı artırabilir.


### 6. Docker Kullanımı:

Docker kullanarak Ocelot ve mikro servislerinizi containerize edebilirsiniz.  Dockerfile oluşturarak, uygulamalarınızı ve bağımlılıklarını image'lere paketleyebilirsiniz.  Docker Compose ile tüm containerları yönetebilirsiniz.

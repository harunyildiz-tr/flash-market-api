FlashMarket; son kullanıcıya Getir benzeri raf odaklı bir alışveriş deneyimi sunarken, arka planda merkezi depo yerine Yemeksepeti iş modelini kullanarak yerel esnafın stoklarını yöneten çok satıcılı (multi-vendor) bir pazaryeri platformudur.
FlashMarket is a multi-vendor Q-commerce platform that combines Getir's shelf-based user experience with a Yemeksepeti-style marketplace model, connecting local merchants' existing inventories to customers instead of relying on central warehouses.

1. Sorun Ne? 

Klasik bir .NET projesinde (N-Layer Architecture) bağımlılık genelde şöyledir:
Controller -> Service -> Data Access (EF Core) -> SQL

Bu yapıda en büyük sorun şudur: Senin bütün iş mantığın (Business Logic), veritabanına göbekten bağlıdır.

Yarın "Entity Framework yerine Dapper kullanacağım" dersen, Service katmanını da değiştirmen gerekir.

"SQL Server yerine MongoDB kullanacağım" dersen, kodun yarısını çöpe atman gerekir.

Test yazmak zordur çünkü Service'i test ederken veritabanı bağlantısı ister.

2. Clean Architecture Nedir?

Robert C. Martin (Uncle Bob) tarafından popülerleştirilen bu mimarinin tek bir kuralı vardır: Bağımlılıklar (Dependencies) sadece İÇE DOĞRU olmalıdır.

Bunu bir soğan (Onion) gibi düşün. En içte en önemli şey, en dışta önemsiz detaylar vardır.

Katmanlar (İçten Dışa Doğru)

Domain (Çekirdek - En İç):

Burada sadece saf C# class'ları (Entities) bulunur. Product, User, Order gibi.

Kural: Burası dış dünyadan (Veritabanı, Web, API) habersizdir. Entity Framework referansı bile olmaz. Sadece kurallar vardır. (Örn: "Bir ürünün fiyatı eksi olamaz").

Application (Uygulama Mantığı):

Burada "Senaryolar" (Use Cases) vardır. "Ürün Ekle", "Sipariş Ver" gibi işlemler burada tanımlanır.

Kritik Nokta: Veritabanına nasıl erişileceğinin Interface'leri (Soyut hali) buradadır (IProductRepository). Ama kodun kendisi burada DEĞİLDİR.

Domain'i kullanır ama dış dünyayı bilmez.

Infrastructure (Altyapı):

Burası "Kirli İşlerin" döndüğü yerdir.

Application katmanındaki IProductRepository interface'ini burada implemente edersin (ProductRepository).

Entity Framework (DbContext), Email gönderme kütüphaneleri, Redis işlemleri burada yapılır.

Presentation (Sunum - En Dış):

Senin Web API veya MVC projen burasıdır.

Sadece Application katmanına emir verir ("Şu ürünü yarat" der). İşin nasıl yapıldığıyla ilgilenmez.

3. .NET Proje Yapısı (Solution Explorer)

Visual Studio'da boş bir "Blank Solution" açtığını hayal et. Klasör yapın şöyle olmalı:

1. MyProject.Domain (Class Library)

Bağımlılık: YOK (Sıfır).

İçerik: Product.cs, Order.cs (Entity'ler), Money (Value Object), DomainException.

2. MyProject.Application (Class Library)

Bağımlılık: Sadece MyProject.Domain.

İçerik:

IProductRepository.cs (Interface - Gövdesi boş!)

CreateProductService.cs (veya CQRS kullanıyorsan Handler'lar)

ProductDto.cs (API'ye dönecek veri modelleri)

Validation kuralları.

3. MyProject.Infrastructure (Class Library)

Bağımlılık: MyProject.Application (ve dolaylı olarak Domain).

İçerik:

NuGet paketleri burada yüklenir (EF Core, SqlClient vs.)

MyDbContext.cs

ProductRepository.cs (Burada IProductRepository implemente edilir ve veritabanına yazılır).

EmailSender.cs

4. MyProject.Api (Web API Projesi)

Bağımlılık: MyProject.Application ve MyProject.Infrastructure.

İçerik:

Controllers

Program.cs (Burada Dependency Injection ile Infrastructure'daki class'ları Application'daki interface'lere bağlarsın).

4. Sihir Nerede Gerçekleşiyor? (Dependency Inversion)

Eski yönteminde:
Controller -> Service -> Repository -> DB
Akış ve bağımlılık aynı yöndeydi.

Clean Architecture'da:
Controller -> Application <- Infrastructure

Dikkat et! Infrastructure (Veritabanı kodları), Application katmanına bağımlı. Çünkü Interface'ler Application katmanında.
Sen Application katmanında kod yazarken:
"Ben veritabanına kayıt atacağım ama veritabanı SQL mi, Oracle mı, Text dosyası mı bilmiyorum. Sadece elimde IRepository var, ben ona Add derim, gerisine karışmam" dersin.

Bu sayede Infrastructure projesini silip yerine bambaşka bir teknoloji koysan bile senin Domain ve Application kodların (Business Logic) asla bozulmaz.

5. Basit Bir Kod Örneği

1. Domain (Core):

code
C#
download
content_copy
expand_less
// Veritabanı ile ilgili hiçbir attribute yok!
public class Product {
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
}

2. Application:

code
C#
download
content_copy
expand_less
// Sadece sözleşme var, kod yok.
public interface IProductRepository {
    void Add(Product product);
}

public class ProductService {
    private readonly IProductRepository _repo;

    public ProductService(IProductRepository repo) {
        _repo = repo;
    }

    public void CreateNewProduct(string name, decimal price) {
        // İş kuralları burada
        if(price < 0) throw new Exception("Fiyat negatif olamaz");

        var p = new Product { Name = name, Price = price };
        _repo.Add(p); // Nereye kaydettiğini bilmiyor!
    }
}

3. Infrastructure:


// EF Core burada devreye giriyor
public class ProductRepository : IProductRepository {
    private readonly AppDbContext _context;
    
    public void Add(Product product) {
        _context.Products.Add(product);
        _context.SaveChanges();
    }
}

4. API (Program.cs):

// Bağlama işlemi (Dependency Injection)
builder.Services.AddScoped<IProductRepository, ProductRepository>();
6. Clean Architecture ile Genelde Kullanılan Kütüphaneler

Bu mimariye geçtiğinde .NET dünyasında şu terimleri ve kütüphaneleri çok duyacaksın:

CQRS (Command Query Responsibility Segregation): Okuma ve Yazma işlemlerini ayırma deseni. Clean Architecture ile çok iyi anlaşır.

MediatR: Controller ile Service arasındaki bağı koparan, mesajlaşma kütüphanesi. Controller bir istek atar, kimin karşılayacağını bilmez.

AutoMapper: Entity'leri DTO'lara çevirmek için.

FluentValidation: if (model.Name == null) gibi kodları Controller'dan çıkarıp Application katmanında temizce yazmak için.

7. Özet: Avantaj ve Dezavantaj

Avantajları:

Framework Bağımsız: .NET sürümü değişse de, Web API yerine Console uygulaması yapsan da Domain ve Application aynen kalır.

Test Edilebilirlik: Veritabanı olmadan tüm iş mantığını test edebilirsin.

Değişime Açık: Veritabanını değiştirmek çok kolaydır.
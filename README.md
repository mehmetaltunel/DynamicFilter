# DYNAMIC FILTERING - DETAYLI SUNUM DOKÜMANI

## 1. YAPILAN DEĞİŞİKLİKLER (GIT CHANGES)

### A. YENİ DOSYALAR (4 ADET)

#### 1.1 DynamicFilter/DynamicFilterParams.cs
**Amaç:** Filter parametrelerini taşıyan model ve operation enum'ları

**Model Yapısı:**
```csharp
public class DynamicFilterParams
{
    public string PropertyName { get; set; }  // Entity property adı
    public string Operation { get; set; }     // İşlem tipi
    public object Value { get; set; }         // Filtrelenecek değer
    public string CombineWith { get; set; }   // "and" veya "or"
}
```

**Enum'lar:**
```csharp
public enum DynamicFilterOperation
{
    Contains = 1,
    StartsWith,
    EndsWith,
    Equals,
    NotEquals,
    GreaterThan,
    GreaterThanOrEqual,
    LessThan,
    LessThanOrEqual,
    IsNull,
    IsNotNull
}

public enum DynamicCombineType
{
    And = 1,
    Or
}
```

**Önemli Metod:**
- `ParseFromArray()`: Fiplatform'un `[[["Code","contains","X"],"and",[...]]]` formatını parse eder

#### 1.2 DynamicFilter/DynamicQueryBuilder.cs
**Amaç:** String bazlı filtreleri LINQ Expression Tree'ye dönüştüren motor

**Ana Metod:**
```csharp
public static IQueryable<T> ApplyDynamicFilters<T>(
    IQueryable<T> source, 
    IList<DynamicFilterParams> filters)
```

**Yaptığı İşlemler:**
1. Reflection ile property bulur (case-insensitive)
2. Tip dönüşümü yapar: string -> int, Guid, DateTime, decimal, bool
3. Expression Tree oluşturur
4. AND/OR mantığı ile birleştirir
5. IQueryable'a WHERE şartı ekler

**Desteklenen Property Tipleri:**
- `string`, `int`, `long`, `decimal`, `double`, `float`
- `bool`
- `Guid`
- `DateTime`
- Nullable tüm veri tipleri
- Nested properties (örn: `Product.Category.Name`)

#### 1.3 DynamicFilter/DynamicFilterModelBinder.cs
**Amaç:** Query string'den gelen JSON parametrelerini parse eder

**Desteklediği Formatlar:**
- `?DynamicFilters={json}` - Tek object
- `?DynamicFilters={json}&DynamicFilters={json}` - Multiple objects
- `?DynamicFilters=[{...},{...}]` - JSON array

#### 1.4 DynamicFilter/DynamicFilterListConverter.cs
**Amaç:** JSON body'den gelen verileri parse eder

**Desteklediği Formatlar:**
- Tek JSON object: `{"propertyName": "...", ...}`
- JSON array: `[{...}, {...}]`
- Fiplatform array: `[[["Code","contains","X"],"and",[...]]]`

### B. DEĞİŞEN DOSYALAR

#### 2.1 PaginationFilterBase.cs

**EKLENEN PROPERTY:**
```csharp
[IgnoreFilter]
[JsonConverter(typeof(DynamicFilterListConverter))]
[ModelBinder(BinderType = typeof(DynamicFilterModelBinder))]
public virtual List<DynamicFilterParams> DynamicFilters { get; set; }
```

**DEĞİŞEN METODLAR:**

`ApplyFilterTo<TEntity>(IQueryable<TEntity> query)`:
```csharp
public override IQueryable<TEntity> ApplyFilterTo<TEntity>(IQueryable<TEntity> query)
{
    // Dinamik filtre varsa önce onu uygula
    if (DynamicFilters?.Any() == true)
        query = DynamicQueryBuilder.ApplyDynamicFilters(query, DynamicFilters);

    // Sonra mevcut statik filtreleri uygula
    return base.ApplyFilterTo(query).ToPaged(Page, PerPage);
}
```

`ApplyFilterTo<TEntity>(IOrderedQueryable<TEntity> query)`:
```csharp
public override IOrderedQueryable<TEntity> ApplyFilterTo<TEntity>(IOrderedQueryable<TEntity> query)
{
    if (DynamicFilters?.Any() == true)
        query = (IOrderedQueryable<TEntity>)DynamicQueryBuilder.ApplyDynamicFilters(query, DynamicFilters);

    return base.ApplyFilterTo(query).ToPaged(Page, PerPage);
}
```

`ApplyFilterWithoutPagination<T>(IQueryable<T> query)`:
```csharp
public override IQueryable<T> ApplyFilterWithoutPagination<T>(IQueryable<T> query)
{
    if (DynamicFilters?.Any() == true)
        query = DynamicQueryBuilder.ApplyDynamicFilters(query, DynamicFilters);

    return base.ApplyFilterWithoutPagination(query);
}
```

`ApplyFilterWithoutPaginationAndOrdering<T>(IQueryable<T> query)`:
```csharp
public override IQueryable<T> ApplyFilterWithoutPaginationAndOrdering<T>(IQueryable<T> query)
{
    if (DynamicFilters?.Any() == true)
        query = DynamicQueryBuilder.ApplyDynamicFilters(query, DynamicFilters);

    return base.ApplyFilterWithoutPaginationAndOrdering(query);
}
```

#### 2.2 Proto Files

**Server: Services.gRPC.Server.Internal.Api.Get/Protos/getstock.proto**
**Client: Services.gRPC.Clients.Internal.Api/protos/Get/getstock.proto**

```protobuf
message DynamicFilterParamsMessage {
    string propertyName = 1;
    string operation = 2;
    string value = 3;
    string combineWith = 4;
}

message GetStockInfoRequestMessage {
    string customerAccountCode = 1;
    google.protobuf.BoolValue isShowZeroStock = 2;
    google.protobuf.Timestamp startDate = 3;
    google.protobuf.Timestamp endDate = 4;
    string sort = 5;
    int32 sortBy = 6;
    int32 page = 7;
    int32 perPage = 8;
    bool isActive = 9;
    bool isDeleted = 10;
    repeated DynamicFilterParamsMessage dynamicFilters = 11;  // YENİ
}
```

#### 2.3 AutoMapper Profiles

**Server: StockProfile.cs**
```csharp
CreateMap<GetStockInfoRequestMessage, GetStockInfoRequestModel>()
    .ForMember(dest => dest.DynamicFilters, opt => opt.MapFrom(src => 
        src.DynamicFilters.Select(f => new DynamicFilterParams
        {
            PropertyName = f.PropertyName,
            Operation = f.Operation,
            Value = f.Value,
            CombineWith = f.CombineWith
        })));
```

**Client: StockProfile.cs**
```csharp
CreateMap<GetStockInfoRequestModel, GetStockInfoRequestMessage>()
    .ForMember(dest => dest.DynamicFilters, opt => opt.MapFrom(src => 
        src.DynamicFilters != null ? src.DynamicFilters.Select(f => 
            new DynamicFilterParamsMessage
            {
                PropertyName = f.PropertyName != null ? f.PropertyName : "",
                Operation = f.Operation != null ? f.Operation : "",
                Value = f.Value != null ? f.Value.ToString() : "",
                CombineWith = f.CombineWith != null ? f.CombineWith : ""
            }) : null));
```

## 2. DESTEKLENEN OPERASYONLAR (11 ADET)

| Operasyon | Ana Kullanım | Tüm Desteklenen Alias'lar | SQL Karşılığı |
|-----------|-------------|---------------------------|---------------|
| Contains | Metin içinde arama | `contains` | `LIKE '%value%'` |
| StartsWith | Başlangıç eşleşmesi | `startswith`, `startwith` | `LIKE 'value%'` |
| EndsWith | Bitiş eşleşmesi | `endswith`, `endwith` | `LIKE '%value'` |
| Equals | Tam eşitlik | `eq`, `=`, `equals` | `= value` |
| NotEquals | Eşit değil | `neq`, `!=`, `<>`, `notequals` | `<> value` |
| GreaterThan | Büyüktür | `gt`, `>`, `greaterthan` | `> value` |
| GreaterThanOrEqual | Büyük eşit | `gte`, `>=`, `greaterthanorequal` | `>= value` |
| LessThan | Küçüktür | `lt`, `<`, `lessthan` | `< value` |
| LessThanOrEqual | Küçük eşit | `lte`, `<=`, `lessthanorequal` | `<= value` |
| IsNull | Null kontrolü | `isnull` | `IS NULL` |
| IsNotNull | Not null kontrolü | `isnotnull` | `IS NOT NULL` |

**ÖNEMLİ:** Tüm operasyonlar case-insensitive'dir (büyük/küçük harf fark etmez)

## 3. DESTEKLENEN VERİ TİPLERİ

- `string`
- `int` (Int32)
- `long` (Int64)
- `decimal`
- `double`
- `float`
- `bool`
- `Guid`
- `DateTime`
- Tüm nullable versiyonları (örn: `int?`, `decimal?`)
- Nested properties (örn: `Order.Customer.Email`)

## 4. DESTEKLENEN GİRİŞ FORMATLARI

### 4.1 HTTP GET (Query String)
```
?DynamicFilters={"propertyName":"ProductCode","operation":"startswith","value":"YIL"}
```

### 4.2 Multiple Filters (GET)
```
?DynamicFilters={"propertyName":"ProductCode","operation":"contains","value":"test"}
&DynamicFilters={"propertyName":"Quantity","operation":"gt","value":"10"}
```

### 4.3 HTTP POST / gRPC (JSON Body)
```json
{
  "customerAccountCode": "Ficommerce",
  "dynamicFilters": [
    {
      "propertyName": "ProductCode",
      "operation": "contains",
      "value": "test",
      "combineWith": "and"
    },
    {
      "propertyName": "PhysicalStock",
      "operation": "gt",
      "value": "0",
      "combineWith": "and"
    }
  ]
}
```

### 4.4 Fiplatform Array Format
```json
[[["ProductCode","contains","test"],"and",["Quantity","gt","5"]]]
```

## 5. KULLANIM SENARYOLARı

### Örnek 1: Basit Filtreleme
**Request:**
```
GET /api/Stock/GetStockInfo?CustomerAccountCode=Ficommerce
&DynamicFilters={"propertyName":"ProductCode","operation":"startwith","value":"YIL"}
```

**Oluşan SQL:**
```sql
WHERE ProductCode LIKE 'YIL%'
```

### Örnek 2: AND Kombinasyonu
**Request:**
```json
{
  "dynamicFilters": [
    {"propertyName": "ProductCode", "operation": "contains", "value": "test", "combineWith": "and"},
    {"propertyName": "PhysicalStock", "operation": "gt", "value": "0", "combineWith": "and"}
  ]
}
```

**Oluşan SQL:**
```sql
WHERE ProductCode LIKE '%test%' AND PhysicalStock > 0
```

### Örnek 3: OR Kombinasyonu
**Request:**
```json
{
  "dynamicFilters": [
    {"propertyName": "PhysicalStock", "operation": "equals", "value": "1", "combineWith": "or"},
    {"propertyName": "ProductCode", "operation": "contains", "value": "fi", "combineWith": "or"}
  ]
}
```

**Oluşan SQL:**
```sql
WHERE (PhysicalStock = 1) OR (ProductCode LIKE '%fi%')
```

## 6. TEKNİK AVANTAJLAR

1. **Zero Breaking Change:** Mevcut handler'lar hiç değişmedi
2. **Database Level:** IQueryable üzerinden çalışır, SQL Server'da execute olur
3. **Generic:** Tüm entity'lerde çalışır
4. **Type Safe:** Otomatik tip dönüşümü yapar
5. **Multi-Protocol:** HTTP GET, POST ve gRPC destekler
6. **Flexible Input:** JSON, Query String, Fiplatform array formatlarını destekler

## 7. TEKNIK DETAY: ApplyDynamicFilters İÇİNDEKİ İŞLEM ADIMLARI

1. **Parameter Creation:** `x => ...` yapısındaki parametre oluşturulur
2. **Property Discovery:** Reflection ile property bulunur (case-insensitive)
3. **Type Conversion:** String değer hedef tipe cast edilir
4. **Expression Building:**
   - Contains: `Expression.Call(property, "Contains", value)`
   - GreaterThan: `Expression.GreaterThan(property, value)`
   - Equals: `Expression.Equal(property, value)`
5. **Logic Combining:** `Expression.AndAlso()` veya `Expression.OrElse()` ile birleştirme
6. **Lambda Creation:** `Expression.Lambda<Func<T, bool>>(...)` oluşturulur
7. **Query Injection:** `source.Where(lambda)` ile uygulanır

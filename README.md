# EklKibanaExampleWebApi      ([ForEnglish](https://github.com/coderentr/EklKibanaExampleWebApi/blob/main/README_EN.md))

Bu projede Serilog ile Elasticsearch'e log atmak ve kibana ile gösterimini yapmak üzerine bir geliştirme yaptım. 

#### Geliştirme adımları; 

Projenin oluşturulması;

```cmd
dotnet new webapi –name EklKibanaExampleWebApi
```
İlgili Paketlerin Projeye eklenmesi;

```cmd

dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Exceptions
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.Debug
dotnet add package Serilog.Sinks.Elasticsearch

```
Paketleri projeye ekledikten sonra program.cs de gerekli configurasyonlerı yapıyoruz.

```c#

void configureLogging(){
    var environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT");

    var configuration = new ConfigurationBuilder()
       .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
       .AddJsonFile(
        $"appsettings.{environment}.json", optional: true)
       .Build();

    Log.Logger = new LoggerConfiguration()
        .Enrich.FromLogContext()
        .Enrich.WithExceptionDetails()
        .WriteTo.Debug()
        .WriteTo.Console()
        .WriteTo.Elasticsearch(ConfigureElasticSink(configuration, environment ?? string.Empty))
        .Enrich.WithProperty("Environment", environment??string.Empty)
        .ReadFrom.Configuration(configuration)
        .CreateLogger();
}

ElasticsearchSinkOptions ConfigureElasticSink(IConfigurationRoot configuration, string environment)
{
    return new ElasticsearchSinkOptions(new Uri(configuration["ElasticConfiguration:Uri"]))
    {
        AutoRegisterTemplate =true,
        IndexFormat=$"{Assembly.GetExecutingAssembly().GetName()?.Name?.ToString().ToLower().Replace(".","-")}-{environment.ToLower()}-{DateTime.UtcNow:yyyy-MM}",
        NumberOfReplicas = 1,
        NumberOfShards = 2
    };

}

```

Bu konfigurasyon ile indeximiz otomatik olarak oluşacaktır. Kibana üzerinden gerekli düzenlemeleri aşağıda görsel olarak ekledim.


appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Information",
        "System": "Warning"
      }
    }
  },
  "ElasticConfiguration": {
    "Uri": "http://elasticsearch:9200"
  },
  "AllowedHosts": "*"
}
```

Ardından build'den önce

```c#
configureLogging();

builder.Host.UseSerilog();

```

ekliyoruz.


Projeye sağ tıklayıp Add> Add Docker Support deyip projemizin Dockerfile'ının oluşturuyoruz.

Sonra Elasticsearc, kibana ve projemizi docker'da ayağa kaldıracak şekilde docker-compose.yml dosyamızı ekleyip aşağıdaki gibi düzenliyoruz.
yml dosyası projenizin gereksinimleri doğrultusunda düzenlenmeli. 

```docker-compose.yml

version: '3.1'

services:
  elasticsearch:
    container_name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:7.13.4
    environment:
      - "discovery.type=single-node"
      - "bootstrap.memory_lock=true"
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    ports:
      - "9200:9200"

  kibana:
    container_name: kibana
    image: docker.elastic.co/kibana/kibana:7.13.4
    environment:
      - "ELASTICSEARCH_URL=http://elasticsearch:9200"
      - "ELASTICSEARCH_HOSTS=http://elasticsearch:9200"
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  EklKibanaExampleWebApi:
    container_name: ElkKibanaExampleWebApi
    build:
      context: .
      dockerfile: Dockerfile   

    image: eklkibanaexamplewebapi   
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://0.0.0.0:80
    ports:
      - "6060:80"
      - "6061:443"
    
    

```

```
docker-compose up
```

komutu ile servislerimizi docker'da ayağa kaldırıyoruz.

<img width="777" alt="Screenshot 2023-08-22 at 23 43 10" src="https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/5d37b4f4-b7ae-41f2-bc2b-e9a50f4e6f02">


ElasticSearch:

<img width="475" alt="Screenshot 2023-08-22 at 23 44 23" src="https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/bf5edb29-2d69-4b03-a7b1-c471bf17af52">

Kibana:

<img width="829" alt="Screenshot 2023-08-22 at 23 45 25" src="https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/8639d937-b9c7-4669-be36-40cf8982202d">

WebApi:

<img width="859" alt="Screenshot 2023-08-22 at 23 47 05" src="https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/859e92c4-8003-4632-8921-a24d9758f7ab">

Kibana'da index'imizi ekleyelim; 

<img width="701" alt="Screenshot 2023-08-22 at 23 48 16" src="https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/4d82a07d-b6d5-46ad-af58-e345531f7004">


<img width="830" alt="Screenshot 2023-08-22 at 23 49 11" src="https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/e3276495-5dc1-4da5-b5c0-6116c851296a">


<img width="1000" alt="Screenshot 2023-08-22 at 23 49 49" src="https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/147018f7-c697-446a-a3c8-4ef15e88d146">

<img width="994" alt="Screenshot 2023-08-22 at 23 50 27" src="https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/7dc5aceb-fd42-47aa-b404-59a53d8b248a">

İndex'imizi oluşturduktan sonra ilgili loglarımızı görüntüleyebiliyoruz. 

<img width="1017" alt="Screenshot 2023-08-22 at 23 51 02" src="https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/87b452b6-6c3c-4f17-949f-aff9be6d9895">



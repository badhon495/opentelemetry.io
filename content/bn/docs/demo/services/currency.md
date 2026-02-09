---
title: কারেন্সি সার্ভিস
linkTitle: কারেন্সি
aliases: [currencyservice]
cSpell:ignore: decltype labelkv noexcept nostd
default_lang_commit: ae417344d183999236c22834435e0dfeb109da29
---

এই সার্ভিসটি বিভিন্ন কারেন্সির মধ্যে পরিমাণ রূপান্তর করার কার্যকারিতা প্রদান করে।

[কারেন্সি সার্ভিস সোর্স](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/currency/)

## ট্রেস

### ট্রেসিং ইনিশিয়ালাইজ করা

OpenTelemetry SDK, `main` থেকে `tracer_common.h` এ ডিফাইন করা `initTracer` ফাংশন ব্যবহার করে ইনিশিয়ালাইজ করা হয়।

```cpp
void initTracer()
{
  auto exporter = opentelemetry::exporter::otlp::OtlpGrpcExporterFactory::Create();
  auto processor =
      opentelemetry::sdk::trace::SimpleSpanProcessorFactory::Create(std::move(exporter));
  std::vector<std::unique_ptr<opentelemetry::sdk::trace::SpanProcessor>> processors;
  processors.push_back(std::move(processor));
  std::shared_ptr<opentelemetry::sdk::trace::TracerContext> context =
      opentelemetry::sdk::trace::TracerContextFactory::Create(std::move(processors));
  std::shared_ptr<opentelemetry::trace::TracerProvider> provider =
      opentelemetry::sdk::trace::TracerProviderFactory::Create(context);
 // Set the global trace provider
  opentelemetry::trace::Provider::SetTracerProvider(provider);

 // set global propagator
  opentelemetry::context::propagation::GlobalTextMapPropagator::SetGlobalPropagator(
      opentelemetry::nostd::shared_ptr<opentelemetry::context::propagation::TextMapPropagator>(
          new opentelemetry::trace::propagation::HttpTraceContext()));
}
```

### নতুন স্প্যান তৈরি করা

নতুন span তৈরি এবং শুরু করা যায় `Tracer->StartSpan("spanName", attributes, options)` ব্যবহার করে। একটি span তৈরি করার পরে আপনাকে `Tracer->WithActiveSpan(span)` ব্যবহার করে এটি শুরু করতে এবং সক্রিয় কনটেক্সটে রাখতে হবে। আপনি `Convert` ফাংশনে এর একটি উদাহরণ খুঁজে পাবেন।

```cpp
std::string span_name = "CurrencyService/Convert";
auto span =
    get_tracer("currency")->StartSpan(span_name,
                                  {{SemanticConventions::kRpcSystem, "grpc"},
                                   {SemanticConventions::kRpcService, "oteldemo.CurrencyService"},
                                   {SemanticConventions::kRpcMethod, "Convert"},
                                   {SemanticConventions::kRpcGrpcStatusCode, 0}},
                                  options);
auto scope = get_tracer("currency")->WithActiveSpan(span);
```

### স্প্যানে অ্যাট্রিবিউট যোগ করা

আপনি `Span->SetAttribute(key, value)` ব্যবহার করে একটি span এ attribute যোগ করতে পারেন।

```cpp
span->SetAttribute("app.currency.conversion.from", from_code);
span->SetAttribute("app.currency.conversion.to", to_code);
```

### স্প্যান ইভেন্ট যোগ করা

span event যোগ করা `Span->AddEvent(name)` ব্যবহার করে সম্পন্ন করা হয়।

```cpp
span->AddEvent("Conversion successful, response sent back");
```

### স্প্যান স্ট্যাটাস সেট করা

আপনার span status `Ok` বা `Error` এ যথাযথভাবে সেট করতে ভুলবেন না। আপনি `Span->SetStatus(status)` ব্যবহার করে এটি করতে পারেন।

```cpp
span->SetStatus(StatusCode::kOk);
```

### ট্রেসিং কনটেক্সট প্রোপাগেশন

C++ তে propagation স্বয়ংক্রিয়ভাবে হ্যান্ডল করা হয় না। আপনাকে caller থেকে এটি এক্সট্র্যাক্ট করতে হবে এবং পরবর্তী span গুলোতে propagation context ইনজেক্ট করতে হবে। `GrpcServerCarrier` ক্লাস inbound gRPC রিকোয়েস্ট থেকে context এক্সট্র্যাক্ট করার জন্য একটি মেথড ডিফাইন করে যা সার্ভিস কল ইমপ্লিমেন্টেশনে ব্যবহার করা হয়।

`GrpcServerCarrier` ক্লাসটি `tracer_common.h` এ নিম্নরূপে ডিফাইন করা হয়েছে:

```cpp
class GrpcServerCarrier : public opentelemetry::context::propagation::TextMapCarrier
{
public:
  GrpcServerCarrier(ServerContext *context) : context_(context) {}
  GrpcServerCarrier() = default;
  virtual opentelemetry::nostd::string_view Get(
      opentelemetry::nostd::string_view key) const noexcept override
  {
    auto it = context_->client_metadata().find(key.data());
    if (it != context_->client_metadata().end())
    {
      return it->second.data();
    }
    return "";
  }

  virtual void Set(opentelemetry::nostd::string_view key,
                   opentelemetry::nostd::string_view value) noexcept override
  {
   // Not required for server
  }

  ServerContext *context_;
};
```

এই ক্লাসটি `Convert` মেথডে context এক্সট্র্যাক্ট করতে এবং সঠিক context ধারণ করার জন্য একটি `StartSpanOptions` অবজেক্ট তৈরি করতে ব্যবহৃত হয় যা নতুন span তৈরি করার সময় ব্যবহৃত হয়।

```cpp
StartSpanOptions options;
options.kind = SpanKind::kServer;
GrpcServerCarrier carrier(context);

auto prop        = context::propagation::GlobalTextMapPropagator::GetGlobalPropagator();
auto current_ctx = context::RuntimeContext::GetCurrent();
auto new_context = prop->Extract(carrier, current_ctx);
options.parent   = GetSpan(new_context)->GetContext();
```

## মেট্রিক্স

### মেট্রিক্স ইনিশিয়ালাইজ করা

OpenTelemetry `MeterProvider`, `main()` থেকে `meter_common.h` এ ডিফাইন করা `initMeter()` ফাংশন ব্যবহার করে ইনিশিয়ালাইজ করা হয়।

```cpp
void initMeter()
{
  // Build MetricExporter
  otlp_exporter::OtlpGrpcMetricExporterOptions otlpOptions;
  auto exporter = otlp_exporter::OtlpGrpcMetricExporterFactory::Create(otlpOptions);

  // Build MeterProvider and Reader
  metric_sdk::PeriodicExportingMetricReaderOptions options;
  std::unique_ptr<metric_sdk::MetricReader> reader{
      new metric_sdk::PeriodicExportingMetricReader(std::move(exporter), options) };
  auto provider = std::shared_ptr<metrics_api::MeterProvider>(new metric_sdk::MeterProvider());
  auto p = std::static_pointer_cast<metric_sdk::MeterProvider>(provider);
  p->AddMetricReader(std::move(reader));
  metrics_api::Provider::SetMeterProvider(provider);
}
```

### IntCounter শুরু করা

একটি গ্লোবাল `currency_counter` ভেরিয়েবল `main()` এ তৈরি করা হয় `meter_common.h` এ ডিফাইন করা `initIntCounter()` ফাংশন কল করে।

```cpp
nostd::unique_ptr<metrics_api::Counter<uint64_t>> initIntCounter()
{
  std::string counter_name = name + "_counter";
  auto provider = metrics_api::Provider::GetMeterProvider();
  nostd::shared_ptr<metrics_api::Meter> meter = provider->GetMeter(name, version);
  auto int_counter = meter->CreateUInt64Counter(counter_name);
  return int_counter;
}
```

### কারেন্সি রূপান্তর রিকোয়েস্ট গণনা করা

`CurrencyCounter()` মেথডটি নিম্নরূপে ইমপ্লিমেন্ট করা হয়েছে:

```cpp
void CurrencyCounter(const std::string& currency_code)
{
    std::map<std::string, std::string> labels = { {"currency_code", currency_code} };
    auto labelkv = common::KeyValueIterableView<decltype(labels)>{ labels };
    currency_counter->Add(1, labelkv);
}
```

প্রতিবার `Convert()` ফাংশন কল করা হলে, `to_code` হিসেবে প্রাপ্ত কারেন্সি কোডটি conversion গণনা করতে ব্যবহার করা হয়।

```cpp
CurrencyCounter(to_code);
```

## লগস

OpenTelemetry `LoggerProvider`, `main()` থেকে `logger_common.h` এ ডিফাইন করা `initLogger()` ফাংশন ব্যবহার করে ইনিশিয়ালাইজ করা হয়।

```cpp
void initLogger() {
  otlp::OtlpGrpcLogRecordExporterOptions loggerOptions;
  auto exporter  = otlp::OtlpGrpcLogRecordExporterFactory::Create(loggerOptions);
  auto processor = logs_sdk::SimpleLogRecordProcessorFactory::Create(std::move(exporter));
  std::vector<std::unique_ptr<logs_sdk::LogRecordProcessor>> processors;
  processors.push_back(std::move(processor));
  auto context = logs_sdk::LoggerContextFactory::Create(std::move(processors));
  std::shared_ptr<logs::LoggerProvider> provider = logs_sdk::LoggerProviderFactory::Create(std::move(context));
  opentelemetry::logs::Provider::SetLoggerProvider(provider);
}
```

### LoggerProvider ব্যবহার করা

ইনিশিয়ালাইজ করা Logger Provider, `server.cpp` এর `main` থেকে কল করা হয়:

```cpp
logger = getLogger(name);
```

এটি `logger` নামক একটি লোকাল ভেরিয়েবলে logger অ্যাসাইন করে:

```cpp
nostd::shared_ptr<opentelemetry::logs::Logger> logger;
```

যা তারপর কোড জুড়ে যখনই আমাদের একটি লাইন লগ করতে হয় তখন ব্যবহার করা হয়:

```cpp
logger->Info(std::string(__func__) + " conversion successful");
```

---
title: কারেন্সি সার্ভিস
linkTitle: কারেন্সি
aliases: [currencyservice]
cSpell:ignore: decltype labelkv noexcept nostd
default_lang_commit: ae417344d183999236c22834435e0dfeb109da29
---

এই সার্ভিসটি বিভিন্ন মুদ্রার মধ্যে অর্থের পরিমাণ রূপান্তর করার সুবিধা প্রদান করে।

[কারেন্সি সার্ভিস সোর্স](https://github.com/open-telemetry/opentelemetry-demo/blob/main/src/currency/)

## ট্রেস {#traces}

### ট্রেসিং ইনিশিয়ালাইজ করা {#initializing-tracing}

OpenTelemetry SDK-টি `main` থেকে `tracer_common.h`-এ সংজ্ঞায়িত `initTracer` ফাংশন ব্যবহার করে ইনিশিয়ালাইজ করা হয়।

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
 // গ্লোবাল ট্রেস প্রোভাইডার সেট করুন
  opentelemetry::trace::Provider::SetTracerProvider(provider);

 // গ্লোবাল প্রপাগেটর সেট করুন
  opentelemetry::context::propagation::GlobalTextMapPropagator::SetGlobalPropagator(
      opentelemetry::nostd::shared_ptr<opentelemetry::context::propagation::TextMapPropagator>(
          new opentelemetry::trace::propagation::HttpTraceContext()));
}
```

### নতুন স্প্যান তৈরি করা {#create-new-spans}

`Tracer->StartSpan("spanName", attributes, options)` ব্যবহার করে নতুন স্প্যান তৈরি এবং শুরু করা যায়। একটি স্প্যান তৈরির পরে `Tracer->WithActiveSpan(span)` ব্যবহার করে এটি শুরু করে সক্রিয় কনটেক্সটে রাখতে হবে। `Convert` ফাংশনে এর একটি উদাহরণ পাওয়া যাবে।

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

### স্প্যানে অ্যাট্রিবিউট যোগ করা {#adding-attributes-to-spans}

`Span->SetAttribute(key, value)` ব্যবহার করে একটি স্প্যানে অ্যাট্রিবিউট যোগ করা যায়।

```cpp
span->SetAttribute("app.currency.conversion.from", from_code);
span->SetAttribute("app.currency.conversion.to", to_code);
```

### স্প্যান ইভেন্ট যোগ করা {#add-span-events}

`Span->AddEvent(name)` ব্যবহার করে স্প্যান ইভেন্ট যোগ করা হয়।

```cpp
span->AddEvent("Conversion successful, response sent back");
```

### স্প্যান স্ট্যাটাস সেট করা {#set-span-status}

আপনার স্প্যান স্ট্যাটাস যথাযথভাবে `Ok` বা `Error` সেট করতে ভুলবেন না। এটি `Span->SetStatus(status)` ব্যবহার করে করা যায়।

```cpp
span->SetStatus(StatusCode::kOk);
```

### ট্রেসিং কনটেক্সট প্রপাগেশন {#tracing-context-propagation}

C++-এ প্রপাগেশন স্বয়ংক্রিয়ভাবে পরিচালিত হয় না। আপনাকে কলার থেকে এটি এক্সট্র্যাক্ট করতে হবে এবং পরবর্তী স্প্যানগুলোতে প্রপাগেশন কনটেক্সট ইনজেক্ট করতে হবে। `GrpcServerCarrier` ক্লাস ইনবাউন্ড gRPC রিকোয়েস্ট থেকে কনটেক্সট এক্সট্র্যাক্ট করার একটি পদ্ধতি সংজ্ঞায়িত করে যা সার্ভিস কল ইম্পলিমেন্টেশনে ব্যবহৃত হয়।

`GrpcServerCarrier` ক্লাসটি `tracer_common.h`-এ নিম্নলিখিতভাবে সংজ্ঞায়িত করা হয়েছে:

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
   // সার্ভারের জন্য প্রয়োজন নেই
  }

  ServerContext *context_;
};
```

এই ক্লাসটি `Convert` মেথডে কনটেক্সট এক্সট্র্যাক্ট করতে এবং নতুন স্প্যান তৈরির সময় ব্যবহারের জন্য সঠিক কনটেক্সট ধারণকারী একটি `StartSpanOptions` অবজেক্ট তৈরি করতে ব্যবহৃত হয়।

```cpp
StartSpanOptions options;
options.kind = SpanKind::kServer;
GrpcServerCarrier carrier(context);

auto prop        = context::propagation::GlobalTextMapPropagator::GetGlobalPropagator();
auto current_ctx = context::RuntimeContext::GetCurrent();
auto new_context = prop->Extract(carrier, current_ctx);
options.parent   = GetSpan(new_context)->GetContext();
```

## মেট্রিক্স {#metrics}

### মেট্রিক্স ইনিশিয়ালাইজ করা {#initializing-metrics}

`meter_common.h`-এ সংজ্ঞায়িত `initMeter()` ফাংশন ব্যবহার করে `main()` থেকে OpenTelemetry `MeterProvider` ইনিশিয়ালাইজ করা হয়।

```cpp
void initMeter()
{
  // MetricExporter তৈরি করুন
  otlp_exporter::OtlpGrpcMetricExporterOptions otlpOptions;
  auto exporter = otlp_exporter::OtlpGrpcMetricExporterFactory::Create(otlpOptions);

  // MeterProvider এবং Reader তৈরি করুন
  metric_sdk::PeriodicExportingMetricReaderOptions options;
  std::unique_ptr<metric_sdk::MetricReader> reader{
      new metric_sdk::PeriodicExportingMetricReader(std::move(exporter), options) };
  auto provider = std::shared_ptr<metrics_api::MeterProvider>(new metric_sdk::MeterProvider());
  auto p = std::static_pointer_cast<metric_sdk::MeterProvider>(provider);
  p->AddMetricReader(std::move(reader));
  metrics_api::Provider::SetMeterProvider(provider);
}
```

### IntCounter শুরু করা {#starting-intcounter}

`meter_common.h`-এ সংজ্ঞায়িত `initIntCounter()` ফাংশন কল করে `main()`-এ একটি গ্লোবাল `currency_counter` ভেরিয়েবল তৈরি করা হয়।

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

### কারেন্সি রূপান্তর রিকোয়েস্ট গণনা করা {#counting-currency-conversion-requests}

`CurrencyCounter()` মেথডটি নিম্নলিখিতভাবে ইম্পলিমেন্ট করা হয়েছে:

```cpp
void CurrencyCounter(const std::string& currency_code)
{
    std::map<std::string, std::string> labels = { {"currency_code", currency_code} };
    auto labelkv = common::KeyValueIterableView<decltype(labels)>{ labels };
    currency_counter->Add(1, labelkv);
}
```

প্রতিবার `Convert()` ফাংশন কল হলে, `to_code` হিসেবে প্রাপ্ত মুদ্রা কোড রূপান্তর গণনা করতে ব্যবহৃত হয়।

```cpp
CurrencyCounter(to_code);
```

## লগস {#logs}

`logger_common.h`-এ সংজ্ঞায়িত `initLogger()` ফাংশন ব্যবহার করে `main()` থেকে OpenTelemetry `LoggerProvider` ইনিশিয়ালাইজ করা হয়।

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

### LoggerProvider ব্যবহার করা {#using-the-loggerprovider}

ইনিশিয়ালাইজ করা Logger Provider `server.cpp`-এ `main` থেকে কল করা হয়:

```cpp
logger = getLogger(name);
```

এটি `logger` নামক একটি লোকাল ভেরিয়েবলে লগারটি অ্যাসাইন করে:

```cpp
nostd::shared_ptr<opentelemetry::logs::Logger> logger;
```

যা পরে কোডে যেখানেই একটি লাইন লগ করার প্রয়োজন হয় সেখানে ব্যবহার করা হয়:

```cpp
logger->Info(std::string(__func__) + " conversion successful");
```
